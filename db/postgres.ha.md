To route traffic effectively in a PostgreSQL Primary/Replica (Leader/Follower) architecture with connection pooling, you need a component that handles **High Availability (HA) and Failover Routing** alongside PgBouncer.

Because PgBouncer only manages connection pools and doesn't inherently know which Postgres node is currently the healthy primary writer, an HA manager like **Patroni** paired with a routing proxy like **HAProxy** (or a Cloud Load Balancer) is typically introduced.

Here is the complete production architecture design and data flow.

---

## 1. High-Level Architecture Diagram

```
                 [ Application Tier / API Nodes ]
                                │
                                ▼ (Sends all DB traffic)
                 [ PgBouncer Connection Poolers ]
                                │
                                ▼ (Proxies pooled connections)
                 [ HAProxy / Load Balancer Layer ]
                    │                         │
      ┌─────────────┘                         └─────────────┐
      │ (Routes Writes/Reads)                               │ (Routes Read-Only)
      ▼                                                     ▼
┌───────────────────────────────┐             ┌───────────────────────────────┐
│     PostgreSQL Node 1         │             │     PostgreSQL Node 2         │
│   (Current Primary/Leader)    │             │    (Replica/Follower)         │
│  [Patroni Agent Active]       │             │  [Patroni Agent Active]       │
└───────────────────────────────┘             └───────────────────────────────┘
      │                                                     ▲
      └─────────────── Streaming Replication ───────────────┘
                                │
                        [ Consul / etcd ]
                     (Distributed Consensus)

```

---

## 2. Core Architectural Components

### A. Application Layer

Your backend application connects directly to **PgBouncer**, not to the databases. The application sees PgBouncer as if it were the actual PostgreSQL database.

### B. PgBouncer Layer (Connection Pooling)

* **Why it's here:** PostgreSQL handles connections by spawning a heavy operating system process for each user. Thousands of app threads can easily overwhelm Postgres memory.
* **How it handles traffic:** PgBouncer sits in front of the infrastructure, accepting thousands of client connections and funneling them down into a tiny, reusable pool of heavy server connections (e.g., matching 1,000 app connections to just 50 database connections).
* **Where to route next:** PgBouncer forwards its pooled queries directly to the **HAProxy / Load Balancer** layer.

### C. HAProxy / Load Balancer Layer (Traffic Routing)

This is the "traffic cop" of your architecture. You typically expose two distinct ports or VIPs (Virtual IPs) on HAProxy:

* **Port 5000 (Write + Read Port):** Points to the current Primary Postgres node.
* **Port 5001 (Read-Only Port):** Load-balances incoming analytical or read queries across your healthy Followers.

### D. PostgreSQL Cluster + Patroni (High Availability)

* **Patroni** is an orchestration tool that runs as a sidecar process on each database server. It hooks into a distributed consensus store (like **etcd** or **Consul**) to elect the primary leader node.
* Patroni exposes a local HTTP health check endpoint on each database node (e.g., `http://localhost:8008/primary`).
* HAProxy continuously pings this health check endpoint. If Node 1 is the leader, its health check returns `HTTP 200`. If Node 2 is a follower, its endpoint returns `HTTP 503`. HAProxy uses these responses to automatically route traffic to the correct machine.

---

## 3. The Query Lifecycle (How Traffic Flows)

### Case A: Executing a Write (Insert/Update)

1. The **Application** sends an `INSERT` statement to PgBouncer.
2. **PgBouncer** assigns a pooled connection and passes the query to **HAProxy on Port 5000** (The Write Endpoint).
3. **HAProxy** checks its health maps, sees that Postgres Node 1 answered `HTTP 200` to the primary check, and forwards the query to **Node 1**.
4. Node 1 commits the write and streams the transaction asynchronously via **WAL Streaming Replication** to Node 2.

### Case B: Handling an Automatic Failover

1. **Postgres Node 1 crashes** due to a hardware failure.
2. **Patroni** detects the failure via the heartbeat lease expiring in etcd/Consul.
3. Patroni safely promotes **Node 2** to become the new Primary writer.
4. **HAProxy's** health check loop notices the shift within 1–2 seconds: Node 1 now fails health checks, while Node 2 begins responding with `HTTP 200` on the `/primary` endpoint.
5. HAProxy instantly shifts all Port 5000 traffic away from Node 1 over to **Node 2**.
6. Your **Application** and **PgBouncer** stay completely oblivious to the database crash—they never have to change their connection string parameters because they only ever look at HAProxy.

---

## 4. Where should you physically deploy PgBouncer?

You have two design choices for deploying PgBouncer:

1. **Sidecar Deployment (On the App Server):** Run a local PgBouncer instance on every individual application server machine. The app connects to `localhost:6432`. This reduces network tracking overhead because connection accumulation is managed before it leaves the application node.
2. **Centralized Layer (Recommended for large scale):** Run a dedicated cluster of PgBouncer machines behind an internal network load balancer. This makes it easier to manage total global max connection limits on your underlying PostgreSQL nodes.
