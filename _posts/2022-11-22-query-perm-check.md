---
layout: writing
title: "Postgres: more on performance of prepared statements with partitioning"
tags: [writing, pg]
last_updated: 2022-11-22
---
# Postgres: more on performance of prepared statements with partitioning

Nov 22, 2022

In a previous [post](https://amitlan.com/2022/05/16/param-query-partition-woes.html), where I
described the performance problems when using prepared statements with partitioning, I
mentioned a [patch](https://www.postgresql.org/message-id/CA%2BHiwqFGkMSge6TgC9KQzde0ohpAycLQuV7ooitEEpbKB0O_mg%40mail.gmail.com)
that I have proposed to alleviate the performance bottleneck caused by locking to some degree.
The following graph shows a comparison of the TPS performance of `pgbench -T60 --protocol=prepared`
with varying number of partitions (I have set `plan_cache_mode = force_generic_plan` in
postgresql.conf to ensure the benchmark measures the performance with cached generic plans.)

Without applying the patch, this is what the `perf` profile of a Postgres backend process
looks like at low partition counts (say, 32):

```
-   97.99%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main                             ◆
     __libc_start_main                                                                                ▒
     main                                                                                             ▒
     PostmasterMain                                                                                   ▒
   - ServerLoop                                                                                       ▒
      - 96.13% PostgresMain                                                                           ▒
         + 39.86% PortalStart                                                                         ▒
         + 12.67% finish_xact_command                                                                 ▒
         - 11.09% GetCachedPlan                                                                       ▒
            + 9.09% LockRelationOid                                                                   ▒
            + 0.88% RevalidateCachedQuery                                                             ▒
         + 9.53% pq_getbyte                                                                           ▒
         + 7.23% PortalRun                                                                            ▒
         + 5.36% ReadyForQuery                                                                        ▒
         + 1.85% start_xact_command                                                                   ▒
         + 0.69% EndCommand                                                                           ▒
           0.54% OidInputFunctionCall                                                                 ▒
           0.50% CreatePortal                                                                         ▒
```

I had mentioned in the previous post that locking that's performed as part of validating a cached plan
constitutes a significant portion of the overall query execution time.  `PortalStart()` and `PortalRun()`
in the above stack have to do with the actual execution of the plan.  Process spends a non-trivial
amount of time in `GetCachedPlan()`, locking the relations referenced in the plan.  This gets worse as
the partition count increases; the following is the profile when the same benchmark is repeated with
with 2048 partitions:


```
-   99.71%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main                             ◆
     __libc_start_main                                                                                ▒
     main                                                                                             ▒
     PostmasterMain                                                                                   ▒
   - ServerLoop                                                                                       ▒
      - 99.54% PostgresMain                                                                           ▒
         - 50.57% GetCachedPlan                                                                       ▒
            + 46.76% LockRelationOid                                                                  ▒
              1.07% IsSharedRelation                                                                  ▒
         + 40.37% finish_xact_command                                                                 ▒
         + 4.56% PortalStart                                                                          ▒
         + 1.60% PortalRun                                                                            ▒
         + 0.72% pq_getbyte                                                                           ▒
         + 0.65% ReadyForQuery                                                                        ▒
```

Almost half of the query execution is time spent locking the partitions, all 2048 of them in this case.

The idea of the proposed patch is to reduce the set of partitions that need to be locked to validate
a plan to just those that will be scanned during query execution, that is, skipping the locking for
those that can be pruned.

Here's the profile with 32 partitions, With the patch applied:

```
-   98.04%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main                             ◆
     __libc_start_main                                                                                ▒
     main                                                                                             ▒
     PostmasterMain                                                                                   ▒
   - ServerLoop                                                                                       ▒
      - 96.01% PostgresMain                                                                           ▒
         + 36.86% PortalStart                                                                         ▒
         + 11.11% pq_getbyte                                                                          ▒
         - 10.40% GetCachedPlan                                                                       ▒
            + 7.90% ExecutorDoInitialPruning                                                          ▒
            + 1.34% RevalidateCachedQuery                                                             ▒
         + 9.10% ReadyForQuery                                                                        ▒
         + 8.51% PortalRun                                                                            ▒
         + 8.31% finish_xact_command                                                                  ▒
         + 1.75% start_xact_command                                                                   ▒
         + 0.85% EndCommand                                                                           ▒
           0.66% OidInputFunctionCall                                                                 ▒
           0.51% CreatePortal                                                                         ▒
```

Hmm, the profile itself doesn't quite tell that the process is now able to spend relatively more time
doing the work of executing the query, but the time spent under `GetCachedPlan()` is now shown as
being spent doing the pruning.  So, not much of an improvement compared with when that time was being
spent doing the locking.

But now look at the profile for 2048 partitions:


```
-   96.97%     0.00%  postgres  libc-2.17.so        [.] __libc_start_main                             ◆
     __libc_start_main                                                                                ▒
     main                                                                                             ▒
     PostmasterMain                                                                                   ▒
   - ServerLoop                                                                                       ▒
      - 95.52% PostgresMain                                                                           ▒
         + 33.99% PortalStart                                                                         ▒
         + 15.54% finish_xact_command                                                                 ▒
         + 14.18% PortalRun                                                                           ▒
         - 8.84% GetCachedPlan                                                                        ▒
            + 6.82% ExecutorDoInitialPruning                                                          ▒
            + 0.96% RevalidateCachedQuery                                                             ▒
         + 8.68% pq_getbyte                                                                           ▒
         + 5.05% ReadyForQuery                                                                        ▒
         + 1.48% start_xact_command                                                                   ▒
```

Unlike without the patch, the time spent under `GetCachedPlan()` hasn't ballooned to over 50%, but has
stayed around the same as with 32 partitions.  Comparing the TPS figure is even more helpful -- almost
10x improvement!


![v16 prepared generic plan tps for partitioned tables](https://s3.ap-northeast-1.amazonaws.com/amitlan.com/files/param-partition-woes-img3.png)


