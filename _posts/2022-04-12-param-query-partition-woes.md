---
layout: writing
title: "Postgres: woes of prepared statements with partitioning"
tags: [writing, pg]
last_updated: 2022-05-16
---
# Postgres: woes of prepared statements with partitioning

May 16, 2022

Prepared statements (aka parameterized queries) suffer when you have partitioned
tables mentioned in them.  Let's see why.

Using prepared statements, a client can issue a query in two stages.  First, *prepare*
it using `PREPARE a_name AS <query>`, followed by multiple *executions* using
`EXECUTE a_name`.  During the `PREPARE` step, the backend will only parse-analyze the
query and remember the resulting parse tree in a cache using the provided name as its
lookup key.  During each `EXECUTE` step, the backend will look that parse tree up in
the cache, create or look up a plan for it, and execute it to return the query's output
rows.  Note that both `PREPARE` and subsequent `EXECUTE`s must run in the same connection,
because the storage used for the parse tree and the plan for a given query is process
local at the moment.

The query specified in `PREPARE` may put numbered parameters ($1, $2, ...) in place of
the actual constant values in the query's conditions (like `WHERE`).  The values themselves
are only supplied during a given `EXECUTE` invocation. 
 
The benefit of this 2-stage processing is that a lot of CPU cycles are saved by not redoing
the parse-analysis processing on every execution, which is fine because the parse tree does
not change unless objects mentioned in the query were changed by DDL, something that tends
to happen rather unfrequently.

Even more CPU cycles are saved if the `EXECUTE` step is able to use a plan that is also
cached, instead of building it from scratch for every execution, that is, for each fresh
set of parameter values.

The performance benefit of this 2-stage protocol of performing queries can be easily
verified using the handy `pgbench` tool, which allows specifying which protocol to use
when running the benchmark queries using the parameter `--protocol=querymode`. The value
*simple* instructs it to execute the queries in one go (parse-plan-execute) and *prepared*
to use the 2-stage method described above.

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

The latency average for query `SELECT abalance FROM pgbench_accounts WHERE aid = ?` is 0.031
milliseconds when performed using the 2-stage protocol versus 0.058 when using the simple protocol,
so the caching does seem productive.

It's interesting to look at the mechanism of how the plan itself is cached, because that is where
the problems with partitioned tables can be traced to.  The backend code relies on the plancache
module which hosts the plan caching logic.  When presented with the parse tree of a query, its job
is to figure out if a plan matching the query already exists and if one does, check if it's still
valid by acquiring read locks on all the relations that it scans.  If any of the relations has
changed since the plan was created, this validation step (the locking) detects those changes and
triggers re-planning.

An important bit about such a cached plan is that, when creating it, the plancache would not have
passed the actual values of parameters mentioned in `EXECUTE` to the planner, so it is a *generic*
(parameter-value-independent) plan.  In most cases, the plan is the same as one that the plancache
would have gotten by passing the values, because the planner is smart enough to perform some of its
optimizations even in the absence of the actual constant values to compare against the table/index
statistics, at least for simpler queries.  For example, it can heuristically determine that an index
on `colname` can be used to optimize `colname = $1` in a similar manner as it can be used for
`colname = <actual-constant-value>.

However, parameter values not being available to the planner when creating a generic plan can
be a big problem if the query contains partitioned tables. Without plan-time partition pruning,
which relies on constant values being available for the planner to compare them against partition
bounds found in the system catalog, the generic plan must include *all* partitions.

Fortunately, pruning will occur during execution (*run-time pruning*), though there are a number
of steps manipulating the plan that must occur before the run-time pruning can occur, each of which
currently takes O(n) time, where n is the number of partitions present in the generic plan.  One of
those steps, the most expensive one, is the plancache module's validation of the plan which must lock
all relations mentioned in the plan.  The more partitions there are, the longer it will take this
validation step to finish.  The following graph shows the average latency reported by
`pgbench -S --protocol=prepared` with varying number of partitions (initialized with
`pgbench -i --partitions=N` in each case):

![v15 prepared generic plan latency for partitioned tables](https://s3.ap-northeast-1.amazonaws.com/amitlan.com/files/param-partition-woes-img1.png)

The following graph plots the latencies one may see by forcing the plancache module to create a
parameter value dependent (*custom*) plan on every `EXECUTE` (by setting `plan_cache_mode =
force_custom_plan` in postgresql.conf).  Note that the latency doesn't degrade as it does by the
use of a parameter value independent cached (*generic*) plan, because the planner is able to prune
the unnecessary partitions in that case.

![v15 prepared custom plan latency for partitioned tables](https://s3.ap-northeast-1.amazonaws.com/amitlan.com/files/param-partition-woes-img2.png)

However, forcing re-planning on each `EXECUTE` means leaving the peformance benefit of plan caching
on the table.  So I proposed a [patch](https://commitfest.postgresql.org/38/3478/) to fix the
generic plan validation step such that it only locks the partitions that match the parameter values
mentioned in `EXECUTE` by performing the run-time pruning earlier than the validation step.  Here is
the graph with the patch applied:

![v16 prepared generic plan latency for partitioned tables](https://s3.ap-northeast-1.amazonaws.com/amitlan.com/files/param-partition-woes-img3.png)

As expected, the latency no longer degrades, and perhaps more importantly, is indeed lower than with
the use of a custom plan on each `EXECUTE`.
