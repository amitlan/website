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

...
