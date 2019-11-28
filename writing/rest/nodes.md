---
layout: writing
---
# Postgres Clustering: Nodes

Postgres is a single-machine RDMBS, which means it is normally installed on one
machine and all the operations (transactions, queries) it performs are limited to that
one machine.  However, there are configurations in which Postgres instances running on
more than one machine together implement a system that is more capable than a single
Postgres instance.  For example:

* Streaming replication, where one of the servers assumes the role of *primary* server,
one that can process writes from the application, while others become *secondary*
servers, all of which just replay the log of writes that have been successfully
performed on the primary

* Accessing tables, views of one or more of the independently running Postgres servers
(let's call them the *downstream* servers) using `postgres_fdw` (a Foreign Data Wrapper)
from the server to which the applications connect (let's call it the *upstream* server)

* Logical replication, where one or more from the collection of independently running
Postgres servers publish the changes made to specific tables (*publishers*), while others
subscribe to those change streams (*subscribers*); a publisher can itself be a subscriber
and vice versa

Each of the above setups implement a bespoke protocol for interactions between the
inter-connected Postgres servers, as summarized below:

* In the streaming replication case, a persistent `libpq` connection is opened between
the primary server and each of the secondary servers to exchange physical replication
protocol messages.  The connection is initiated by the secondary server when an administrator
spins it up to run as a streaming replication secondary.

* In the `postgres_fdw` case, a `libpq` connection is opened from the upstream server
to the downstream server to exhange query and its result.  The connection is initiated
when an application issues a query on the upstream server that accesses tables/views on
the downstream server.  Once established by the first query, `postgres_fdw` keeps the
connection open as long as a given the application session is active.

* In the logical replication case, a persistent `libpq` connection is opened between a
publisher and a subscriber to exchange logical replication messages.  The connection is
initiated by the subscriber when an adminitrator runs `CREATE SUBSCRIPTION` command on it
to create a subscription object pointing to a publication object on the publishing server.
There is one connection for each subscription created on the subscriber.
