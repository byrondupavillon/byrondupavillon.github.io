---
title: "🕒 ElastiCache Redis Service Update: Real-World Timeline & What Each Event Means"
date: 2025-09-15
description: "A minute-by-minute breakdown of an ElastiCache for Redis service update (cluster mode disabled) with primary + replica. Includes the exact sequence, expected downtime window, and what to monitor."
categories:
  - aws
  - redis
  - elasticache
  - maintenance
  - high-availability
tags:
  - aws
  - elasticache
  - redis
  - maintenance
  - failover
  - sres
  - devops
---

This post documents a **live service update** on an ElastiCache for Redis replication group with **cluster mode disabled** (single shard, one primary + one replica).

All timestamps below are **Europe/London (UTC+01:00)**.

> **TL;DR:** The full maintenance window ran **23m16s** end-to-end, with a **brief failover blip** at 21:16:15 (typically seconds). Everything else was non-disruptive.

---

## 🧭 Topology & Update Pattern

- **Topology:** 1 shard → **Primary** (`node-primary`) + **Replica** (`node-replica`)
- **Update method (rolling):** 
  - 1) Patch replica  
  - 2) **Failover** (replica promoted to primary) ← *brief interruption here*  
  - 3) Patch the old primary (now replica)  
- **Blast radius:** With a healthy replica and automatic failover, the only noticeable impact is a short reconnect window during failover.

---

## ⏱️ Timeline at a Glance

```
21:03:37   Replica begins patch ("Recovering")
      └────────── 9m53s ──────────┘
21:13:30   Replica patched ("Finished recovery") – ready to be promoted
      └── 2m45s ──┘
21:16:15   FAILOVER COMPLETED  ← brief client interruption here (seconds)
            └─ 39s ─┘
21:16:54   Old primary (now replica) begins patch ("Recovering")
      └────────── 9m59s ──────────┘
21:26:53   Old primary patched ("Finished recovery") – replication healthy
```

**Durations (precise):**
- Replica patch: **9m53s** (21:03:37 → 21:13:30)  
- Prep → Failover: **2m45s** (21:13:30 → 21:16:15)  
- Failover → next patch start: **39s** (21:16:15 → 21:16:54)  
- Old primary patch: **9m59s** (21:16:54 → 21:26:53)  
- **Total window:** **23m16s** (21:03:37 → 21:26:53)

---

## 📜 Event-by-Event: What’s Actually Happening

> I’ve generalized the event text and node names to keep this reusable.

1) **21:03:37 — `node-replica`: “Recovering cache nodes 0001”**  
   - AWS takes the **replica** out of rotation to apply the service update.  
   - **Impact:** None to traffic. Primary continues serving reads/writes.

2) **21:13:30 — `node-replica`: “Finished recovery for cache nodes 0001”**  
   - Replica is patched, healthy, and in sync. It’s ready to be promoted.  
   - **Impact:** None (still running on the original primary).

3) **21:16:15 — Replication group + `node-primary`: “Failover … completed”**  
   - AWS promotes the updated replica to **new primary**.  
   - **This is the only interruption window**: clients briefly reconnect to the new primary endpoint.  
   - **Observed/expected:** seconds; typically visible as a short spike in connection errors/latency.

4) **21:16:54 — `node-primary` (now the replica): “Recovering cache nodes 0001”**  
   - The old primary, now acting as the replica, is patched.  
   - **Impact:** None (the new primary keeps serving traffic).

5) **21:26:53 — `node-primary` (replica): “Finished recovery for cache nodes 0001”**  
   - The second node is patched and replication is healthy again.  
   - **Maintenance complete.**

> You may see **duplicate “Failover completed”** entries at both the **replication group** and **cache-cluster** levels. That’s normal.

---

## 🧪 What to Monitor (and What “Good” Looks Like)

- **During failover (≈ seconds):**  
  - *Clients:* brief `ECONNRESET`/`READONLY`/timeout blips; auto-retries should succeed.  
  - *Endpoints:* If you use the **primary endpoint**, your client should reconnect automatically.

- **CloudWatch metrics (per node and/or replication group):**  
  - `CurrConnections` — small dip/spike at failover, then normalizes.  
  - `ReplicationLag` — near zero after each node rejoins (may momentarily increase as the second node catches up post-patch).  
  - `EngineCPUUtilization` / `CPUUtilization` — brief transient changes are fine; look for quick stabilization.  
  - `NetworkBytesIn/Out` — returns to baseline post-failover.

- **Quick Redis checks (from an app bastion):**
  - `INFO REPLICATION` — confirm `role:master` on the new primary; the peer shows as `slave`/`replica` and `state=online`.  
  - `ROLE` — fast sanity check for master/replica roles.  
  - A simple **SET/GET** smoke test against the primary endpoint.

---

## 🧯 Downtime, Risk, and Expectations

- With **cluster mode disabled** but a **healthy replica** and **automatic failover**:
  - **User-visible impact:** a short reconnect window at the failover timestamp (here: **21:16:15**).  
  - **Everything else:** non-disruptive patching while the other node carries traffic.
- If you had **no replica**, the node would be restarted in place → **full downtime** during patch.

---

## ✅ Post-Maintenance Health Checklist (copy/paste)

1. **Endpoints & roles**
   - Connect to the **primary endpoint**, run `ROLE` or `INFO REPLICATION` → verify `master` role.
2. **Replication**
   - On the replica: `INFO REPLICATION` → `master_link_status:up`, `master_sync_in_progress:0`, `slave_repl_offset` close to master.
   - CloudWatch `ReplicationLag` trending to near-zero.
3. **Traffic**
   - App error rates/latency back to baseline within minutes of failover.
   - No sustained `READONLY` or timeout errors in logs.
4. **Capacity**
   - `CPUUtilization`, `FreeableMemory`, `Evictions`, `BytesUsedForCache` within normal bounds.
5. **Events**
   - Last events show both nodes “Finished recovery …” with no subsequent warnings.

---

## 🧠 Takeaways & Benchmarks

- **Total maintenance window:** **~23 minutes** is a reasonable end-to-end expectation for a 2-node shard.  
- **User-visible impact:** **seconds**, concentrated at the **failover** moment.  
- **Consistency:** Both node patch phases were ~10 minutes each, with <3 minutes prep between first recovery and failover.

If your window is dramatically longer or you see lingering replication lag, dig into:
- Suboptimal client retry/backoff settings  
- Network path issues (look for elevated packet loss)  
- A replica catching up a large write burst right after failover

---

## 📎 Reuse This Pattern

You can treat this as a template any time you apply **service updates** or do **planned failovers** on ElastiCache for Redis with **cluster mode disabled**:
- Patch replica → **Failover** → Patch old primary → Verify health.

For **cluster mode enabled**, the pattern is the same but **per-shard**. Only the shard being updated experiences the brief blip; others continue serving normally.

---
