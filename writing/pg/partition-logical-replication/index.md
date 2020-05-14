---
layout: writing
title: "Postgres 13: Partitioned tables can now be replicated"
---
# Postgres 13: Partitioned tables can now be replicated
## ...slightly more conveniently

If your shop uses logical replication to selectively replicate tables between two
PostgreSQL clusters and you also happen to rely on partitioning for some of your
tables that need to be replicated, you may have been faced with a frustrating
situation that would start with encountering this error on the publishing server:

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
adding them to the publication.  This still implies that a matching partition
set must be present on the receiving end.  Actually, `INSERT`, `UPDATE`,
`DELETE` operations on partitioned tables are physically applied to its leaf
partitions and so each operation's `WAL` record (or specifically its logical
portion) contains the leaf partitions' schema information.  Because logical
replication publisher plugin (`pgoutput`) generates records to publish by
decoding `WAL`, individual logical changes to be published thus contain leaf
partition information. That is why the receiving end must have the same leaf
partitions present to be able to consume the changes, hopefully matching in not
just the name, but also the partition bound.


Optionally, you can ask the partitioned table changes to be published using its
own schema, which can be done with the following new publication parameter:

```
CREATE PUBLICATION "pub" FOR TABLE "foo" WITH (publish_via_partition_root = true);
```

With this otion, for each decoded change, `pgoutput` maps the leaf partition
information contained in change record back to the ancestor that is actually
present in the publication.  This ancestor in most cases is the root partitioned
table, in which case, changes of all leaf partitions contained in the partition
tree are published using the root partitioned table's schema.  One may also
publish only a subtree of a multi-level partition tree by adding the subtree's
root table to the publication, in which case only leaf partitions of that subtree
are published.

This feature to replicate using an ancestor's schema allows the receiving end to
consume the received changes without having the matching leaf partitions present,
as is required when you don't use the new `publish_via_partition_root` option for
publishing.  The changes can be received into either a non-partitioned table or a
partitioned table of the same name as the published ancestor.  The receiving table
may also contain a different set of partitions than those of the published table.
Actually, receiving changes into a partitioned table is also something you couldn't
do before.  You may have seen the following error with a receiving server that's
running on Postgres 12 or less:

```
create subscription sub connection '$connection_string' publication pub;
ERROR:  cannot use relation "public.foo" as logical replication target
DETAIL:  "public.foo" is a partitioned table.
```

Upgrading the receiving server to Postgres 13 will fix that. 
