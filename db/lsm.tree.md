An **LSM Tree (Log-Structured Merge-Tree)** is a data structure optimized for handling **high-volume write workloads**.

Traditional databases (like standard PostgreSQL or MySQL) typically use **B-Trees** to store data. While B-Trees are incredibly fast for reading data, they require "random writes" to disk, which becomes a massive performance bottleneck when you are writing thousands of records per second.

The LSM Tree solves this by turning random writes into sequential writes, making ingestion incredibly fast. It is the underlying storage engine architecture behind modern NoSQL and distributed databases like **Cassandra, RocksDB, ScyllaDB, Bigtable, and CockroachDB**.

---

## The Core Architecture of an LSM Tree

An LSM Tree doesn't store all its data in one place. Instead, it breaks storage down into distinct **layers** separated by memory and physical storage (disk), processing data in stages.

### 1. The In-Memory Components

* **Write-Ahead Log (WAL):** When a write request (insert/update/delete) hits the database, it is immediately appended to a raw text log file on disk called the WAL. Because this is purely an *append-only* operation, it is lightning fast. The WAL exists solely for disaster recovery—if the server loses power, the database replays this log to recover lost data.
* **MemTable (Memory Table):** Simultaneously, the data is injected into an in-memory buffer called the `MemTable`. The MemTable stores data in a sorted order (usually implemented via a *Skip List* or *Red-Black Tree*).

> **Why writes are so fast:** Once the data is appended to the WAL and stored in the volatile `MemTable`, the database immediately tells the client "Success!" The write lifecycle never waits for a slow disk search.

### 2. The On-Disk Components (SSTables)

When the `MemTable` fills up to a certain threshold (e.g., $32\text{MB}$ or $64\text{MB}$), it is frozen, and its contents are flushed to the disk as an immutable file called a **Sorted String Table (SSTable)**.

* **SSTables are Sorted:** Because the MemTable was already sorted in memory, writing the SSTable to disk is a highly efficient, sequential write.
* **SSTables are Immutable:** Once written to disk, an SSTable is **never modified**. If a user updates an existing record, the database doesn't search the disk to replace the old value. It simply writes a *brand new entry* with a newer timestamp. Deletions are handled by writing a special marker called a **Tombstone**.

---

## How Data Operations Work

Because data is fragmented across multiple on-disk SSTables, reading and updating data behaves differently than in a traditional database.

### The Write Path (Fast)

1. Write to WAL (for crash safety).
2. Insert into MemTable.
3. Return success.

### The Read Path (Slower)

Because the same key might exist in multiple places (with newer versions in memory and older versions on disk), finding a record requires a strategic search order:

1. Check the **MemTable**. If found, return it (it's guaranteed to be the freshest copy).
2. Check the most recent **SSTables on disk**, moving backwards from newer files to older files.
3. Compare timestamps of any matching keys found across SSTables, and return the newest one.

---

## Compaction: Managing the Chaos

Because SSTables are immutable, your disk will eventually fill up with multiple redundant copies of modified rows and dead "tombstone" records. To prevent disk bloat and slow read times, the LSM engine runs a background process called **Compaction**.

During compaction, a background worker selects multiple SSTables, merges their contents, removes duplicate outdated keys, discards deleted records, and writes out a clean, newly sorted SSTable.

The two most common strategies are:

* **Size-Tiered Compaction:** Merges SSTables of roughly equal sizes together. (Optimized for writes).
* **Leveled Compaction:** Divides disk storage into tiers (Level 1, Level 2, Level 3). Each level has a strict capacity limit, and keys do not overlap within a single level. (Optimized for reads).

---

## Mitigating Read Latency with Bloom Filters

The biggest flaw of an LSM Tree is **Read Amplification**—the risk that a read operation has to open and scan 10 different SSTables on disk just to find out a key doesn't even exist.

To solve this, LSM engines pair every SSTable with an in-memory **Bloom Filter**.

* A Bloom Filter is a space-efficient probabilistic data structure that can tell you with 100% certainty if an element is **not** in a set.
* Before the database reads an SSTable from the disk, it checks the Bloom Filter. If the filter says "No, this key is definitely not in this SSTable," the database completely skips reading that file from disk, saving massive amounts of I/O cycles.

---

## B-Tree vs. LSM Tree Summary

| Metric | B-Tree (e.g., PostgreSQL, MySQL) | LSM Tree (e.g., Cassandra, RocksDB) |
| --- | --- | --- |
| **Primary Design Goal** | Optimized for fast reads. | Optimized for intense write throughput. |
| **Write Type** | **Random Writes:** Modifies specific pages directly on disk. | **Sequential Writes:** Appends to logs and flushes memory chunks. |
| **Storage Layout** | Mutable pages on disk. | Immutable sorted files (SSTables). |
| **Space Efficiency** | High (Updates overwrite old data immediately). | Temporary fragmentation (Requires compaction to clean disk space). |
| **Best Use Case** | Relational data, complex queries, banking ledgers. | Time-series data, log aggregators, IoT metrics, high QPS streaming data. |
