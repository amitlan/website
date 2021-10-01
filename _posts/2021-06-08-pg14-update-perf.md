---
layout: writing
title: "Postgres: UPDATE will work differently in v14"
tags: [writing, pg]
last_updated: 2021-06-08
---
# Postgres: UPDATE will work differently in v14

June 8, 2021

Long story short, Tom Lane committed a few changes
(commmits [86dc90056d](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=86dc90056d),
[c5b7ba4e67a](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c5b7ba4e67a),
[a1115fa0782](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1115fa0782))
to the v14 development branch that will help `update` and `delete` queries run
little a bit faster, especially on tables with many columns of which only a few are
typically updated in a given query. More importantly, those changes allowed the
refactoring of some legacy code for handling `update` and `delete` queries on
partitioned tables, which will make them run still faster compared to v13, and
will enable them to use execution-time partition pruning (only `select` queries
could use execution-time pruning before).  These and some other improvements in
the execution of `update`/`delete` plans will allow prepared `update` and `delete`
queries on partitioned tables run faster compared to v13.

Now the long story.

To understand what changed, let's consider how Postgres carries out
an update statement. The plan for an `update` consists of a node to retrieve the
rows to be updated and another node on top that invokes the target table's
access method routine to perform the actual update and peforms other auxiliary
actions like updating the indexes, fire triggers, etc.  While the latter (the
top-level node) mostly works the same in v14, there's a new task for it now due
to some changes made to the output format of the former (the node producing the
rows to be updated).  Previously, that node produced the whole *new* row that
the top-level node could pass as-is to the table access method, which in turn
would replace the old version with the new one.  This is how it would look as
a plan:


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
tuples across nodes occurs by copying the data column-by-column (whether by
value or by reference depending on the column type), and as there will now be fewer
columns to be passed across due to this change (only the changed columns), one
can expect the overhead of that copying to be less.  That effect would be even
more pronounced if the scan node is not a simple table scan like in the above
example, but say a join, in which case there are more plan levels for the data
to have to be passed across.

Okay, so how does this help partitioning?

Because the old way of carrying out updates needed the scan nodes to produce
a full new row matching the target table's schema, the plan would need to
contain a separate node for each child table when updating inherited/partitioned
tables.  Remember that non-partitioning inheritance allows each child table to
have its own columns, so this hassle was necessary in that case, but an
annoyance for partitioning which doesn't allow such a thing.  This is how it
looked:

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

You may notice that there's a `tableoid` column present in the output that
wasn't there before.  It tells the top-level node the OID of the child
table from which a given row to be updated came from.  Such an identity
column was not necessary before, because each child table would have its own
subplan.  Mapping from `tableoid` to the `update`/`delete` execution state
struct of the relevant child table does add a tiny bit of overhead per row.

Note that there are still as many nodes to scan children as there were before,
but because they are added at the bottom the plan tree, not at the top as
before, they can be added more cheaply.  (This statement only makes sense
if you consider that the amount of work the planner has to do to create a
node at the bottom is less versus creating it at the top.)

While this new arrangement makes it a bit faster to make the plan for `update`
on a partitioned table which is good in and of itself, a more important bit is that
the plan will now have an `Append` node in it to scan the partitions.  What's great
about it is that that means that `update` (and `delete`) can now use execution-time
partition pruning, because the `Append` node provides that ability. Having that ability
allows generic plans that may be used when using prepared statements for `update` and
`delete` to be executed without having to process the partitions that need not
processed per the query's prunable `where` clauses.

With the new execution-time pruning ability and a few other efficiency improvements
in how the plan for an `update` is executed now results in better performance.
Although, there are still architectural inefficiencies left to be fixed, so the
performance still tapers off as the partitions count grows.  To end this post,
here is a graph showing the performance of a prunable prepared `update` of a 10-column
partitioned table with various partition counts.
![v14 prepared ppdate performance for partitioned tables](https://s3-ap-northeast-1.amazonaws.com/amitlan.com/files/pg14-update-perf-partitions.png)
