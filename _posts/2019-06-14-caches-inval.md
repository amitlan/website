---
layout: writing
title: "Postgres: Caches and invalidation"
tags: [pg]
last_updated: 2019-06-14
---
# Postgres: Caches and Invalidation

June 14, 2019

The other day, while debugging a patch to fix a memory leak bug in Postgres 12,  I got to learn
quite a bit about the *caches* that are used internally in Postgres and the internal machanics of
*invalidating* them when the cached content becomes stale.  I thought to write down the details
while they are still in the accessible part of my brain.

## What's a Cache

Postgres stores the metadata of database objects, such as tables, attributes, functions, operators,
etc. in tables called *system catalogs*, which are very much like tables that contain user data.
They can be queried using SQL by users just like they would their own tables.  Also, although
they can technically be modified by `INSERT`, `UPDATE`, `DELETE`, doing so is not recommeded in
the normal database operation; only modifications they ever receive are those automatically
performed by the system when handling DDL operations against the database objects.

The primary user of the system catalogs is Postgres itself, which needs to refer to their content
numerous times when processing any given non-trivial user request (such as an SQL query).  Given
that requirement, each access to a system catalog has to be very fast, which won't be the case
if Postgres used SQL queries every time. (Yes, internal interfaces to do exactly that do exist!)
If it had, most of its time would be spent parsing (and planning) the queries, whereas it would
be much faster to read the records directly from the table, possibly using an index for fast
lookup of the keys, which is the execution plan it would've come up with for the query anyway.
Actually, that's pretty close to what it actually does, except it doesn't have to go to the table on
every access, because what's the point of going to the table (and possibly the file system) if the
same record will be returned every time for given keys.  That's true most of the time, because
catalog records rarely change, provided there is not much DDL and other maintenance activity.
Postgres leverages this fact and *caches* the records after fetching them from the table, that is,
saves them in local memory using data structures that allow efficient look up of the keys, so that
subsequent accesses to the same records can be served from there.  These caches are internally
known as *syscaches*.  As you can imagine, caches make looking up individual catalog records several
orders of magnitude faster than if those records were to be retrieved by issuing SQL queries.

Syscaches are exposed to the rest of the system via a set of functions and macros whose arguments
consist of the syscache's name and key(s) to be searched.  For example, there is a syscache for
`pg_class` (the system catalog used for storing relation metadata) to look up a relation using keys
that consist of schema name and relation name.  There are a few other caches beside syscaches, such
as the *relcache*, which cache information that is more complex than a single system catalog
record, but share the same mechanics as syscaches for cache invalidation. For brevity, I won't
get into the details of differences between syscaches and other types of caches.

On the inside of each of the syscaches (and other types of caches) is a hash table to efficiently
look up the keys that have already been processed previously and so any tuples matching the keys
already read into the hash table from the underlying table sitting on the disk.  If a key is not
found in the hash table, it is looked up in the catalog table using an index earmarked for the
cache, which coordinates with the table access module to fetch the latest version of the tuple (if
still present).  This is the case of a *cache miss*. As just mentioned, when read from the table
in the case of a cache miss, one can be sure that the returned tuple is the latest version of the
record for the database object, because the table access method layer is explicitly told to get one.
However, the keys that *are* found in the syscache (its hash table) may point to an outdated version
of the database object, if the object was modified or deleted by a concurrent backend process.

## Caches and Concurrent DDL

One may wonder how the system works sanely if the information it retrieves from one of the many caches
may be obsoleted by concurrent DDL changing the information in the underlying system catalog tables.
Before we discuss the cache *invalidation* machanism which allows processes to keep the cache
entries up to date, a few words about *locking*.

Locking of a database object by a session prevents any concurrent changes to it by other sessions,
at least as long as the concurrent changes would require a lock on the object that conflicts with
the lock held by the session.  Once a given database object is locked, the session can continue
to use the object without worrying about its cached metadata going stale.  Some non-critical
changes can be made concurrently despite the lock held by a session on the relation. In such cases,
some information about the relation that is currently being used by the session might be obsolete,
but that's considered to be OK.  An example is concurrently running `ANALYZE` command on relation,
which updates its statistics stored in the catalog. If the current session is running `SELECT` on
the relation, it might miss those updates to the statistics, but the `SELECT` can still finish,
albeit with a possibly inferior plan.  In fact, standalone objects other than table, such as
functions, languages (which have their own caches) don't even have a notion of locking.  So, it's
possible for a session to run with stale definitions of such objects if they have been modified
concurrently, which again is not very harmful.

