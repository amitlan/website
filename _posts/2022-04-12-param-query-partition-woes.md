---
layout: writing
title: "Postgres: woes of parameterized queries with partitioning"
tags: [writing, pg]
last_updated: 2022-05-11
---
# Postgres: woes of parameterized queries with partitioning

May 11, 2022

Parameterized queries (aka prepared statements) suffer when you partition tables
mentioned in those queries.  Let's see why.

Using the parameterized queries feagure allows a client to issue a query in two
stages.  With Postgres, those two stages involve using the statement `PREPARE a_name AS <SQL>`
the first time to pass the query text to the server to be remembered by the given
name, followed by multiple invocations of the statement `EXECUTE a_name` each of
which will compute and return the query's output rows.  The backend process which
receives the `PREPARE` will parse, analyze, rewrite the query text and store the
resulting C struct holding the parse tree in a process-local hash table using the
name given in `PREPARE` as the key.  Each of the subsequent `EXECUTE` statements
sent to the same backend process will look up the query's parse tree using the name
given and execute its plan (either one created from scratch or a cached one, more on
this in a bit) to produce the query's result rows.

The point of this exercise is that it can save CPU cycles by not redoing the
parse/analyze/rewrite processing on every request, which is fine because the result
of that processing would be the exact same internal parse tree unless some object
mentioned in the query was changed by DDL, something that tends to happen rather
unfrequently.  Even more CPU cycles are saved if the `EXECUTE` step is able to use
a plan that is also cached, instead of building it from scratch for that particular
execution.

It is easy to check the performance benefit of performoing the queries this way using
the handy `pgbench` tool. pgbench allows to specify which protocol to use when running
the individual queries in the benchmark using the parameter `--protocol=querymode`.
The value *simple* instructs it to execute the queries in one go and *prepared* to
use the 2-stage method described above.

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
