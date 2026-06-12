Yes, both options impact the PostgreSQL master node, but they do so in different ways and through different mechanisms. [1]  
Here is a breakdown of how a replica versus a Change Data Capture (CDC) pipeline affects your master node's performance. 
Streaming Replication Impact 
PostgreSQL replication (typically physical streaming replication) has a low to moderate impact on the master. 

• Sequential Disk I/O: The master writes Write-Ahead Logs (WAL) sequentially anyway. Sending these pre-existing blocks to a replica requires very little extra CPU or disk overhead. 
• Network Bandwidth: The master must stream the WAL data over the network. High write volumes can saturate the master's network interface card (NIC). 
• WAL Retention: If the replica falls behind or disconnects, the master must retain WAL files on its disk. This can lead to disk space exhaustion if not monitored. 
• Synchronous Replication Penalty: If you use synchronous replication, the master must wait for the replica to confirm receipt of data before committing. This directly increases write latency on the master. [9, 10, 11, 12]  

CDC Pipeline Impact 
CDC pipelines (like Debezium or AWS Database Migration Service) rely on PostgreSQL Logical Decoding. This has a moderate to high impact on the master. 

• High CPU Usage: The master node must actively read the binary WAL data and decode it into a readable format (like JSON or Avro). This logical translation is CPU-intensive. 
• Memory Pressure: PostgreSQL allocates  to assemble transactions in memory before sending them to the CDC tool. If transactions are large, they spill to disk, causing heavy local I/O. 
• Connection Overhead: The CDC agent maintains a persistent connection to the master node, consuming slot resources. 
• The Replication Slot Trap: CDC tools use logical replication slots. If the CDC pipeline pauses or fails, the master node stops deleting WAL files. The master will keep saving WALs until the disk fills up completely and crashes the database. [20, 21, 22, 23]  

Summary Comparison 

| Impact Metric [24, 25, 26, 27, 28] | Streaming Replica | CDC Pipeline (Logical)  |
| --- | --- | --- |
| CPU Overhead | Very Low | Moderate to High (due to decoding)  |
| Memory Overhead | Minimal | Moderate (transaction reassembly)  |
| Disk I/O | Low (Sequential reads) | High if memory spills to disk  |
| Risk of Master Crash | Low (unless using replication slots) | High (if the CDC pipeline stalls)  |

Hybrid Best Practice 
To completely eliminate CDC overhead from your primary database, you can connect the CDC pipeline to a physical replica instead of the master. This offloads the heavy logical decoding CPU and memory tax to the replica, keeping your master node highly performant. [29]  


## Does a Read Replica Generate WAL?

### Short Answer: **Partially — and in a limited way**

A read replica **receives and applies WAL** from the primary. It does **generate some WAL locally**, but with critical restrictions:

```
Primary          →      WAL Stream       →      Read Replica
(generates WAL)      (physical/binary)        (applies WAL)
                                               ↓
                                         Generates WAL only for:
                                         - Hint bits
                                         - Temporary tables
                                         - Minimal recovery tracking
```

### What WAL a Replica Generates vs. Does NOT Generate

| WAL Type | Generated on Replica? | Reason |
|---|---|---|
| **Heap/index changes from primary** | ❌ No | Applied directly from primary WAL stream |
| **Hint bit writes** | ✅ Yes (minimal) | Visibility tracking, not streamed |
| **Temp table operations** | ✅ Yes | Temp tables are session-local |
| **Logical decoding WAL** | ⚠️ Conditional | Only if `wal_level=logical` on primary |
| **Replication slot WAL** | ⚠️ Conditional | Only with cascading replication enabled |

> **Key point**: A replica does **not** independently generate WAL for data changes — it only replays what the primary sends. Its `wal_level` is inherited from the primary.

-------------------------------------
## CDC vs Replica in PostgreSQL: Comparison & Performance Impact

### What Each Approach Does

**CDC (Change Data Capture)**
Captures row-level changes (INSERT/UPDATE/DELETE) from the WAL (Write-Ahead Log) and streams them to a target (Kafka, data warehouse, another DB, etc.).

