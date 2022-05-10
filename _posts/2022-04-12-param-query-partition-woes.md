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
which will compute and return the query's output rows.
