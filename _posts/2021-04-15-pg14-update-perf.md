---
layout: writing
title: "Postgres: UPDATE will work differently in v14"
tags: [writing, pg]
last_updated: 2021-04-15
---
# Postgres: UPDATE will work differently in v14

April 15, 2021 (draft)

Tom Lane committed a few changes ([86dc90056d](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=86dc90056d),
[c5b7ba4e67a](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c5b7ba4e67a),
[a1115fa0782](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1115fa0782))
to the v14 development branch that will help `update` and `delete` queries run
little a bit faster, especially on tables with many columns of which only few are
typically updated in a given query. More importantly, those changes allowed the
refactoring of some lagacy code for handling `update` and `delete` queries on
partitioned tables, which will make them run still faster compared to v13, and
will enable them to use execution-time partition pruning (only `select` queries
could use execution-time pruning before).  These and some other improvements in
the execution of `update`/`delete` plans will allow prepared `update` and `delete`
queries on partitioned tables run faster compared to v13, especially at higher
partition counts.

To understand what changed, let's consider how Postgres typically carries out
an update statement. The plan for an `update` consists of a node to retrieve the
rows to be updated and another node on top that invokes the target table's
access method routine to perform the actual update and peforms other auxiliary
actions like updating the indexes, fire triggers, etc.  While the latter (the
top-level node) works mostly the same in v14, there's a new task for it now due
to some changes made to the output format of the former (the node producing the
rows to be updated).  Previously, that node produced the whole *new* row that
the top-level node could pass as-is to the table access method, which in turn
would use it replace the old version.  This is how it would look:


```
$ psql
psql (13.3)
Type "help" for help.

postgres=# create table foo (a int, b int, c int);
CREATE TABLE
postgres=# explain verbose update foo set a = a + 1 where b = 1;
                            QUERY PLAN
-------------------------------------------------------------------
 Update on public.foo  (cost=0.00..35.52 rows=10 width=18)
   ->  Seq Scan on public.foo  (cost=0.00..35.52 rows=10 width=18)
         Output: (a + 1), b, c, ctid
         Filter: (foo.b = 1)
(4 rows)
```

Note the `Output: (a + 1), b, c, ctid` line of the `Seq Scan` node.  Along with
the new value for the `a` column that is changed in the statement, it also
contains the values for unchanged columns `b`, `c`.  (Note that they come from
the old row that was read from the relation file and matched against the
`where` quals by the scan node.)

In v14, the scan node no longer outputs the values for `b` and `c`, the unchanged
columns:

```
$ psql
psql (14devel)
Type "help" for help.

postgres=# create table foo (a int, b int, c int);
CREATE TABLE
postgres=# explain verbose update foo set a = a + 1 where b = 1;
                            QUERY PLAN
-------------------------------------------------------------------
 Update on public.foo  (cost=0.00..35.52 rows=0 width=0)
   ->  Seq Scan on public.foo  (cost=0.00..35.52 rows=10 width=10)
         Output: (a + 1), ctid
         Filter: (foo.b = 1)
(4 rows)
```

The scan node now only outputs the values for the changed columns and so the
top-level node must fetch the old row again (not as expensive as it may sound)
to form the full *new* row.  `EXPLAIN` unfortunately isn't of much help to
figure out where exactly the new row gets formed, but maybe it's not that
important for users to know.

One performance benefit of this new arrangement is that the scan node now
needs to pass narrower tuples up to the top-level node.  Because that passing of
tuples across the nodes occurs by copying the data column-by-column (whether by
value or by reference), and as there will now be fewer columns to be passed
across due to this change (only the changed columns), one can expect this
overhead to be less.  That effect would be even more pronounced if the scan
node is not a simple table scan like in the above example, but say a join, in
which case there are more plan levels for the data to have to be passed across.

Okay, so how does this help partitioning?

Because the old way of carrying out updates needed the scan nodes to produce
a full new row matching the target table's schema, the plan would need to
contain a separate node for each child table when updating inherited/partitioned
tables.  Remember that non-partitioning inheritance allows each child table to
have their own columns, so this hassle was necessary in that case, but an
annoyance for partitioning which doesn't allow such thing.  This is how it looked:

