---
layout: writing
title: "Postgres 13: Partitioned tables can now be replicated"
---
# Postgres 13: Partitioned tables can now be replicated

If your shop uses logical replication to selectively replicate tables between two
PostgreSQL clusters, you may have been faced with a frustrating situation that would
start with encountering this error on the publishing server:

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
set must be present on the receiving end.  Actually, any `INSERT`, `UPDATE`,
`DELETE` operations on partitioned tables are physically applied to its leaf
partitions and hence the operation's `WAL` record contains the leaf partitions'
schema information.  Because logical replication publisher plugin (`pgoutput`)
generates records to publish by decoding `WAL`, individual logical changes to be
published contain leaf partition information. That is why the receiving end must
have the same leaf partitions present.

Optionally, you can ask the partitioned table changes to be published using its
own schema, which can be done with a new publication parameter:

```
CREATE PUBLICATION "pub" FOR TABLE "foo" WITH (publish_via_partition_root = true);
```

With this setting, for each decoded change, `pgoutput` maps the leaf partition
information contained in it back to the ancestor that is actually present in the
publication.  This ancestor in most cases is the root partitioned table in which
case changes of all leaf partitions contained in a partition tree are published
using the root partitioned table's schema.  One may also publish only a subtree
of a multi-level partition tree by adding the subtree's root table to the
publication, in which case only leaf partitions of that subtree are published.
