---
layout: writing
title: "Postgres: woes of parameterized queries with partitioning"
tags: [writing, pg]
last_updated: 2022-05-11
---
# Postgres: woes of parameterized queries with partitioning

May 11, 2022

Parameterized queries (aka prepared statements) suffer when you partition tables
mentioned in those queries.  Let's see why.

Using the parameterized queries feagure allows a client to issue a query in two
stages.  With Postgres, those two stages involve using the statement `PREPARE a_name AS <SQL>`
the first time to pass the query text to the server to be remembered by the given
name, followed by multiple invocations of the statement `EXECUTE a_name` each of
which will compute and return the query's output rows.  The backend process which
receives the `PREPARE` will parse, analyze, rewrite the query text and store the
resulting C struct holding the parse tree in a process-local hash table using the
name given in `PREPARE` as the key.  Each of the subsequent `EXECUTE` statements
sent to the same backend process will look up the query's parse tree using the name
given and execute its plan (either one created from scratch or a cached one, more on
this in a bit) to produce the query's result rows.

The point of this exercise is that it can save a lot of CPU cycles by not redoing
the parse/analyze/rewrite processing on every request, which is fine because the
result of that processing is the exact same internal parse tree unless some object
mentioned in the query was changed by DDL which happens much less frequently.  Even
more CPU cycles are saved if the `EXECUTE` step is able to use a plan that is also
cached (that is, along with the parse tree).