While locking prevents concurrent changes that the current process cannot tolerate, what about the
changes that may have already occurred before the lock was obtained but not yet seen by us? That's
what cache invalidation is meant to take care of.  It refers to discarding the entries in the cache
that have become stale due to catalog data changing, so that new ones could be made on the next
access to the same catalog data.

## Life of an Invalidation Message

An invalidation message is created every time a system catalog record is changed or deleted due to
executing some DDL operation, affecting one or more database objects.  The message will be shared with
other running processes so that they can discard the cache entries that have become stale due to those
changes.  All the  messages generated in a given (sub-) transaction are gathered in a data structure,
from where they will be flushed into a shared memory queue when the (sub-) transaction is committed;
they will be discarded if the (sub-) transaction is aborted.  Each message contains enough
information to identify both an object that's changed and the cache which might contain a copy of
tuple of the object.

Messages sit in the shared memory queue until they are read by all other processes.  Normally,
receiving processes read the messages only at certain times, so it would appear that messages may 
continue to sit on the shared memory indefinitely if the receiving processes are idle (that is not
processing any user requests) or busy doing other stuff such that they don't have time to read these
messages. In unfortunate cicurmstances where this shared memory space is no longer available for a
process to store new messages, that process will have to take up the task of cleaning things up.
(Actually, this kind of cleaning up is done proactively, so it's rare to run out of space.) To discard
old messages, it must be sure that all other processes have read them.  If some processes haven't been
able to for aforementioned reasons, it will have to explicitly send a signal to the lagging processes
to ask them to *catch up*.  Once the lagging processes catch up, it's free to discard those messages.

In the receiving process, all unread messages on the shared memory queue are first copied into
local memory and having done so is advertised via shared memory so that other processes can know
how far it has managed to catch up.  The exact time when a receiving process will read messages from
the shared memory is not synchronized with when they are put there, because it would be busy doing
other stuff.  In fact, there are two main events when the receiving processes try to read those
messages: 1. Upon starting a new transaction or a new command in the same transaction, 2. Upon
locking a relation.  The latter allows to absorb changes that may have occurred to the relation
since the last time invalidation messages were read and processed in the same transaction, up to the
instant the process got the lock.

Processing a message results in first checking if the catalog tuple specified in the message is
currently in the cache (the syscache identity is also specified in the message), and if it is, remove
it from the cache's hash table.  The next time that tuple is requested, it will be freshly read from
the underlying catalog table and added to the hash table, so the subsequent accesses will read the new
value.  If the process has locked the particular database object, preventing concurrent processes from
modifying it anymore, it can continue to use the cached tuple until releasing the lock.

# Concluding Remarks

There are many other details that I haven't mentioned, but the ones I have should give a rough idea
of one of the many internal mechanisms that allow Postgres to return user queries so quickly.  Now,
while that quickness itself is perhaps something we all take for granted in 2019, I'd like to point
out how neat and robust Postgres' caching looked to my eyes.  That is not of course to say that it
cannot be improved further...

# References

See the C source code files if interested.

* Invalidation dispatch: [src/backend/utils/cache/inval.c](https://git.postgresql.org/gitweb/?p=postgresql.git;f=src/backend/utils/cache/inval.c;hb=HEAD)
* Invalidation message sharing dispatch: [src/backend/storage/ipc/sinval.c](https://git.postgresql.org/gitweb/?p=postgresql.git;f=src/backend/storage/ipc/sinval.c;hb=HEAD)
* Invalidation message sharing data structures: [src/backend/storage/ipc/sinvaladt.c](https://git.postgresql.org/gitweb/?p=postgresql.git;f=src/backend/storage/ipc/sinvaladt.c;hb=HEAD)
* Invalidation interface: [src/include/utils/inval.h](https://git.postgresql.org/gitweb/?p=postgresql.git;f=src/include/utils/inval.h;hb=HEAD)
* Invalidation message sharing interface: [src/include/storage/sinval.h](https://git.postgresql.org/gitweb/?p=postgresql.git;f=src/include/storage/sinval.h;hb=HEAD)
* Invalidation message sharing data structures interface: [src/include/storage/sinvaladt.h](https://git.postgresql.org/gitweb/?p=postgresql.git;f=src/include/storage/sinvaladt.h;hb=HEAD)
