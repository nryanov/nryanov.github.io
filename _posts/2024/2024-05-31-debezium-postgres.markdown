---
layout: single
title: "PostgreSQL: Log-based CDC using debezium"
date: 2024-05-10 04:30:00 +0300
categories: postgres replication cdc debezium logical-replication wal
---

Before diving into details about streaming I want to start from the task which we will try to solve during this article.
Imagine that we have some table which represents user's balance:
```sql
CREATE TABLE user_balances
(
    user_id     BIGINT NOT NULL,
    balance     BIGINT NOT NULL,
    last_update DATE   NOT NULL,
    CONSTRAINT user_balances_pk PRIMARY KEY (user_id)
);
```

Now we want to handle each change of user's balance and according to actual balance notify or not the target user by its id.
How we can achieve this? This is the main question which we will consider and try to find the answer during this article.

# Table of contents
1. [CDC: Change Data Capture](#cdc)
2. [How Postgres handle data changes?](#how-postgres-handle-data-changes)
3. [Logical replication and WAL](#logical-replication-and-wal)
4. [Debezium](#debezium)
5. [Real usages of CDC](#cdc-usages)
6. [Conclusion](#conclusion)

# CDC: Change Data Capture <a name="cdc"></a>
In the Internet the `CDC` is described as a design pattern which allows to track data changes (deltas).
Let's consider this approach on table `user_balances`. Initial state of table is:

```sql
| user_id | balance | last_updated | 
| 1       | 100     | 2024-01-01   | 
| 2       | 100     | 2024-01-01   | 
| 3       | 100     | 2024-01-01   | 
```

We have three users and all of them have the same balances. After some time the table content has changed to:
```sql
| user_id | balance | last_updated | 
| 1       | 200     | 2024-01-02   | 
| 2       | 150     | 2024-01-03   | 
| 3       | 50      | 2024-01-04   | 
| 4       | 500     | 2024-01-05   | 
```

Each user were updated and also the new one was added. The question is `how to track these changes`? 
Imagine that you develop some kind of audit application, and you have to track each change of this table.
Your goal will be to get something like that:
```sql
| __operation__ | __timetamp__  | user_id | balance | last_updated | 
| UPDATE        | 2024-01-02    | 1       | 200     | 2024-01-02   |
| UPDATE        | 2024-01-03    | 2       | 150     | 2024-01-03   |
| UPDATE        | 2024-01-04    | 3       | 50      | 2024-01-04   |
| CREATE        | 2024-01-05    | 4       | 500     | 2024-01-05   | 
```

And `CDC` can help you with it. Let's move on and consider different ways to achieve this.

# How Postgres handle data changes? <a name="how-postgres-handle-data-changes"></a>