**Read Replica**
A standby PostgreSQL instance that continuously replays WAL to maintain an identical copy of the primary, usable for read queries.

---

## Head-to-Head Comparison

| Dimension | CDC | Read Replica |
|---|---|---|
| **Primary Purpose** | Event streaming, ETL, audit logs, data sync | Offload read queries, HA/failover |
| **Data Format** | Change events (before/after row images) | Full relational copy |
| **Latency** | Near real-time (ms to seconds) | Near real-time (ms to seconds) |
| **Query Capability** | No — consumers process events | Yes — full SQL queries |
| **Schema Flexibility** | Can transform/filter on the way out | Must mirror primary schema |
| **Target** | Kafka, S3, DWH, another DB | PostgreSQL only |
| **Failover** | ❌ Not a failover candidate | ✅ Can be promoted to primary |
| **Historical replay** | ✅ Replay from WAL retention window | ❌ Point-in-time only |
| **Operational complexity** | Higher (connector, pipeline infra) | Lower (built-in streaming replication) |

---

## Performance Impact on the Primary Node

### CDC Impact

CDC uses **logical replication** (`wal_level = logical`), which has a **higher overhead** than physical replication:

```
Physical replication WAL overhead:  ~baseline
Logical replication WAL overhead:   +10–25% WAL volume
```

**Sources of overhead:**

| Factor | Detail |
|---|---|
| **WAL amplification** | Logical decoding writes more verbose WAL (full row images needed) — especially with `REPLICA IDENTITY FULL` |
| **WAL sender process** | A `walsender` process runs on primary per CDC consumer/slot |
| **Replication slot retention** | If consumer lags, WAL accumulates — **risk of disk exhaustion** |
| **CPU for decoding** | Logical decoding happens on the primary (in older setups) or on a standby (pg_logical on replica) |
| **`REPLICA IDENTITY`** | `FULL` mode (needed for UPDATE/DELETE on tables without PK) writes entire old row — **significant WAL bloat** |

> ⚠️ **Biggest risk with CDC**: A stale/lagging replication slot forces the primary to **retain WAL indefinitely** until the consumer catches up — this can fill your disk and crash the primary.

---

### Read Replica Impact

Uses **physical (streaming) replication** — much lighter on the primary:

| Factor | Detail |
|---|---|
| **WAL overhead** | Minimal — same WAL already being written, just streamed |
| **WAL sender process** | One `walsender` per replica (lightweight) |
| **No slot risk by default** | Replicas use `recovery_min_apply_delay` / streaming — no slot retention by default |
| **Replication lag** | Replica can fall behind under write storms, but doesn't block primary |
| **`hot_standby_feedback`** | If enabled, replica can delay VACUUM on primary to avoid query cancellations — minor overhead |

---

## Configuration Knobs That Matter

```sql
-- For CDC (logical replication)
wal_level = logical           -- required (vs replica for physical)
max_replication_slots = 10    -- one per CDC consumer
max_wal_senders = 10
wal_keep_size = 1GB           -- safety buffer

-- Monitor slot lag (CRITICAL)
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
FROM pg_replication_slots;

-- For Read Replica
wal_level = replica           -- sufficient
max_wal_senders = 5
```

---

## When to Choose What

**Choose CDC when:**
- Feeding a data warehouse, Kafka, or external system
- You need an audit trail / event history
- Targets are non-PostgreSQL (Snowflake, BigQuery, Elasticsearch)
- You want selective table/column capture

**Choose Read Replica when:**
- You need to offload analytical/read queries from the primary
- You need HA and fast failover capability
- Targets must run live SQL queries
- Simplicity of operations is a priority

**Use Both together** (common production pattern):
```
Primary → Physical Replica → Logical CDC from replica
```
Running CDC off a **standby** instead of the primary offloads logical decoding CPU/WAL overhead entirely from the primary — the best of both worlds.
-------------------------------------

## Can We Use a Read Replica for CDC?

### Short Answer: **Yes — from PostgreSQL 16, and partially before that**

