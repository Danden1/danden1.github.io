---
title: "Investigating PostgreSQL Memory Growth with R2DBC"
description: "How disabling the prepared statement cache changed our memory pattern, and what I learned about statement reuse in PostgreSQL."
date: 2026-03-22 23:00:00 +0900
categories: [Backend, Spring]
tags: [Backend, Spring, Webflux, PostgreSQL, Prepared Statement]
---

Our service collected and processed data every minute, then inserted it into PostgreSQL in batches. We used Spring WebFlux and r2dbc-postgresql on two application servers, without custom connection lifetime settings.

The total memory usage shown by our database monitoring repeatedly increased during the day. After I set `preparedStatementCacheQueries=0`, it stayed around 60% instead of following the previous pattern.

That change solved the symptom we were seeing. Understanding what it changed took more work. This post describes the result and the cache behavior that could explain it. I did not measure prepared-statement memory separately, so I cannot attribute the entire increase to it.

## The INSERTs We Were Sending

The tables were partitioned by date, and our SQL directly named the date-suffixed table. Here is a simplified example; the columns and date are illustrative:

```sql
INSERT INTO logs_20260318 (id, message) VALUES
  ($1, $2),
  ($3, $4),
  ($5, $6);
```

The total amount of data collected each time was roughly similar, but the number of rows in an individual batch could vary. A 999-row batch and a 1,000-row batch had different SQL strings because they contained different numbers of placeholders.

The table name also changed with the date. Both details matter when the driver decides whether it can reuse a statement.

## What Happens to a Prepared Statement?

Consider a smaller INSERT:

```sql
INSERT INTO logs_20260318 (id, message) VALUES ($1, $2);
```

The SQL describes the operation. The values for `$1` and `$2` are sent separately, so a message is treated as data rather than SQL syntax.

PostgreSQL's extended query protocol separates the work into three main steps:

1. **Parse:** Send the SQL, parameter type information, and a statement name. PostgreSQL parses and analyzes the query.
2. **Bind:** Refer to that statement and supply the values. PostgreSQL prepares an execution instance, called a portal. Planning usually happens here.
3. **Execute:** Run that portal.

These are protocol messages. The application does not need to send SQL `PREPARE` and `EXECUTE` commands itself. This is a simplified flow; the protocol also includes synchronization and other messages. See PostgreSQL's [extended query documentation](https://www.postgresql.org/docs/current/protocol-flow.html#PROTOCOL-FLOW-EXT-QUERY).

An execution plan describes how PostgreSQL will carry out the SQL. For a SELECT, that might include choosing an index scan or a table scan. An INSERT also needs a plan to feed rows into the insertion operation.

