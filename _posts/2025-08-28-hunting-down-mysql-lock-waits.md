---
title: "🔎 Hunting Down MySQL/InnoDB Lock Waits (RDS‑friendly Runbook)"
date: 2025-08-29
description: "A practical, copy‑pasteable checklist for tracking and clearing lock waits and replication blockers in MySQL (works great on AWS RDS/Aurora). Includes queries to identify waiting vs. blocking statements, map them to sessions, and safely unblock."
categories:
  - mysql
  - rds
  - databases
  - performance
tags:
  - mysql
  - innodb
  - locking
  - replication
  - rds
  - devops
  - sysadmin
---

Lock contention usually shows up at the worst time: dashboards stuck, replicas lagging, batch jobs timing out. This runbook gives you a **battle‑tested, minimal set of queries** to quickly identify *who is waiting*, *who is blocking*, and *what to kill* — with **RDS‑safe** options.

> TL;DR: Start at **Step 1** and work downward. Each step tells you *why* you’re running it, *what to look for*, and *what to do next*.

---

## 🧭 Overview: What you’re trying to answer
1. **Which queries are waiting?** (symptoms)  
2. **Which session is blocking them?** (root cause)  
3. **Is replication also blocked?** (extra pain)  
4. **Is it safe to kill the blocker?** (minimize blast radius)

---

## Prereqs & Tips
- Use a session with `PROCESS` privilege (RDS/Aurora: standard admin user usually has this).
- Prefer **short LIMITs** and **ORDER BY “age”** in diagnostics to surface the worst issues first.
- On AWS RDS/Aurora, prefer `mysql.rds_kill_query()` / `mysql.rds_kill()` when terminating sessions.

---

## Step 1 — Get the top lock waits
**Why:** See the worst waits first and capture both the waiting and blocking sides.

```sql
SELECT 
  ilw.wait_age, 
  ilw.locked_table, 
  ilw.locked_index, 
  ilw.locked_type, 
  ilw.waiting_trx_id, 
  ilw.waiting_pid AS waiting_conn_id, 
  ilw.waiting_query, 
  ilw.blocking_trx_id, 
  ilw.blocking_pid AS blocking_conn_id, 
  ilw.blocking_query 
FROM 
  sys.innodb_lock_waits AS ilw 
ORDER BY 
  ilw.wait_age DESC 
LIMIT 
  10;
```

**Read this output as:**  
- `waiting_query` = the **blocked** statement.  
- `blocking_query` = the **lock holder** (if still active).  
- `blocking_conn_id` = the **session id** you can act on (kill).

> If `blocking_query` is NULL, the transaction may be idle but still holding locks — go to **Step 3** to find the session by user/text and age.

---

## Step 2 — Enrich with user/host (optional but recommended)
**Why:** To find the responsible app/user and machine. Works across MySQL 5.7/8.0 and RDS.

```sql
SELECT 
  ilw.wait_age, 
  ilw.locked_table, 
  ilw.locked_index, 
  ilw.locked_type, 
  ilw.waiting_trx_id, 
  ilw.waiting_pid AS waiting_conn_id, 
  tw.PROCESSLIST_USER AS waiting_user, 
  tw.PROCESSLIST_HOST AS waiting_host, 
  ilw.waiting_query, 
  ilw.blocking_trx_id, 
  ilw.blocking_pid AS blocking_conn_id, 
  tb.PROCESSLIST_USER AS blocking_user, 
  tb.PROCESSLIST_HOST AS blocking_host, 
  ilw.blocking_query 
FROM 
  sys.innodb_lock_waits AS ilw 
  LEFT JOIN performance_schema.threads AS tw ON tw.PROCESSLIST_ID = ilw.waiting_pid 
  LEFT JOIN performance_schema.threads AS tb ON tb.PROCESSLIST_ID = ilw.blocking_pid 
ORDER BY 
  ilw.wait_age DESC 
LIMIT 
  10;
```

---

## Step 3 — Hunt down suspect sessions by user or query text
**Why:** Sometimes the blocker is idle or its SQL text is gone. Search by user or a known pattern.

```sql
-- Example from a reporting job you suspected:
SELECT 
  id, 
  user, 
  host, 
  time, 
  state, 
  LEFT(info, 400) AS info 
FROM 
  information_schema.processlist 
WHERE 
  user = 'report_readonly' 
  OR info LIKE '%tmpSuspectOrderCustomers%' 
ORDER BY 
  time DESC;
```

**Look for:** long `time`, `state` like “Locked”, or `info` that matches your batch/reporting tasks.

---

## Step 4 — Inspect active transactions & age
**Why:** Find idle-in-transaction or long‑running transactions still holding locks.

```sql
SELECT 
  it.trx_id, 
  it.trx_state, 
  it.trx_started, 
  TIMESTAMPDIFF(SECOND, it.trx_started, NOW()) AS trx_age_s, 
  it.trx_wait_started, 
  it.trx_rows_locked, 
  it.trx_rows_modified, 
  it.trx_mysql_thread_id AS conn_id 
FROM 
  information_schema.innodb_trx AS it 
ORDER BY 
  it.trx_started;
```

**Tip:** Join to `processlist` to see who/where:

