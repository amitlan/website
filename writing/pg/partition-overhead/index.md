---
layout: writing
title: "Partitioning is overhead"
---
# Partitioning is overhead
## ...sometimes very useful

Partitioning a table can sometimes improve performance, but mostly it seems to be doing the
opposite -- slow queries down, something people occasionally even report as a bug.

In fact, partitioning a table can only improve the performance of deleting records from a table,
because it can be as simple as dropping a partition, which involves simply unlinking the partition's
disk file, so it would normally finish in an instant.  On the other hand, deleting records from a
table that is not partitioned involves reading each record from the table's file, matching it against
the query's condition, marking those that match the condition as deleted, and finally writing the marked
records back into the file.
