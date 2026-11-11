---
title: "Investigating a Deadlock During Batch Inserts in PostgreSQL"
date: 2026-11-11 23:00:00 +0900
categories: [Backend, DB]
tags: [PostgreSQL, Deadlock, Batch Insert]
description: "A deadlock during a PostgreSQL 17 load test and what I found while investigating it"
math: true
mermaid: true
---

I had mainly associated deadlocks with `UPDATE` statements. Then I saw one during a load test that inserted data into PostgreSQL 17.

## Background

My application inserted records in batches. The table had a primary key made up of three columns. Several server instances were writing to it at the same time.

Here is a simplified version of the two statements. The table name is a placeholder; the primary key is `(id, host_name, created_at)`.

```sql
-- Server 1
INSERT INTO table_name (id, host_name, created_at) VALUES
  (1, '01', TIMESTAMP '2026-11-11 10:00:00'),
  (2, '02', TIMESTAMP '2026-11-11 10:00:00');

-- Server 2
INSERT INTO table_name (id, host_name, created_at) VALUES
  (2, '02', TIMESTAMP '2026-11-11 10:00:00'),
  (1, '01', TIMESTAMP '2026-11-11 10:00:00');
```

## What I investigated

At first, I did not understand why an `INSERT` could deadlock, so I asked an AI assistant. In this example, each server sends one `INSERT` statement with multiple rows. I had assumed that using one statement meant the batch could not get stuck partway through. But PostgreSQL still checks each proposed row against the unique index.

I asked the AI assistant to help me test this locally and explain the results using the metrics. The local test reproduced a deadlock. I also checked the PostgreSQL documentation it cited.

PostgreSQL 17 explains what happens when a unique key conflicts with a row inserted by another transaction that has not committed yet:

> “the would-be inserter must wait to see if that transaction commits.”

— [PostgreSQL 17, Section 62.5, “Index Uniqueness Checks”](https://www.postgresql.org/docs/17/index-unique-checks.html)

An `INSERT` can therefore wait for another transaction even though it does not update an existing row.

A wait becomes a deadlock when two transactions wait for each other. Suppose the rows are processed in the order shown: Server 1 inserts the key `(1, '01', 2026-11-11 10:00:00)`, and Server 2 inserts `(2, '02', 2026-11-11 10:00:00)`. Server 1 then waits for Server 2's key, while Server 2 waits for Server 1's key. This is the cycle described in PostgreSQL's [deadlock documentation](https://www.postgresql.org/docs/17/explicit-locking.html#LOCKING-DEADLOCKS). I reproduced a deadlock locally, but I have not confirmed that the load test followed this exact sequence.

<!-- Add the deadlock log and insert order here if available. -->

## Possible solutions

- **Sort each batch by the primary-key columns.** If both servers submit the rows in the same key order, it could prevent the cycle shown above. It would not prevent a duplicate-key error. Sorting has a cost, though it may be small for our batch size. I have not measured it yet.
- **Check whether the primary key is necessary.** The table stores log-like data, so the key may not be needed. Removing it would also remove the uniqueness guarantee, so this needs a separate decision.

## What I learned

I used to trust AI explanations too quickly. I sometimes tried a suggestion before checking the load-test results or metrics myself. After talking with other engineers, I have started to value that evidence more.

I want to get better at reading those metrics. I cannot access all of them myself, so I need to ask infrastructure engineers and DBAs for help and work through the problem with them.