```sql
SELECT 
  it.trx_id, 
  it.trx_state, 
  it.trx_started, 
  TIMESTAMPDIFF(SECOND, it.trx_started, NOW()) AS trx_age_s, 
  it.trx_rows_locked, 
  it.trx_rows_modified, 
  p.id AS conn_id, 
  p.user, 
  p.host, 
  p.time, 
  p.state, 
  LEFT(p.info, 400) AS info 
FROM 
  information_schema.innodb_trx it 
  LEFT JOIN information_schema.processlist p ON p.id = it.trx_mysql_thread_id 
ORDER BY 
  it.trx_started;

```

**Red flags:** very old `trx_started`, `trx_state='ACTIVE'` with `p.state='Sleep'` (idle‑in‑transaction), or high `trx_rows_locked`.

---

## Step 5 — Check replication health and blockers
**Why:** Replication can stall if the **SQL applier** is waiting for a lock on the replica.

```sql
-- MySQL 8.0 naming
SHOW REPLICA STATUS\G;
-- MySQL 5.7 naming
SHOW SLAVE STATUS\G;
```

**What to scan:** look for `Replica_SQL_Running` / `Slave_SQL_Running = No|Yes`, and `Last_SQL_Error` / `Last_Error`. If running but lagging, the applier may be waiting on a lock.

```sql
-- Applier threads often appear as 'system user' in the processlist
SELECT 
  id, 
  user, 
  host, 
  time, 
  state, 
  LEFT(info, 400) AS info 
FROM 
  information_schema.processlist 
WHERE 
  user = 'system user' 
ORDER BY 
  time DESC;
```

> If you see the applier `state` mentioning waiting/locking, then **someone on the replica** is holding a lock that blocks replication. Use **Steps 1–4** on the **replica** to identify and clear the blocker.

---

## Step 6 — Who is locking which tables? (quick inventory)
**Why:** Surface hot tables and lock types at a glance (MySQL 8.0+).

```sql
SELECT
  dl.OBJECT_SCHEMA  AS schema_name,
  dl.OBJECT_NAME    AS object_name,
  dl.LOCK_TYPE,
  COUNT(*)          AS locks
FROM performance_schema.data_locks AS dl
WHERE dl.OBJECT_SCHEMA NOT IN ('mysql','performance_schema','sys','information_schema')
GROUP BY 1,2,3
ORDER BY locks DESC
LIMIT 20;
```

---

## Step 7 — Decide & act (safely)
**Why:** Minimize blast radius. Prefer killing the **query** first, then the **connection** only if needed.

**RDS‑safe options:**

```sql
-- Stop just the running statement, keep the session
CALL mysql.rds_kill_query(<blocking_conn_id>);

-- Terminate the session entirely
CALL mysql.rds_kill(<blocking_conn_id>);
```

**Generic MySQL (non‑RDS):**

```sql
-- Stop just the current statement
KILL QUERY <blocking_conn_id>;

-- Terminate the session
KILL CONNECTION <blocking_conn_id>;
```

> After killing, **re‑run Step 1** to confirm the backlog is shrinking.

---

## Optional — Deep dive / supporting queries

### A. Show the current InnoDB lock wait section
```sql
SHOW ENGINE INNODB STATUS\G
```

### B. See statement patterns causing heavy waits on a hot table (8.0+)
```sql
-- Adjust the LIKE to your table name/pattern
SELECT
  esb.schema_name,
  esb.digest_text,
  esb.count_star,
  ROUND(esb.sum_timer_wait/1e12, 3) AS total_seconds
FROM performance_schema.events_statements_summary_by_digest AS esb
WHERE esb.digest_text LIKE '%UPDATE% orders %'  -- example pattern
ORDER BY total_seconds DESC
LIMIT 10;
```

### C. End‑to‑end “Top Waits” one‑liner (handy for paging)
```sql
SELECT
  ilw.wait_age, ilw.locked_type, ilw.locked_table, ilw.locked_index,
  ilw.waiting_pid  AS waiting_conn_id,
  ilw.blocking_pid AS blocking_conn_id,
  LEFT(ilw.waiting_query, 200)  AS waiting_q_short,
  LEFT(ilw.blocking_query, 200) AS blocking_q_short
FROM sys.innodb_lock_waits AS ilw
ORDER BY ilw.wait_age DESC
LIMIT 10;
```

---

## Suggested order of operations (copy/paste)
1. **Top waits:** run the query in **Step 1**.  
2. **Add context:** run **Step 2** if you need user/host.  
3. **Search suspects:** run **Step 3** using your app’s user/patterns.  
4. **Check txn age:** run **Step 4**; note any idle‑in‑transaction offenders.  
5. **Check replication:** run **Step 5** on the **replica** if lagging.  
6. **Inventory locks:** run **Step 6** if the hotspot isn’t obvious.  
7. **Act safely:** apply **Step 7**. Re‑check **Step 1** afterwards.

---

## FAQ
- **Does `waiting_query` show the lock holder?** No — it shows the **blocked** query. The lock holder is in `blocking_query` (and `blocking_pid`).  
- **Why is `blocking_query` empty?** The blocker might be idle (no current statement), but its transaction still holds locks. Use Steps 3–4 to identify it.  
- **Can I always kill blockers?** Prefer to coordinate with owners; kill the **query** first. If it reappears, fix the app/transaction pattern.

---

**Keep this runbook handy** — it’s short enough to use under pressure, and it covers both application lock storms and replication stalls. If you want, I can wrap these into a stored view/package tailored to your schema names and users.