```
postgres=# create table foo_child1 (d int) inherits (foo);
CREATE TABLE
postgres=# create table foo_child2 (d int, e int) inherits (foo);
CREATE TABLE
postgres=# explain verbose update foo set a = a + 1 where b = 1;
                                  QUERY PLAN
-------------------------------------------------------------------------------
 Update on public.foo  (cost=0.00..64.42 rows=18 width=24)
   Update on public.foo
   Update on public.foo_child1 foo_1
   Update on public.foo_child2 foo_2
   ->  Seq Scan on public.foo  (cost=0.00..0.00 rows=1 width=18)
         Output: (foo.a + 1), foo.b, foo.c, foo.ctid
         Filter: (foo.b = 1)
   ->  Seq Scan on public.foo_child1 foo_1  (cost=0.00..33.15 rows=9 width=22)
         Output: (foo_1.a + 1), foo_1.b, foo_1.c, foo_1.d, foo_1.ctid
         Filter: (foo_1.b = 1)
   ->  Seq Scan on public.foo_child2 foo_2  (cost=0.00..31.27 rows=8 width=26)
         Output: (foo_2.a + 1), foo_2.b, foo_2.c, foo_2.d, foo_2.e, foo_2.ctid
         Filter: (foo_2.b = 1)
(13 rows)
```

Now that the scan node only needs to produce the updated columns in its
output, and only the columns that are present in the root parent table
(and hence all of the child tables) can be updated, there's no need to
create a node for each child.  Instead, there only needs to be one node
for the root parent that covers all the children, essentially what you'd
get if you were only `select`ing the columns to be updated from the root
parent table, which looks like this now:

```
postgres=# create table foo_child1 (d int) inherits (foo);
CREATE TABLE
postgres=# create table foo_child2 (d int, e int) inherits (foo);
CREATE TABLE
postgres=# explain verbose update foo set a = a + 1 where b = 1;
                                        QUERY PLAN
-------------------------------------------------------------------------------------------
 Update on public.foo  (cost=0.00..64.69 rows=0 width=0)
   Update on public.foo foo_1
   Update on public.foo_child1 foo_2
   Update on public.foo_child2 foo_3
   ->  Result  (cost=0.00..64.69 rows=18 width=14)
         Output: (foo.a + 1), foo.tableoid, foo.ctid
         ->  Append  (cost=0.00..64.47 rows=18 width=14)
               ->  Seq Scan on public.foo foo_1  (cost=0.00..0.00 rows=1 width=14)
                     Output: foo_1.a, foo_1.tableoid, foo_1.ctid
                     Filter: (foo_1.b = 1)
               ->  Seq Scan on public.foo_child1 foo_2  (cost=0.00..33.12 rows=9 width=14)
                     Output: foo_2.a, foo_2.tableoid, foo_2.ctid
                     Filter: (foo_2.b = 1)
               ->  Seq Scan on public.foo_child2 foo_3  (cost=0.00..31.25 rows=8 width=14)
                     Output: foo_3.a, foo_3.tableoid, foo_3.ctid
                     Filter: (foo_3.b = 1)
(16 rows)
```

There are still as many nodes to scan children as there were before,
but because they are added at the bottom-most level of the plan tree,
not at the top, they can be added more cheaply now.

<!--
Because scan nodes no longer emit columns that are not changed, the output
looks the same for all child relations (some may notice that the new value for
the changed column `a` (that is, `a + 1`) is now computed by a separate node
that is above the scan node, but that's a deficiency of the current
implementation when dealing with traditional inheritance).

Anyway, this simple change that the scan nodes no longer have to emit unchanged
columns allows the planner to avoid making the scan node for each child relation
through a separate replanning iteration, which would previously be needed to
add unchanged columns in the scan nodes' output target lists.  Now the scan nodes
look just as they would if it were for a `select` query, appearing as leaf nodes
of a single plan.  The new system column `tableoid` is there to identify which
child table a given tuple to be updated comes from, which is now necessary,
because the target relations can no longer mapped one-to-one with their
corresponding subplans.
-->
