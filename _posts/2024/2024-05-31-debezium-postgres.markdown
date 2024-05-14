---
layout: single
title: "PostgreSQL: how to stream data changes to kafka?"
date: 2024-05-10 04:30:00 +0300
categories: kafka-connect postgres replication cdc debezium logical-replication wal kafka
---

Before diving into details about streaming I want to start from the task which we will try to solve during this article.
Imagine that we have some table which represents user's balance:
```plantuml
entity user_balances {
  * user_id
  --
  * balance
  additional_columns
}
```

Now we want to handle each change of user's balance and according to actual balance notify or not the target user by its id.
How we can achieve this? This is the main question which we will consider and try to find the answer during this article.

# Table of contents
1. [Data changes: what is it?](#data-changes)
2. [CDC: Change Data Capture](#cdc)
3. [Postgres: NOTIFY](#postgres-notify)
4. [Postgres: outbox pattern](#postgres-outbox)
5. [Step back: how Postgres handle data changes?](#how-postgres-handle-data-changes)
6. [Logical replication and WAL](#logical-replication-and-wal)
7. [Debezium](#debezium)
8. [Real usages of CDC](#cdc-usages)
9. [Conclusion](#conclusion)