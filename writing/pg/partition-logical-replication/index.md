---
layout: writing
title: "Postgres 13: Partitioned tables can now be replicated"
---
# Postgres 13: Partitioned tables can now be replicated
## ...slightly more conveniently

If your shop uses logical replication to selectively replicate tables between two
PostgreSQL clusters and you also happen to rely on partitioning for some of those
tables, you may have been annoyed this error on the publishing server:

```
CREATE PUBLICATION "pub" FOR TABLE "foo";
ERROR:  "foo" is a partitioned table
DETAIL:  Adding partitioned tables to publications is not supported.
HINT:  You can add the table partitions individually.
```

That is, you cannot replicate partitioned tables directly.  As the hint says, you
can replicate the individual (leaf) partitions by explicitly adding them to the
publication, provided the set of partitions on the receiving server matches one-to-one
with the published partitions.  It is perhaps manageable by having your partitioning DDL
script also have some code to manage the publication status of individual
partitions.

With Postgres 13, partitioned tables can now be directly added to publications.
By default, publishing a partitioned table by adding it to publication is really
just a shorthand for publishing all of its present and future leaf partitions.
This still implies though that matching partitions must be present on the receiving
server. Actually, `INSERT`, `UPDATE`, `DELETE` operations on a partitioned table are
physically applied to its leaf partitions and so each operation's `WAL` record
(or specifically logical information contained in it) contains the leaf partitions'
schema information.  Because logical replication publisher plugin (`pgoutput`)
generates logical change records to publish by decoding `WAL`, those records thus
contain leaf partition information. That is why the receiving server must have the
same leaf partitions present to be able to consume the changes.

If that sounds too restrictive, one can now ask the partitioned table changes to be
published using its own schema as follows:

```
CREATE PUBLICATION "pub" FOR TABLE "foo" WITH (publish_via_partition_root = true);
```

With this, for each decoded change, `pgoutput` maps the leaf partition information
contained in decoded change record back to the ancestor that is actually present in
the publication.  This ancestor in most cases is the root partitioned table, in which
case, changes of all leaf partitions contained in the partition tree are published
using the root partitioned table's schema.  One may also publish only a subtree of a
multi-level partition tree by adding the subtree's root table to the publication, in
which case only leaf partitions of that subtree are published.

This feature to replicate using an ancestor's schema allows the receiving end to
consume the received changes without having the matching leaf partitions present,
as is required by default.  The changes can be received into either a non-partitioned
table or a  partitioned table of the same name as the published ancestor.  The
receiving table may also contain a different set of partitions than those of the
published table.

Actually, receiving changes into a partitioned table is also something that wasn't
allowed before; you may have seen the following error on a receiving server:

```
create subscription sub connection '$connection_string' publication pub;
ERROR:  cannot use relation "public.foo" as logical replication target
DETAIL:  "public.foo" is a partitioned table.
```

Upgrading the receiving server to Postgres 13 will fix that too.

To summarize:

1. On publishing server, you can now add partitioned tables directly to
publications, causing the changes of its leaf partitions to be published.

2. Changes can be published using the name of the ancestor (typically the
root partitioned table) that's actually present in the publication by
enabling the new publication parameter `publish_via_partition_root`, which
is disabled by default, meaning the changes are published using the name
of the leaf partition that is actually changed by the replicated operation.

3. On receiving server, changes can now be received into partitioned tables,
whereas previously only non-partitioned tables were allowed.  This
limitation meant that partitions would have to match one-to-one.

If performance of replication is something you care about, then it’s better to
avoid relying on #2 and #3 above, because there is a certain overhead associated
with it, even though very minimal. Especially, if you are ensuring that matching
leaf partitions are present on both servers, using only #1 gets the job done.
