---
layout: writing
title: "Partitioning is overhead"
---
# Partitioning is overhead
## ...sometimes very useful

Partitioning a table in Postgres can sometimes improve performance, but mostly it seems to be doing
the opposite -- slow queries down, something people occasionally even report as a bug.

In fact, the only thing partitioning a table can improve the performance of is deleting records from
the table, because it can be as simple as dropping a partition.  Dropping a partition involves simply
unlinking the partition's disk file, so it would normally finish in an instant.  On the other hand,
deleting records from a table that is not partitioned involves reading every record from the table's
file, matching it against the query's condition, marking those that match the condition as deleted,
and finally writing the marked records back into the file; the more the records to delete and the
bigger the table, the longer this would take.

For other operations, partitions underlying a partitioned table incur overhead, because Postgres
must consider their properties in addition to the table's own.  Consider selecting from the table.
To perform the select operation, Postgres will make a plan, which for a regular non-partitioned
table looks like this:

```
                      QUERY PLAN                       
-------------------------------------------------------
 Seq Scan on foo  (cost=0.00..35.50 rows=2550 width=4)
(1 row)
```

If `foo` were partitioned with partitions `foo_1`, `foo_2`, `foo_3`, the plan would looks like this:

```
                           QUERY PLAN                           
----------------------------------------------------------------
 Append  (cost=0.00..121.80 rows=6120 width=12)
   ->  Seq Scan on foo_1  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_2  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_3  (cost=0.00..30.40 rows=2040 width=12)
(4 rows)
```

Obviously, the plan looks "bigger".  Actually, it's not just that it looks bigger, but Postgres also
puts proportionally more time into making it.  That can be explained as follows: for any table that
must be scanned to answer a query, Postgres planner considers between a few alternatives of the best
way to do the scan -- sequentially go through all the records in the table and match against the
filter, use an index suitable to the filter to get the matching records, use a bitmap scan, etc.
Postgres can only make these decisions for tables that actually store data, that is, have on-disk
file storing the records and indexes pointing into those.  For a table that is partitioned, only its
partitions stores data, whereas the table itself is just a logical object with only logical properties
such as columns, constraints, etc.  So to scan a partitioned table, Postgres must scan all its
partitions and hence the planner must consider whether to use a sequential scan, index scan, bitmap scan,
etc. for each of them.  To see that the planner is actually spending time doing that, compare the
planning times of selecting from a regular non-partitioned table:

```
                      QUERY PLAN                       
-------------------------------------------------------
 Seq Scan on foo  (cost=0.00..35.50 rows=2550 width=4)
 Planning Time: 0.068 ms
(2 rows)
```

a table with 10 partitions:

```                           QUERY PLAN                            
-----------------------------------------------------------------
 Append  (cost=0.00..406.00 rows=20400 width=12)
   ->  Seq Scan on foo_1  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_2  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_3  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_4  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_5  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_6  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_7  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_8  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_9  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_10  (cost=0.00..30.40 rows=2040 width=12)
 Planning Time: 0.268 ms
(12 rows)
```

another with 100:

```                            QUERY PLAN                            
------------------------------------------------------------------
 Append  (cost=0.00..4060.00 rows=204000 width=12)
   ->  Seq Scan on foo_1  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_2  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_3  (cost=0.00..30.40 rows=2040 width=12)
   ...
   ->  Seq Scan on foo_98  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_99  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_100  (cost=0.00..30.40 rows=2040 width=12)
 Planning Time: 2.357 ms
(102 rows)
```

The planning times with 1000 and 10000 partitions are roughly 21 and 252 ms.  All
tables (and partitions) have a primary key index on them, so planner is spending
time considering it.

Although, rarely does one query without a filter, especially a partitioned
one, in which case, planner would need to consider only one or few of all partitions.
But even then the planner is doing slightly more work than if there are no partitions.
For table without partitions:

```
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Index Only Scan using foo_pkey on foo  (cost=0.15..8.17 rows=1 width=4)
   Index Cond: (a = 1)
 Planning Time: 0.098 ms
(3 rows)
```

With 1000 partitions:

```
                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Index Scan using foo_40_pkey on foo_40 foo  (cost=0.15..8.17 rows=1 width=8)
   Index Cond: (a = 1)
 Planning Time: 0.165 ms
(3 rows)
```
