---
layout: writing
title: "Partitioned tables can now be replicated"
---
# Partitioned tables can now be replicated

If your shop uses logical replication to selectively replicate tables between, say,
an operations cluster and an analytics cluster, you may be faced with a frustrating
situation that would have started with encountering this on the publishing server:

```
CREATE PUBLICATION "pub" FOR TABLE "foo";
ERROR:  "foo" is a partitioned table
DETAIL:  Adding partitioned tables to publications is not supported.
HINT:  You can add the table partitions individually.
```