A prepared statement does not always reuse one fixed plan. PostgreSQL can make a **custom plan** using the current parameter values, or reuse a **generic plan** that is independent of those values. Preparing the statement saves the earlier parsing and analysis work even when PostgreSQL makes a new custom plan. The [PREPARE documentation](https://www.postgresql.org/docs/current/sql-prepare.html) explains this distinction.

## What Is Cached, and Where?

There are two places to consider.

In the application, the driver remembers which statement name belongs to a SQL string and its parameter types. The current [unlimited cache implementation](https://github.com/pgjdbc/r2dbc-postgresql/blob/main/src/main/java/io/r2dbc/postgresql/IndefiniteStatementCache.java) generates names such as `S_0` and `S_1`.

In PostgreSQL, the backend serving that connection stores named statements in a hash table keyed by those names. Each entry refers to cached query information, including the SQL text, parsed and analyzed representations, parameter types, and dependencies. A reusable generic execution plan may also be retained. These are server-side objects, separate from the driver's map. PostgreSQL's [statement storage](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/prepare.c) and [plan cache](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/utils/cache/plancache.c) implementations show this arrangement.

The following example shows the relationship for one connection:

```text
Application driver                         PostgreSQL session
SQL + parameter types -> S_0                S_0 -> saved query information
```

On reuse, the driver can refer to `S_0` and send new values without sending another Parse for the same statement. PostgreSQL looks up the name in that session.

The name is only an identifier. It does not represent a table or an individual batch of data. With caching enabled, the following rules apply within the same connection:

| What changes? | Can the driver reuse the entry? |
| --- | --- |
| Only the bound values | Yes, if the SQL and parameter types match |
| The number of rows in the INSERT | No: the SQL has different placeholders |
| The date in the table name | No: the SQL text changes |
| The parameter types | A separate entry is needed |

This means similar data volumes can still produce many cached SQL forms.

Prepared statements belong to a session; they are not shared across PostgreSQL connections. A second physical connection needs its own statements, even if it executes identical SQL. The relevant count is therefore the entries held by each connection, added together—not the number of pool objects. I do not have the historical connection or statement counts for this incident.

## The Option I Changed

The r2dbc-postgresql [configuration documentation](https://github.com/pgjdbc/r2dbc-postgresql#programmatic-configuration) describes `preparedStatementCacheQueries` as a per-connection setting:

| Value | Behavior |
| --- | --- |
| `-1` | Unlimited cache; the documented default |
| `0` | Disable the statement cache |
| A positive number | Limit the number of cached statements |

I changed it to:

```text
preparedStatementCacheQueries=0
```

The driver source explains why this matters. Its [disabled cache implementation](https://github.com/pgjdbc/r2dbc-postgresql/blob/main/src/main/java/io/r2dbc/postgresql/DisabledStatementCache.java) returns an empty statement name, always requires preparation, and stores no cache entry.

An empty name selects PostgreSQL's unnamed statement. A new unnamed Parse replaces the previous unnamed statement. Parameter binding still works; disabling this cache does not mean concatenating values into SQL. Named statements, by contrast, survive until explicitly removed or the session ends. These lifetimes are defined in the protocol documentation linked above.

For our changing INSERT strings, this removes the driver's retention of a separate named statement for each SQL-and-type combination. It also gives up reuse of preparation work across those executions. I did not record a controlled CPU or latency comparison, so I cannot say what that trade-off cost us.

The implementation references here describe the sources reviewed for this write-up. I have not matched them to the exact driver version used during the incident.

## What the Memory Graph Does Not Explain

Disabling the cache stopped the repeated upward pattern. Together with the changing SQL strings, that makes statement retention a plausible explanation for the growth. It does not tell me how much memory each statement used.

There was also a daily drop in memory. From what I remember, it happened around midnight, while old tables were dropped at about 2 a.m. I cannot explain the earlier drop with the later table deletion. The baseline did not keep rising from day to day.

I initially thought dropping a table would delete its prepared statements. That was too simple. PostgreSQL invalidates cached query information affected by schema changes so it can be checked again before reuse. Invalidation does not itself remove the named statement. If its SQL still refers to a table that no longer exists, reuse can fail during reanalysis. The plan-cache implementation linked above distinguishes invalidation from removal.

I no longer have the session-level records needed to explain the midnight decrease. If I investigated this again, I would capture statement counts and backend memory alongside the overall graph. PostgreSQL's [`pg_prepared_statements`](https://www.postgresql.org/docs/current/view-pg-prepared-statements.html) view can show statements in the current session, but querying it from a separate admin connection does not show the application's statements.

## Why I Opened a GitHub Issue

I opened [r2dbc-postgresql issue #702](https://github.com/pgjdbc/r2dbc-postgresql/issues/702) to ask whether disabling statement caching by default would be safer. I described the memory behavior we had seen and offered to help with a change if the proposal was accepted.

The issue records my question about the default. It is not confirmation of a driver bug or of the cause of this incident. My original argument also quoted HikariCP's general discussion of statement caching. PostgreSQL's session-local behavior needs to be considered separately; that quotation should not be read as evidence that PostgreSQL shares these plans across connections.

For this workload, disabling the cache was an effective fix for the observed memory pattern. The part I understand better now is what makes statements reusable: sending roughly the same amount of data is not enough. The SQL text, parameter types, and connection determine whether the driver can use an existing entry.
