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

The point of this exercise is that it can save CPU cycles by not redoing the
parse/analyze/rewrite processing on every request, which is fine because the result
of that processing would be the exact same internal parse tree unless some object
mentioned in the query was changed by DDL, something that tends to happen rather
unfrequently.  Even more CPU cycles are saved if the `EXECUTE` step is able to use
a plan that is also cached, instead of building it from scratch for that particular
execution.

The decision of whether or not to use a cached plan is made by the plancache module
present in the backend that is involved in the processing of the `EXECUTE` statement.
The way it does that is by keeping track of and comparing the costs of two types of
plan: 1) that obtained by binding the values of parameters provided in the `EXECUTE`
statement (that is by making them known to the planner), 2) that obtained by leaving
the query parameters unbounded (that is, by leaving the planner in the dark about what
they are).  The former is called a "custom" plan, because the planner would have
customized it to the given set of parameter values and the latter a "generic" plan,
because it is not specific to any given set of parameter values.  By definition, only
a "generic" plan can ever be deemed reusable and thus cached.
