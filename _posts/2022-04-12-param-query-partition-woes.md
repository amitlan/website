---
layout: writing
title: "Postgres: Woes of Parameterized Queries With Partitioning"
tags: [writing, pg]
last_updated: 2022-04-13
---
# Postgres: Woes of Parameterized Queries With Partitioning

April 13, 2022

Parameterized queries (aka prepared statements) suffer when you partition tables
mentioned in those queries.  Let's see why.

Using the parameterized queries feagture allows a client to issue a query in two
stages.  With Postgres, those two stages involve using the statement `PREPARE a_name AS <SQL>`
the first time to pass the query text to the server to be remembered by the given
name, followed by multiple invocations of the statement `EXECUTE a_name` each of
which will compute and return the query's output rows.  The backend process which
receives the `PREPARE` will parse, analyze, rewrite the query text and store the
resulting C struct holding the parse tree in a process-local hash table using the
name given in `PREPARE` as the key.  Each of the subsequent `EXECUTE` statements
sent to the same backend process will look up the parse tree using the name and
execute its plan (either one created from scratch or a cached one, more on this
in a bit) to produce the query's result rows.
