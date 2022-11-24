---
layout: writing
title: "Postgres: more on performance of prepared statements with partitioning"
tags: [writing, pg]
last_updated: 2022-11-22
---
# Postgres: more on performance of prepared statements with partitioning

Nov 22, 2022

In a previous [post](https://amitlan.com/2022/05/16/param-query-partition-woes.html), I
described some performance problems when using prepared statements with partitioning that
are caused by excessive locking of partitions causing a bottleneck as the number of
partitions that the table has increases.  I also mentioned a
[patch](https://www.postgresql.org/message-id/CA%2BHiwqFGkMSge6TgC9KQzde0ohpAycLQuV7ooitEEpbKB0O_mg%40mail.gmail.com)
that I have proposed to address that particular problem.  The following graph shows a comparison
of the TPS performance of `pgbench -T60 --protocol=prepared` with varying number of partitions,
initialized using `pgbench -i --partitions=$count`, before and applying that patch.  Note that
I have set `plan_cache_mode = force_generic_plan` in postgresql.conf to ensure the benchmark
measures the performance with cached generic plans.

![v16 prepared generic plan tps for partitioned tables](https://s3.ap-northeast-1.amazonaws.com/amitlan.com/files/unpatched-patch1.png)

In this post, I would like to highlight another bottleneck that pops it head out when the
locking bottleneck is addressed.  Addressing that overhead with
[another patch](https://www.postgresql.org/message-id/CA%2BHiwqGjJDmUhDSfv-U2qhKJjt9ST7Xh9JXC_irsAQ1TAUsJYg%40mail.gmail.com)
that is also in the pipeline for v16 gives a somewhat significant performance improvement,
provided the locking overhead is addressed first.

So let's look at the `perf` profile of a Postgres backend process running the above benchmark
without applying any patch, at low partition counts (say, 32) first:

```
-   97.99%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main
     __libc_start_main
     main
     PostmasterMain
   - ServerLoop
      - 96.13% PostgresMain
         + 39.86% PortalStart
         + 12.67% finish_xact_command
         - 11.09% GetCachedPlan
            + 9.09% LockRelationOid
            + 0.88% RevalidateCachedQuery
         + 9.53% pq_getbyte
         + 7.23% PortalRun
         + 5.36% ReadyForQuery
         + 1.85% start_xact_command
         + 0.69% EndCommand
           0.54% OidInputFunctionCall
           0.50% CreatePortal
```

Process spends a non-trivial amount of time in `GetCachedPlan()`, the backend function that
looks up, validates, and returns a cached plan for a given prepared statement.  One of the
things it does is locking the relations referenced in the plan.  This gets worse as the
partition count increases as can be seen in the following profile taken when the same
benchmark is repeated with with 2048 partitions:


```
-   99.71%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main
     __libc_start_main
     main
     PostmasterMain
   - ServerLoop
      - 99.54% PostgresMain
         - 50.57% GetCachedPlan
            + 46.76% LockRelationOid
              1.07% IsSharedRelation
         + 40.37% finish_xact_command
         + 4.56% PortalStart
         + 1.60% PortalRun
         + 0.72% pq_getbyte
         + 0.65% ReadyForQuery
```

Almost half of the query execution is time spent locking the partitions, all 2048 of them in this case.

The idea of the proposed patch (one mentioned in the 1st paragraph) is to reduce the set of partitions
that are locked by `GetCachedPlan()` to just those that will be scanned during query execution, that is,
skipping the locking for those that can be pruned.

Here's the profile again, with the patch applied:

```
-   96.97%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main
     __libc_start_main
     main
     PostmasterMain
   - ServerLoop
      - 95.52% PostgresMain
         + 33.99% PortalStart
         + 15.54% finish_xact_command
         + 14.18% PortalRun
         - 8.84% GetCachedPlan
            + 6.82% ExecutorDoInitialPruning
            + 0.96% RevalidateCachedQuery
         + 8.68% pq_getbyte
         + 5.05% ReadyForQuery
         + 1.48% start_xact_command
```

Unlike without the patch, the time spent under `GetCachedPlan()` hasn't ballooned to over 50%.
Comparing the TPS figure is even more helpful -- almost 10x improvement!

OK, so where's the other bottleneck?

`PortalStart()`, whose job is to initialize the plan tree for execution spends a non-trivial
amount of time too, found in roughly 40% of the samples.  Expanding that frame, one hopes to
find the culprits:

```
-   97.20%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main
     __libc_start_main
     main
     PostmasterMain
   - ServerLoop
      - 95.61% PostgresMain
         - 34.23% PortalStart
            - 33.62% standard_ExecutorStart
               + 19.52% ExecInitNode
                 13.23% ExecCheckRTPerms
                 0.55% ExecInitRangeTable
         + 15.57% finish_xact_command
         + 14.27% PortalRun
         + 9.27% GetCachedPlan
         + 8.53% pq_getbyte
         + 4.47% ReadyForQuery
         + 1.58% start_xact_command
           0.51% OidInputFunctionCall
```

`ExecCheckRTPerms()` is there to check whether the user has needed permissions on the
table mentioned in the query.  It should really be quick as there is just one table to
be checked in this case (the "root" partitioned table), but the current implementation
is such that it spends `O(n)` amount of time in the number of partitions.  Note that
the code visits each partition, it doesn't actually checks its permissions, so that's
just wasteful looping.

The patch I mentioned in the 2nd paragraph changes things (specifically, the list contained
the plan that records the relations whose permissions should be checked) such that
`ExecCheckRTPerms()` no longer needs to visit each partition.  That reduces the time
spent in that function significantly, as can be seen in the following updated profile
after applying that patch:

```
-   96.37%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main
     __libc_start_main
     main
     PostmasterMain
   - ServerLoop
      - 94.58% PostgresMain
         - 24.37% PortalStart
            - 23.64% standard_ExecutorStart
               + 22.30% ExecInitNode
               + 0.70% ExecInitRangeTable
         + 19.20% finish_xact_command
         + 17.54% PortalRun
         + 9.73% GetCachedPlan
         + 8.90% pq_getbyte
         + 4.86% ReadyForQuery
         + 1.63% start_xact_command
           0.54% OidInputFunctionCall
```

So the time spent in `PortalStart()` goes from 34% down to 24%.  The TPS is slightly improved too
as can be seen in the following graph:

![v16 prepared generic plan tps for partitioned tables_with_permissions_patch](https://s3.ap-northeast-1.amazonaws.com/amitlan.com/files/unpatched-patch1-patch2.png)

While it is unquestionable that these patches are necessary for seeing these bottlenecks out the door,
the changes they make to the query execution pipeline are non-trivial and haven't been fully reviewed
yet.  The work is ongoing and I hope they make the finish line for the next year's v16.

Addendum: The `finish_xact_command()` also spends more time than it really should in this particular
benchmark (generic plan, with many partitions).  There's [work underway](https://www.postgresql.org/message-id/0A3221C70F24FB45833433255569204D1FB976EF%40G01JPEXMBYT05) to fix that too.
