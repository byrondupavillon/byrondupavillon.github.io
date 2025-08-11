---
title: "🧪 Verify & Repair MySQL Replication Drift with Percona Toolkit (pt-table-checksum + pt-table-sync)"
date: 2025-08-11
description: "A gentle, production-safe workflow to detect and fix data drift between a MySQL primary and replicas using Percona Toolkit—without drowning your replica."
categories:
  - databases
  - mysql
  - reliability
tags:
  - percona
  - replication
  - pt-table-checksum
  - pt-table-sync
  - devops
---

Replication drift happens—rows diverge quietly, and you only notice when a report looks “off.” Here’s a practical, throttled approach to **detect** and **repair** drift with Percona Toolkit.

You’ll:

1) Set up the DSN table once  
2) Run a gentle checksum on the **primary**  
3) Inspect differences on the **replica**  
4) Generate and (optionally) execute safe fixes

---

## 🧾 1) Prerequisites

- **Percona Toolkit** installed on a host that can reach your MySQL servers
- Access to the **primary** and at least one **replica**
- Account with sufficient privileges (read on primary; write to `percona.checksums`; ability to connect to the replica for sync)

Install:

```bash
apt-get install percona-toolkit
```

Create a place for checksum metadata and DSN discovery on the **primary**:

```sql
CREATE DATABASE IF NOT EXISTS percona;

CREATE TABLE IF NOT EXISTS percona.dsntable (
  id  INT AUTO_INCREMENT PRIMARY KEY,
  dsn VARCHAR(255) NOT NULL
);
```

Register your replica connection (generic example):

```sql
INSERT INTO percona.dsntable (dsn)
VALUES ('h=replica_ip,u=root,p=password,P=3306');
-- Replace host/user/password/port appropriately
```

---

## 🔍 2) Run a throttled checksum (on the primary)

This command has proven nicely gentle in practice—adaptive chunking, quick backoff on lag, and basic load guards:

```bash
pt-table-checksum \
--replicate=percona.checksums \
--create-replicate-table \
--empty-replicate-table \
--recursion-method=dsn=t=percona.dsntable \
--no-check-binlog-format \
--chunk-time=0.5 \
--chunk-size-limit=2.0 \
--max-lag=1 \
--check-interval=10 \
--max-load='Threads_running=25' \
--ignore-databases mysql,percona \
-h primary_ip -P 3306 -u root -p'password'
```

### Scope it (optional)

Only include specific databases/tables:

```bash
--databases=db_name 
--tables=table_name
```

Exclude specific databases/tables:

```bash
--ignore-databases=db_name
--ignore-tables=table_name
```

> 💡 Tip: You can mix and match the include/exclude flags to keep the run focused and fast.

---

## 🧭 3) Inspect drift on the replica

Query the **replica** to find chunks that differ from the primary:

```sql
SELECT db, tbl, chunk, chunk_index, lower_boundary, upper_boundary,
       this_crc, this_cnt, master_crc, master_cnt, ts
FROM percona.checksums
WHERE (this_crc <> master_crc OR this_cnt <> master_cnt)
  AND master_crc IS NOT NULL
  AND master_cnt IS NOT NULL;
```

- Any rows returned indicate a mismatch for that chunk.
- Use `db`, `tbl`, and boundaries to understand **where** the difference lies.

---

## 🛠️ 4) Generate safe fixes with `pt-table-sync`

Start with a **dry run** to review SQL:

```bash
pt-table-sync \
--print \
--replicate=percona.checksums \
--sync-to-master h=replica_ip,u=root,p='password' \
--ignore-databases=mysql,percona
```

Scope to one database/table if you prefer:

```bash
--databases=db_name
--tables=table_name
```

**Example (scoped):**

```bash
pt-table-sync \
--print \
--replicate=percona.checksums \
--sync-to-master h=replica_ip,u=root,p='password' \
--ignore-databases=mysql,percona \
--databases=db_name \
--tables=table_name
```

When you’re satisfied with the printed SQL, you can let the tool execute:

> ⚠️ **Caution:** Executing changes is permanent and can delete rows that exist only on the replica. Always review a `--print` run first and consider a backup/snapshot.

```bash
pt-table-sync \
--sync-to-master h=replica_ip,u=root,p='password' \
--databases=db_name \
--tables=table_name \
--execute \
--no-foreign-key-checks
```

---

## ⚖️ 5) Why this checksum run is “gentle”

- **Adaptive chunking** (`--chunk-time=0.5`): targets ~0.5s per chunk to avoid big bursts  
- **Replica-aware backoff** (`--max-lag=1`, `--check-interval=10`): pauses when lag exceeds 1s  
- **Load guard** (`--max-load='Threads_running=25'`): pauses when the primary is busy  
- **System schemas excluded** (`--ignore-databases=mysql,percona`)  

If you still see lag, try:

- Lowering `--chunk-time` (e.g., `0.3`) or switching to a fixed `--chunk-size=500`
- Running during off-peak hours
- Ensuring chunking uses a selective index (ideally the PRIMARY KEY)
- Verifying replica I/O and enabling parallel replication workers where appropriate

---

## 🧩 Common gotchas

- **Replication filters**: If replicas ignore some databases/tables, scope your checksum/sync accordingly.  
- **Binlog format**: The command skips binlog-format checks (`--no-check-binlog-format`). Remove that flag if you prefer the tool to validate consistency.  
- **Replica-only data**: If replicas intentionally have extra rows, **don’t** run a wide-open `--execute`. Always scope to known-drifting objects.  
- **Foreign keys**: `--no-foreign-key-checks` can be necessary, but understand the implications in your schema.

---

## ✅ TL;DR

1) Install Percona Toolkit; create `percona` DB + `dsntable` on the primary  
2) Run the throttled `pt-table-checksum` above on the **primary**  
3) Query `percona.checksums` on the **replica** for differences  
4) Use `pt-table-sync --print` to preview fixes, then (carefully) `--execute` if you’re sure

This workflow keeps production happy while giving you a reliable, repeatable way to detect and repair data drift.
