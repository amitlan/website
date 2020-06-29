---
layout: writing
title: "Postgres: where is the data"
tags: [writing, pg]
last_updated: 2020-05-14
---
# Postgres: where is the data

June 29, 2020

Postgres stores the data one tells it to store in files using a Postgres-specific
binary format. 

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
*eventually* flushed down to.

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

What about partitioned tables:

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
