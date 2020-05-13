---
layout: writing
title: "Postgres 13: Partitioned tables can now be replicated"
---
# Postgres 13: Partitioned tables can now be replicated

If your shop uses logical replication to selectively replicate tables between two
PostgreSQL clusters, you may have faced with a frustrating situation that would start
with encountering this on the publishing server:

```
CREATE PUBLICATION "pub" FOR TABLE "foo";
ERROR:  "foo" is a partitioned table
DETAIL:  Adding partitioned tables to publications is not supported.
HINT:  You can add the table partitions individually.
```

That is, you cannot replicate partitioned tables directly.  As the hint says, you
can replicate the individual (leaf) partitions by explicitly adding them to the
publication, provided your partition set on the receiving end matches one-to-one
with the published set of partitions.  It is perhaps manageable by having your
partitioning DDL script also have some code to manage the publications membership
of individual partitions.

With Postgres 13, partitioned tables can now be directly added to publications.
By default, publishing a partitioned table by adding it to publication is really
just a shorthand for publishing all of its leaf partitions without actually
adding them to the publication.  This means that, by default, a matching partition
set must be present on the receiving end.
