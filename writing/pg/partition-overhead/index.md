---
layout: writing
title: "Partitioning is overhead"
---
# Partitioning is overhead
## ...sometimes very useful

Partitioning a table can sometimes improve performance, but mostly it seems to be doing the
opposite -- slow queries down, something people occasionally even report as a bug.

In fact, partitioning a table can only improve the performance of deleting records from a table,
because it can be as simple as dropping a partition.  Dropping a partition involves simply unlinking
the partition's disk file, so it would normally finish in an instant.  On the other hand, deleting
records from a table that is not partitioned involves reading each record from the table's file,
matching it against the query's condition, marking those that match the condition as deleted, and
finally writing the marked records back into the file.

For any other operation, partitions underlying a partitioned table incur overhead, because Postgres
must consider their properties in addition to the table's own.  Consider selecting from the table.
To perform the select operation, Postgres will make a plan, which for a regular non-partitioned
table looks like this:

```
                      QUERY PLAN                       
-------------------------------------------------------
 Seq Scan on foo  (cost=0.00..35.50 rows=2550 width=4)
(1 row)
```

If `foo` is a partitioned table with partitions `foo_1`, `foo_2`, `foo_3`, the plan looks like this:

```
                           QUERY PLAN                           
----------------------------------------------------------------
 Append  (cost=0.00..121.80 rows=6120 width=12)
   ->  Seq Scan on foo_1  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_2  (cost=0.00..30.40 rows=2040 width=12)
   ->  Seq Scan on foo_3  (cost=0.00..30.40 rows=2040 width=12)
(4 rows)
```

Obviously, the plan looks "bigger".  Actually, it's not just that it looks big, but Postgres needed to put
proportionally more time in making it that big.
