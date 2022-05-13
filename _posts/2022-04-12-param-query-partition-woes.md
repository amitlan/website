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
it using `PREPARE a_name AS <query>` when the connected Postgres backend process will only
parse-analyze the query and remember the parse tree in a (process-local) hash table using
the provided name as its lookup key, followed by *execution* of the query using the
statement `EXECUTE a_name`, each of which will compute and return the query's output rows.

The query specified in `PREPARE` can put numbered parameters ($1, $2, ...) in place of
the actual constant values in the query's conditions (like `WHERE`).  The values themselves
are only supplied during a given `EXECUTE` invocation.  During each execution, the backend
process will look up the parse tree in the hash table and make a plan based on it to compute
the result of the query for given set of parameter values.
 
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
is 0.031 milliseconds when performed using the 2-stage protocol versus 0.058 when
using the simple protocol.

It's interesting to consider how the plan itself may be cached, because that is where the
problems with partitioned tables that I want to talk about can be traced to.  The backend
code relies on the plancache module which hosts the plan caching logic.  When presented
with the parse tree of a query, its job is to figure out if a plan matching the query
already exists and if one does, check if it's still valid by acquiring read locks all the
relations that it scans.  If any of the relations has changed since the plan was created,
this validation step detects those changes and triggers replanning.

An important bit about such a cached plan is that plancache does not pass the parameter
values that the `EXECUTE` would have provided to the planner, so it is a *generic*
(parameter-value-indepedent) plan.  The plan is in most cases same as the one plancache
would get by passing the parameter values, because the planner is smart enough, for
example, to determine that an index on `colname` can be used to optimize both
`colname = <constant-value>` and `colname = $1`.

Parameter values not being available to the planner when creating a generic plan can be a
big problem if the query contains partitioned tables.  Partition pruning can be very helpful
to create an optimal plan containing only the partitions whose bounds that match the values
mentioned in the `WHERE` condition.  Without parameter values pruning cannot occur, so the
generic plan must include *all* partitions.

Note that pruning does occur during execution (*run-time pruning*) using the parameter values
provided be `EXECUTE`, though there are a number of steps that must occur before the pruning
occurs, each of which take O(n) time, where n is the number of partitions present in the
generic plan.  One of those steps, the most expensive one, is the plancache's validation of
the plan which must lock all relations mentioned in the plan.  The more partitions there are,
the longer it will take this validation step to finish.
