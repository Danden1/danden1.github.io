---
title: "R2DBC Prepared Statement Issue"
description: "Exploring why R2DBC-PostgreSQL defaults to an unlimited statement cache and how it differs from the JDBC ecosystem."
date: 2026-03-22 23:00:00 +0900
categories: [Backend, Spring]
tags: [Backend, Spring, Webflux, PostgreSQL, Prepared Statement]
---

**Prepared statements** are standard practice in backend development for two main reasons:
1. **Query Efficiency**: Reusing execution plans.
2. **Security**: Preventing SQL Injection.

I also use them at work. Recently, however, I ran into an issue where **PostgreSQL memory usage kept increasing.**

### My Environment
* Spring WebFlux
* R2DBC (r2dbc-postgresql)
* PostgreSQL

---

## What Is a Prepared Statement?

A prepared statement is a pre-compiled SQL query that separates the SQL structure from the data values. Instead of concatenating values into a raw string, placeholders are used for parameters, which are bound at execution time.

**Without a Prepared Statement (Vulnerable):**
```sql
SELECT * FROM users WHERE id = 123 AND name = 'John';


-- PostgreSQL style
PREPARE get_user(INTEGER, VARCHAR) AS
  SELECT * FROM users WHERE id = $1 AND name = $2;

EXECUTE get_user(123, 'John');

```


Key Benefits:

- SQL Injection Prevention: Parameters are safely escaped by the driver.

- Query Caching: The database caches the execution plan for reuse.

- Type Safety: Parameters have strictly defined types.


### The Problem: Dynamic Batch Inserts

In my case, I was using prepared statements for bulk log inserts. The query looked like this:

```sql
INSERT INTO logs (user_id, level, message, created_at) VALUES
  ($1, $2, $3, $4),
  ($5, $6, $7, $8),
  ($9, $10, $11, $12),
  ...
```


The problem was that the number of logs varied from batch to batch. Each variation, such as a batch of 10 rows versus a batch of 20, generated a unique SQL string. As a result, PostgreSQL created and cached a different prepared statement for each batch size.

This caused PostgreSQL memory usage to increase steadily. The memory was released only when the log table was dropped or the session ended.



## R2DBC and Driver-Level Caching
In a typical R2DBC setup, the max-lifetime of the pool is often set to 0 (infinite) by default. While idle timeouts exist, they rarely trigger under heavy traffic.

Specifically, the r2dbc-postgresql driver defaults to an unlimited prepared statement cache (-1). In high-traffic services, these caches are never cleared because they are tied to the session.

Interestingly, HikariCP (the common JDBC pool) doesn't support statement caching directly—it leaves that to the driver. However, the standard PostgreSQL JDBC driver (pgjdbc) implements a default limit of 256 queries or 5 MB.

I wanted to understand why R2DBC was designed without such safety limits. Applications can avoid this problem with careful design, but a safe default would help prevent production issues. I opened an issue in the official repository and am currently awaiting a response:
[GitHub issue](https://github.com/pgjdbc/r2dbc-postgresql/issues/702)



## Conclusion
I ran into this memory issue because of an implementation detail in our bulk inserts. Reading the official documentation more carefully could have helped me avoid it, but I still believe frameworks should provide safeguards for common pitfalls like this.

This taught me a valuable lesson: don't just accept documentation at face value. It is important to ask, "Why was it designed this way?"
