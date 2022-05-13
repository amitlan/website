---
layout: writing
title: "Postgres: woes of prepared statements with partitioning"
tags: [writing, pg]
last_updated: 2022-05-11
---
# Postgres: woes of prepared statements with partitioning

May 11, 2022

Prepared statements (aka parameterized queries) suffer when you have partitioned
tables mentioned in them.  Let's see why.

Using prepared statements, a client can issue a query in two stages.  First, *prepare*
it using `PREPARE a_name AS <query>` when the connected Postgres backend process will
parse-analyze the query and remember the parse tree in a hash table using the provided
name as its lookup key, followed by multiple *executions* using the statement `EXECUTE
a_name`, each of which will compute and return the query's output rows.

The query specified in `PREPARE` can put numbered parameters ($1, $2, ...) in place of
the actual constant values that may be present in the query's conditions.  The values
themselves are only supplied during a given `EXECUTE` invocation.  During each execution,
the backend process will look up the parse tree in the hash table and make a plan based
on it to compute the result of the query for given set of values of the paramters.
 
The benefit of this 2-stage processing is that a lot of CPU cycles are saved by not
redoing the parse-analysis processing on every execution, which is fine because the result
of that processing would be the exact same parse tree unless some object mentioned in the
query was changed by DDL, something that tends to happen rather unfrequently.

Even more CPU cycles are saved if the `EXECUTE` step is able to use a plan that is also
cached, instead of building it from scratch for that particular execution.

It is easy to check the performance benefit of this 2-stage protocol of performing queries
using the handy `pgbench` tool, which allows specifying which protocol to use when running
the benchmark queries using the parameter `--protocol=querymode`. The value *simple*
instructs it to execute the queries in one go and *prepared* to use the 2-stage method.

```
$ pgbench -i > /dev/null 2>&1
$ pgbench -S -t 100000 --protocol=simple
pgbench (15devel)
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
number of transactions per client: 100000
number of transactions actually processed: 100000/100000
number of failed transactions: 0 (0.000%)
latency average = 0.058 ms
initial connection time = 9.907 ms
tps = 17201.844726 (without initial connection time)
```

and:

```
$ pgbench -i > /dev/null 2>&1
$ pgbench -S -t 100000 --protocol=prepared
...
latency average = 0.031 ms
initial connection time = 9.686 ms
tps = 32234.211200 (without initial connection time)
```

So the latency average for query `SELECT abalance FROM pgbench_accounts WHERE aid = ?`
is 0.031 milliseconds when performed using the parameterized query protocol versus 0.058
when using the simple protocol.

It's interesting to consider how the plan itself may be cached, because that is where the
problems with partitioned tables that I want to talk about can be traced to.  The backend
code relies on the plancache module which hosts the plan caching logic.  When presented
with the parse tree of a query and the set of parameter values provided in `EXECUTE`, its
job is to figure out if a plan matching the query already exists and if one does, check if
it's still valid by acquiring read locks all the relations that it scans.  If any of the
relations has changed since the plan was created, this validation steps detects those
changes and triggers replanning.

An important bit about such a cached plan is that plancache does not pass the parameter
values that the `EXECUTE` would have provided to the planner, so it is a *generic*
(parameter-value-indepedent) plan.  The plan is in most cases same as the one plancache
would get by passing the parameter values, because the planner is smart enough, for
example, to determine that an index on `colname` can be used to optimize both
`colname = <constant-value>` and `colname = $1`.

However, if the query contains a partitioned table, the planner's lack of access to
parameter values means that the planner is unable to use partition pruning, because
pruning relies on using the values mentioned in prunable conditions to select the
matching partitions.

The decision of whether or not to use a cached plan is made by the plancache module
present in the backend that is involved in the processing of the `EXECUTE` statement.
The way it does that is by keeping track of and comparing the costs of two types of
plan: 1) that obtained by binding the values of parameters provided in the `EXECUTE`
statement (that is by making them known to the planner), 2) that obtained by leaving
the query parameters unbounded (that is, by leaving the planner in the dark about what
their values are).  The former is called a "custom" plan, because the planner would have
customized it to the given set of parameter values and the latter a "generic" plan,
because it is not specific to any given set of parameter values.  By definition, only
a generic plan is deemed reusable across multiple `EXECUTE`s of a given preapred query
and thus cached.  It may seem as if a custom plan would always be cheaper than a generic
plan, though given that the effort to create the plan itself may be significant when
considered in terms of plan cost unit, generic plan often comes out ahead.