---

### PostgreSQL Version Breakdown

#### ✅ PostgreSQL 16+ — Fully Supported (Native)

PostgreSQL 16 introduced **logical replication from standby**, making this officially supported:

```sql
-- On PRIMARY (required setting)
wal_level = logical    -- must be set on primary, replica inherits it

-- On REPLICA — create logical replication slot
SELECT pg_create_logical_replication_slot('cdc_slot', 'pgoutput');

-- CDC tool connects to REPLICA instead of primary
-- Debezium, pglogical etc. point to replica host
```

**What pg16 enables:**
- Logical decoding runs **on the standby**
- Primary is completely offloaded from CDC decoding overhead
- Slot is created and maintained on the replica

---

#### ⚠️ PostgreSQL 10–15 — Workaround Possible (Not Native)

Before pg16, logical replication slots could not be created on standbys natively. Workarounds existed:

```
Option 1: pg_logical extension
  → Third-party, complex, limited support

Option 2: Create slot on primary, decode on primary
  → Defeats the purpose (still hits primary)

Option 3: Logical standby (pg_logical promote trick)
  → Operationally risky
```

---

### Architecture: CDC from Replica (Recommended Pattern)

```
┌─────────────┐    Physical WAL Stream    ┌──────────────────┐
│   PRIMARY   │ ────────────────────────► │   READ REPLICA   │
│             │                           │  (wal_level=     │
│ No CDC load │                           │   logical,       │
│             │                           │   inherited)     │
└─────────────┘                           └────────┬─────────┘
                                                   │
                                          Logical Decoding
                                          (runs here, not
                                           on primary)
                                                   │
                                          ┌────────▼─────────┐
                                          │   CDC Consumer   │
                                          │ (Debezium/Kafka  │
                                          │  Connect etc.)   │
                                          └──────────────────┘
```

---

### Key Constraints & Gotchas

```sql
-- 1. Primary MUST have wal_level = logical
--    (not just 'replica') — even if CDC runs on standby
SHOW wal_level;  -- must return 'logical' on primary

-- 2. Monitor replication slot lag on replica
SELECT slot_name,
       active,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS lag_size
FROM pg_replication_slots
WHERE slot_type = 'logical';

-- 3. If replica is promoted (failover), CDC slot is lost
--    Must recreate slot on new primary
```

| Constraint | Detail |
|---|---|
| **`wal_level=logical` on primary** | Still required even in pg16 — slight WAL overhead on primary remains |
| **Slot survives failover?** | ❌ No — logical slots on replica are lost on promotion |
| **Cascading replica CDC** | ❌ Not supported — CDC replica must be a direct standby of primary |
| **DDL changes** | CDC consumers must handle schema changes carefully |
| **Replica lag affects CDC** | If replica lags, CDC events are delayed too |

---
----------------------------------------------

### Summary

| Scenario | CDC from Primary | CDC from Replica |
|---|---|---|
| **Primary load** | Higher (decoding on primary) | Lower ✅ |
| **pg version needed** | Any (pg10+) | pg16+ for native |
| **Failover safe** | Slot survives | Slot lost on promotion ⚠️ |
| **WAL overhead on primary** | Yes | Yes (wal_level=logical still needed) |
| **Recommended for prod?** | Only if <pg16 | ✅ Yes for pg16+ |

> **Best practice**: On PostgreSQL 16+, always run CDC from a dedicated replica. Keep a separate replica for read queries and HA — don't mix roles.
---------------------
## Great Insight — But It's More Nuanced Than That

### The Core Problem You're Referring To

When CDC runs on the **primary** with a logical replication slot, and the consumer lags or disconnects:

```
Primary keeps accumulating WAL
because slot says "consumer hasn't consumed up to this LSN yet"
                    ↓
WAL cannot be recycled/deleted
                    ↓
Disk fills up → Primary crashes 💥
```

---

## Does Running CDC from Replica Fully Solve This?

### **Partially YES — but the risk doesn't fully disappear**

It depends on **where the logical replication slot lives**.

