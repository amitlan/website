---
layout: writing
title: "Postgres: where is the data"
tags: [writing, pg]
last_updated: 2020-05-14
---
# Postgres: where is the data

June 29, 2020

Postgres stores the data that one inserts into tables in files, where there is one for
each table (actually more than one if it's bigger than a gigabyte), using a Postgres-
specific binary format.

```
create table foo (a text);
insert into foo values ('xyxyxy'), ('ababab');
select pg_relation_filepath('foo'::regclass);
 pg_relation_filepath 
----------------------
 base/13586/22408
(1 row)
```

The function `pg_relation_filepath()` gives the path of the file, relative
to root of the data directory, where a given relation's contents are
*eventually* flushed down to.  The contents can be inspected.

```
$ hexdump -C $PGDATA/base/13586/16384
00000000  00 00 00 00 50 69 5e 01  00 00 00 00 20 00 c0 1f  |....Pi^..... ...|
00000010  00 20 04 20 00 00 00 00  e0 9f 3e 00 c0 9f 3e 00  |. . ......>...>.|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001fc0  f9 01 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001fd0  02 00 01 00 02 08 18 00  0f 61 62 61 62 61 62 00  |.........ababab.|
00001fe0  f9 01 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001ff0  01 00 01 00 02 08 18 00  0f 78 79 78 79 78 79 00  |.........xyxyxy.|
00002000
```

You can see the strings that were inserted, inter-mixed with some bookkeeping
data that only Postgres knows how to interpret.

What about partitioned tables?

```
create table bar (a text) partition by list (a);
create table bar1 partition of bar for values in ('ababab');
create table bar2 partition of bar for values in ('xyxyxy');
insert into bar values ('xyxyxy'), ('ababab');
select pg_relation_filepath('bar'::regclass);
 pg_relation_filepath 
----------------------
 
(1 row)
```

Yep, `bar`, the parent table, has no file.  Actually, in Postgres, a partitioned
table is merely a logical relation that you can refer to in SQL statements, but
has no physical assets.  Physically, the data goes into partitions, which
individually behave just like normal tables.

```
select pg_relation_filepath('bar1'::regclass);
 pg_relation_filepath 
----------------------
 base/13586/16393
(1 row)

select pg_relation_filepath('bar2'::regclass);
 pg_relation_filepath 
----------------------
 base/13586/16399
(1 row)

$ hexdump -C $PGDATA/base/13586/16393
00000000  00 00 00 00 88 97 60 01  00 00 00 00 1c 00 e0 1f  |......`.........|
00000010  00 20 04 20 00 00 00 00  e0 9f 3e 00 00 00 00 00  |. . ......>.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001fe0  fd 01 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001ff0  01 00 01 00 02 08 18 00  0f 61 62 61 62 61 62 00  |.........ababab.|
00002000

$ hexdump -C $PGDATA/base/13586/16399
00000000  00 00 00 00 48 97 60 01  00 00 00 00 1c 00 e0 1f  |....H.`.........|
00000010  00 20 04 20 00 00 00 00  e0 9f 3e 00 00 00 00 00  |. . ......>.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001fe0  fd 01 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001ff0  01 00 01 00 02 08 18 00  0f 78 79 78 79 78 79 00  |.........xyxyxy.|
00002000
```

Indexes are similarly files all the way down.

```
create index on foo (a);
insert into foo values ('pqpqpq');
select pg_relation_filepath('foo_a_idx'::regclass);
 pg_relation_filepath 
----------------------
 base/13586/16430
(1 row)

$ hexdump -C $PGDATA/base/13586/16430
00000000  00 00 00 00 38 d2 64 01  00 00 00 00 48 00 f0 1f  |....8.d.....H...|
00000010  f0 1f 04 20 00 00 00 00  62 31 05 00 04 00 00 00  |... ....b1......|
00000020  01 00 00 00 00 00 00 00  01 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 f0 bf  |................|
00000040  01 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001ff0  00 00 00 00 00 00 00 00  00 00 00 00 08 00 00 00  |................|
00002000  00 00 00 00 c8 dc 64 01  00 00 00 00 24 00 c0 1f  |......d.....$...|
00002010  f0 1f 04 20 00 00 00 00  e0 9f 20 00 c0 9f 20 00  |... ...... ... .|
00002020  d0 9f 20 00 00 00 00 00  00 00 00 00 00 00 00 00  |.. .............|
00002030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00003fc0  00 00 00 00 03 00 10 40  0f 70 71 70 71 70 71 00  |.......@.pqpqpq.|
00003fd0  00 00 00 00 01 00 10 40  0f 78 79 78 79 78 79 00  |.......@.xyxyxy.|
00003fe0  00 00 00 00 02 00 10 40  0f 61 62 61 62 61 62 00  |.......@.ababab.|
00003ff0  00 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00004000
```

It should be no surprise that the contents of system catalog tables such as
`pg_trigger`, which stores the metadata about triggers, are stored in files,
like any other relation:

```
create function print_new () returns trigger as $$
   begin raise notice '%', new; return new; end;
   $$ language plpgsql;
create trigger foo_insert_trigger
    before insert on foo for row
    execute function print_new();
select pg_relation_filepath('pg_trigger'::regclass);
 pg_relation_filepath 
----------------------
 base/13586/2620
(1 row)

$ hexdump -C $PGDATA/base/13586/2620
00000000  00 00 00 00 d8 6b 65 01  00 00 00 00 1c 00 60 1f  |.....ke.......`.|
00000010  00 20 04 20 00 00 00 00  60 9f 3a 01 00 00 00 00  |. . ....`.:.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001f60  0b 02 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00001f70  01 00 13 00 03 08 20 ff  ff 00 00 00 00 00 00 00  |...... .........|
00001f80  30 40 00 00 00 40 00 00  00 00 00 00 66 6f 6f 5f  |0@...@......foo_|
00001f90  69 6e 73 65 72 74 5f 74  72 69 67 67 65 72 00 00  |insert_trigger..|
00001fa0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001fc0  00 00 00 00 00 00 00 00  00 00 00 00 2f 40 00 00  |............/@..|
00001fd0  07 00 4f 00 00 00 00 00  00 00 00 00 00 00 00 00  |..O.............|
00001fe0  00 00 00 00 60 00 00 00  01 00 00 00 00 00 00 00  |....`...........|
00001ff0  15 00 00 00 00 00 00 00  00 00 00 00 03 00 00 00  |................|
00002000
```

Maybe you can make out the trigger name from the rest of the gibberish.