---

### Scenario Breakdown

#### Case 1: Logical Slot on PRIMARY → CDC reads from Replica
```
PRIMARY (slot here) → WAL stream → REPLICA → CDC Consumer
```

| Situation | WAL retained on... | Disk risk on Primary? |
|---|---|---|
| CDC consumer lags | PRIMARY (slot is here) | ✅ YES — still at risk |
| Sync broken | PRIMARY holds WAL | ✅ YES — still at risk |

> ❌ **Moving CDC reader to replica does NOT help** if the slot still lives on the primary. The primary still waits for the slot's LSN to advance.

---

#### Case 2: Logical Slot on REPLICA (pg16+) → CDC reads from Replica
```
PRIMARY → WAL stream → REPLICA (slot here) → CDC Consumer
```

| Situation | WAL retained on... | Disk risk on Primary? |
|---|---|---|
| CDC consumer lags | REPLICA (slot is here) | ⚠️ Partially |
| Sync broken | REPLICA holds WAL | ✅ Primary mostly safe |

---

### Why "Partially" — The Hidden Dependency

Even with slot on replica (pg16+), the primary is **not completely free**:

```
┌─────────────────────────────────────────────────────┐
│                    PRIMARY                          │
│                                                     │
│  Must retain WAL until replica's restart_lsn        │
│  (the slot on replica reports its LSN back           │
│   to primary via physical replication feedback)     │
└─────────────────────────────────────────────────────┘
```

```sql
-- Primary tracks this via:
SELECT
  application_name,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag
FROM pg_stat_replication;
```

So if the **replica itself** falls behind or disconnects:
- The logical slot lag on replica causes replica to request older WAL from primary
- Primary must retain that WAL
- Disk pressure moves to primary **indirectly**

---

### Full Risk Comparison Table

| Setup | Consumer Lags | Replica Disconnects | Primary Disk Risk |
|---|---|---|---|
| Slot on Primary, CDC on Primary | WAL piles on primary | — | 🔴 High |
| Slot on Primary, CDC on Replica | WAL piles on primary | — | 🔴 High |
| Slot on Replica (pg16+), CDC on Replica | WAL piles on replica | WAL piles on primary | 🟡 Medium |
| **Slot on Replica + Safety configs** | Replica disk at risk | Managed by timeout | 🟢 Low |

---

### The Real Solution — Safety Configs

Regardless of where the slot lives, these configs are your **true protection**:

```sql
-- 1. Limit how much WAL a slot can retain (pg13+)
max_slot_wal_keep_size = '10GB'   -- slot is INVALIDATED if lag exceeds this
-- ⚠️ invalidated slot = CDC must resync from scratch, but primary is safe

-- 2. Automatically drop inactive slots (pg17+)
idle_replication_slot_timeout = '24h'  -- drop slot if idle too long

-- 3. Monitor slot lag proactively
SELECT slot_name,
       active,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS wal_retained
FROM pg_replication_slots
ORDER BY restart_lsn;

-- 4. Alert threshold (run via monitoring/cron)
-- Alert if any slot retains > 5GB
```

---

### Recommended Production Architecture

```
┌──────────────────────────────────────────────────┐
│                   PRIMARY                        │
│  wal_level = logical                             │
│  max_slot_wal_keep_size = 10GB  ← safety net     │
│  No replication slots here      ← clean          │
└────────────────┬─────────────────────────────────┘
                 │ Physical WAL Stream
                 ▼
┌──────────────────────────────────────────────────┐
│              CDC REPLICA (pg16+)                 │
│  Logical slot lives here                         │
│  max_slot_wal_keep_size = 10GB  ← safety net     │
│  CDC consumer connects here                      │
└────────────────┬─────────────────────────────────┘
                 │ Change Events
                 ▼
┌──────────────────────────────────────────────────┐
│         Kafka / Debezium / Target DB             │
└──────────────────────────────────────────────────┘
         +
┌──────────────────────────────────────────────────┐
│           READ REPLICA (separate)                │
│  For read queries / HA failover                  │
│  No logical slots here                           │
└──────────────────────────────────────────────────┘
```

---

### Key Takeaways

| Claim | Verdict |
|---|---|
| CDC from replica eliminates primary disk risk | ❌ Not fully — depends on slot location |
| Slot on replica (pg16+) reduces primary risk | ✅ Yes — significantly |
| `max_slot_wal_keep_size` is the real safety net | ✅ Yes — use on both primary and replica |
| Replica disconnect can still pressure primary | ✅ Yes — physical replication lag still retains WAL |
| Separate CDC replica from HA replica | ✅ Best practice always |


[1] https://dev3lop.com/blog/differences-between-postgresql-and-sql-server/

[2] https://www.pgedge.com/blog/postgresql-replication-and-upcoming-logical-replication-improvements-in-postgresql-16

[3] https://www.percona.com/blog/setting-up-streaming-replication-postgresql/

[4] https://www.enterprisedb.com/blog/postgres-storage-system-raid-levels-compare-performance-costs-das-nas-iscsi-nfs

[5] https://medium.com/@neeraj1.shri/how-chatgpt-pushed-postgresql-to-its-limits-lessons-in-database-scaling-d4d2bd7a9fcd

[6] https://severalnines.com/blog/postgresql-streaming-replication-vs-logical-replication/

[7] https://docs.oracle.com/en-us/iaas/Content/postgresql/storage-best-practices.htm

[8] https://blog.ydb.tech/when-postgres-is-not-enough-performance-evaluation-of-postgresql-vs-distributed-dbmss-23bf39db2d31

[9] https://algomaster.io/learn/system-design-interviews/postgresql

[10] https://docs.vultr.com/set-up-highly-available-postgresql-replication-cluster-on-ubuntu-20-04-server

[11] https://www.datadoghq.com/blog/engineering/postgresql-ha-kubernetes/

[12] https://medium.com/@tirthraj2004/introduction-to-database-clustering-using-postgresql-docker-and-pgpool-ii-ac2a7bf96a5f

[13] https://www.rapydo.io/blog/event-driven-architectures-and-databases-can-sql-keep-up

[14] https://blog.devgenius.io/inside-postgresql-replication-wal-logical-slots-and-cdc-3d711c8191fe

[15] https://www.tinybird.co/blog/change-data-capture-tools

[16] https://ijaniszewski.medium.com/cdc-explained-streaming-database-changes-with-kafka-and-debezium-cff3e03afae7

[17] https://techcommunity.microsoft.com/blog/adforpostgresql/performance-tuning-for-cdc-managing-replication-lag-in-azure-database-for-postgr/4473232

[18] https://www.percona.com/blog/logical-replication-decoding-improvements-in-postgresql-13-and-14/

[19] https://medium.com/@bladepipe.ltd/building-cdc-pipelines-heres-how-mysql-and-postgresql-really-differ-86bc49aed78c

[20] https://blog.dataengineerthings.org/building-a-cdc-pipeline-part-1-postgresql-wal-internals-b5d225031dc5

[21] https://www.ssp.sh/brain/change-data-capture-cdc/

[22] https://ideas.paasup.io/global/postgresql-starrocks-cdcen/

[23] https://medium.com/@shimont85/optimizing-debezium-cdc-preventing-postgresql-wal-accumulation-c15a6ab8d854

[24] https://www.cybertec-postgresql.com/en/detecting-performance-problems-easily-in-postgresql/

[25] https://www.cybertec-postgresql.com/en/end-of-the-road-for-postgresql-streaming-replication/

[26] https://www.cdata.com/blog/postgresql-etl-tools-2025

[27] https://medium.com/@Nexumo_/7-logical-replication-tweaks-for-htap-ish-postgres-1e580ec19f33

[28] https://www.linkedin.com/posts/vitobotta_today-one-replica-in-one-of-our-cloudnativepg-activity-7414328096205594624-rK4b

[29] https://pipeline2insights.substack.com/p/change-data-capture-cdc-fundamentals-for-data-engineers

