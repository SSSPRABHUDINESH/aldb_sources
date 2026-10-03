# **☁️ GCP Database Architecture: Complete Master Notes**

## **📊 Master Comparison Table (All GCP Databases)**

| Database Service | Data Model | Primary Workload | Storage Medium | Read/Write Latency | Scalability Model | Primary Use Cases |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **Cloud SQL** | Relational (RDBMS) | OLTP (Transactional) | Persistent Disk (SSD / HDD) | Single-digit ms | Vertical (Read replicas for read-scaling) | Standard web apps, ERP, CRM, transactional backends |
| **AlloyDB** | Relational (PostgreSQL) | Demanding OLTP & HTAP | Disaggregated distributed block storage | Sub-millisecond to low ms | Vertical compute, auto-scaling reads, decoupled storage | Enterprise-scale PostgreSQL, high-throughput transactions |
| **Memorystore** | In-Memory NoSQL (Key-Value) | Low-latency Cache / Store | RAM (System Memory) | Sub-millisecond (Microseconds) | Vertical RAM sizing / Cluster sharding | Session storage, application caching, real-time counters |
| **Firestore** | NoSQL (Document Store) | OLTP / Real-time Sync | Distributed SSD multi-region storage | Single-digit ms | Fully horizontal (Automatic autoscaling) | Mobile/web backends, user profiles, collaborative apps |
| **Cloud Bigtable** | NoSQL (Wide-Column) | Heavy OLTP / Ingestion | Colossus (Distributed File System \- SSD/HDD) | Sub-10ms (Down to \<1ms) | Horizontal (Node additions without storage rebalance) | IoT telemetry, time-series, financial tick feeds, AdTech |
| **BigQuery** | Relational / Columnar | OLAP (Analytical) | Colossus (Capacitor columnar format) | Seconds to minutes (Scan-heavy) | Serverless dynamic compute slots | Enterprise data warehousing, BI dashboards, petabyte SQL |

## **Decision Tree Architecture**

                         WORKLOAD EVALUATION  
                                  │  
         ┌────────────────────────┴────────────────────────┐  
         ▼                                                 ▼  
   Transactional (OLTP)                              Analytical (OLAP)  
         │                                                 │  
   ┌─────┴─────────────────────┐                           ▼  
   ▼                           ▼                        BigQuery  
Relational?                  NoSQL?                 (Data Warehouse / BI)  
   │                           │  
   ├─────────────┐             ├──────────────────────────┐  
   ▼             ▼             ▼                          ▼  
Standard     Demanding      Document                 Wide-Column  
Cloud SQL    AlloyDB        Firestore                 Bigtable  
(MySQL/PG)   (High-Perf PG) (Web/Mobile Apps)        (IoT/Time-Series/AdTech)  
                               │  
                               ▼  
                       Low-Latency Cache?  
                               │  
                               ▼  
                          Memorystore  
                         (Redis/Valkey)

## **1\. Cloud SQL**

### **1.1 Overview & Core Purpose**

Cloud SQL is Google Cloud's fully managed relational database engine supporting **PostgreSQL, MySQL, and SQL Server**. It is designed for standard transactional (OLTP) workloads where ACID compliance, strong relational integrity, and pre-existing SQL schemas are non-negotiable requirements.

### **1.2 Caching Strategy & Access Pattern**

Cloud SQL acts primarily as the disk-backed **System of Record / Source of Truth**. To prevent read exhaustion from high-concurrency requests, caching strategies are implemented via the Cache-Aside pattern using an in-memory layer (like Memorystore).

\[ Client App \] ───(1. Cache-Aside Check)───► \[ Memorystore (Redis) \]  
      │                                             │  
 (2. DB Fallback on Miss)                           │ (Cache HIT)  
      ▼                                             ▼  
\[ Cloud SQL (Primary) \] ───(Async Replication)───► \[ Read Replicas (Read Scaling) \]

### **1.3 Storage Hierarchy & Comparisons**

| Feature | Cloud SQL | Self-Managed DB on GCE | AlloyDB |
| :---- | :---- | :---- | :---- |
| **Management** | Fully Managed (Automated backups, patches, failover) | Self-Managed (OS, DB software, manual HA) | Fully Managed (Engine-level AI optimizations) |
| **Storage Layer** | Network-Attached Persistent Disk (SSD / HDD) | Attached PD or Local SSD | Decoupled, multi-zoned distributed storage engine |
| **Max Read Scaling** | Cross-region & read replicas | Manual replication setup | Linear read pools with auto-scaling |

### **1.4 Under the Hood: Persistence & Durability**

> * **Persistence Engine:** Stores physical data on Google Persistent Disks. Supports *Automatic Storage Increase* to dynamically resize storage without requiring instance restarts.  
> * **Write-Ahead Logging (WAL):** Transactions are appended to the engine-specific write-ahead log prior to memory buffer commit.  
> * **High Availability (HA):** Implements a regional primary-standby architecture across separate zones using synchronous block-level replication over persistent disks. Failover completes automatically within 60 seconds without data loss.  
> * **Backups & PITR:** Automated daily backups combined with continuous transaction logging provide Point-in-Time Recovery (PITR) down to the exact second.

### **1.5 GCP Architecture & Provisioning**

┌──────────────────────────────────────┐          ┌──────────────────────────────────────┐  
│        Customer VPC Network          │          │    Google Managed Service Network    │  
│                                      │          │                                      │  
│  \[ GKE Pod / Workload Identity \]     │          │         Zone A        Zone B         │  
│                 │                    │          │       ┌─────────┐   ┌─────────┐      │  
│                 ▼                    │          │       │ Primary │──►│ Standby │      │  
│       \[ Cloud SQL Auth Proxy \]       │          │       └────┬────┘   └─────────┘      │  
│                 │                    │          │            │ (Sync Storage Layer)    │  
│                 └──( Private IP / PSA )─────────┼────────────┘                         │  
└──────────────────────────────────────┘          └──────────────────────────────────────┘

> * **Private Connectivity:** Connected via Private Services Access (PSA) VPC peering. Requires no public IP allocation.  
> * **Authentication & Security:** Uses IAM Database Authentication or Cloud SQL Auth Proxy with ephemeral mTLS certificates; credentials managed via Secret Manager.

### **1.6 Sample Structure & Example**

CREATE TABLE customers (  
    customer\_id SERIAL PRIMARY KEY,  
    full\_name VARCHAR(100) NOT NULL,  
    email VARCHAR(150) UNIQUE NOT NULL,  
    created\_at TIMESTAMP DEFAULT CURRENT\_TIMESTAMP  
);

CREATE TABLE orders (  
    order\_id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),  
    customer\_id INT REFERENCES customers(customer\_id),  
    total\_amount NUMERIC(10, 2\) NOT NULL,  
    order\_status VARCHAR(20) CHECK (order\_status IN ('PENDING', 'COMPLETED', 'CANCELLED'))  
);

## **2\. AlloyDB for PostgreSQL**

### **2.1 Overview & Core Purpose**

AlloyDB is a fully managed, 100% PostgreSQL-compatible relational database built specifically for enterprise-grade transactional and hybrid transactional/analytical (HTAP) workloads, offering up to 4x faster transactional throughput and up to 100x faster analytical throughput than standard PostgreSQL.

### **2.2 Caching Strategy & Access Pattern**

AlloyDB incorporates a built-in multi-tier cache that minimizes disk I/O without requiring external caching layers for many heavy read workloads:

> * **Tier 1:** Primary Database Memory Buffer (RAM).  
> * **Tier 2:** Ultra-fast Local NVMe UltraSSD cache managed dynamically by the storage layer.  
> * **Columnar Engine:** Automatically generates and stores an in-memory columnar representation of transactional data for real-time analytical queries.

### **2.3 Storage Hierarchy & Comparisons**

| Dimension | Cloud SQL for PostgreSQL | AlloyDB for PostgreSQL |
| :---- | :---- | :---- |
| **Storage Model** | Monolithic Persistent Disk bound to VM | Disaggregated, distributed shared storage layer |
| **WAL Processing** | Compute node processes and writes full WAL | Offloaded WAL processing directly to distributed storage |
| **Analytical Offload** | Row-based execution on primary/replica | Built-in Columnar Engine with vectorized execution |
| **Failover Time** | Under 60 seconds | Typically under 10 seconds (Storage-level HA) |

### **2.4 Under the Hood: Persistence & Durability**

> * **Decoupled Compute & Storage:** Storage exists as an independent, multi-zoned service that scales dynamically up to 128TB.  
> * **Log-Structured Storage:** The compute instance sends only minimal WAL logs over the internal network. The distributed storage layer processes, materializes, and persists the database blocks independently.  
> * **Continuous Backup:** Backups run continuously without performance degradation or I/O locks on the primary compute instance.

### **2.5 GCP Architecture & Provisioning**

┌────────────────────────────────────────────────────────────────────────┐  
│                        AlloyDB Cluster (Regional)                      │  
│                                                                        │  
│  ┌──────────────────────────┐          ┌────────────────────────────┐  │  
│  │ Primary Instance (Zone A)│          │ Read Pool Instances (Auto) │  │  
│  └─────────────┬────────────┘          └─────────────┬──────────────┘  │  
│                │ (Only WAL records)                  │                 │  
│                ▼                                     ▼                 │  
│  ════════════════════════════════════════════════════════════════════  │  
│         Decoupled, Multi-Zone Distributed Storage Layer (128TB)        │  
│  ════════════════════════════════════════════════════════════════════  │  
└────────────────────────────────────────────────────────────────────────┘

> * **Provisioning Unit:** Configured as a *Cluster* containing storage, an active *Primary Instance*, and one or more independent *Read Pools*.  
> * **VPC Integration:** Deployed exclusively inside customer-peered VPC networks via Private Services Access (PSA).

### **2.6 Sample Structure & Example**

\-- HTAP Execution leveraging AlloyDB Columnar Engine  
ALTER TABLE financial\_transactions SET (google\_columnar\_engine.enabled \= true);

SELECT   
    DATE\_TRUNC('month', transaction\_date) AS month,  
    account\_type,  
    SUM(amount) AS total\_settlement,  
    AVG(processing\_latency\_ms) AS avg\_latency  
FROM financial\_transactions  
WHERE transaction\_date \>= '2026-01-01'  
GROUP BY 1, 2  
ORDER BY 1 DESC;

## **3\. Memorystore (Redis & Valkey)**

### **3.1 Overview & Core Purpose**

Memorystore is a fully managed in-memory data store service built on open-source Redis and Valkey engines. It operates strictly in RAM to deliver microsecond-level response times for caching, active session state, pub/sub communication, and transient real-time workloads.

### **3.2 Caching Strategy & Access Pattern**

> * **Cache-Aside (Lazy Loading):** Applications inspect Memorystore first. On a cache miss, data is read from the underlying database and populated into Memorystore.  
> * **Write-Through / Write-Behind:** Synchronous or asynchronous cache updates triggered whenever transactional writes hit the backend database.  
> * **TTL & Eviction:** Keys are assigned Time-To-Live (TTL) values, controlled by eviction policies such as allkeys-lru or volatile-lru when memory capacity reaches thresholds.

### **3.3 Storage Hierarchy & Comparisons**

| Attribute | Memorystore (Redis/Valkey) | Cloud SQL | Firestore |
| :---- | :---- | :---- | :---- |
| **Storage Medium** | **RAM (Volatile)** | Persistent Disk (Non-volatile) | Distributed SSD (Non-volatile) |
| **Query Primitive** | Key lookup / Data structure ops | SQL Queries & Joins | Document property filters |
| **Primary Role** | Acceleration layer (Ephemeral) | Source of Truth (Transactional) | Source of Truth (Document) |

### **3.4 Under the Hood: Persistence & Durability**

> * **RDB (Snapshots):** Periodically saves a full point-in-time binary snapshot of memory to Persistent Disk. Restarts load from the last saved snapshot.  
> * **AOF (Append-Only File):** Logs every write mutation sequentially to disk (equivalent to a relational database WAL). Configured via appendfsync everysec (default) or always.  
> * **High Availability Tier:** Standard Tier deploys cross-zone primary and replica instances with automated health checking and sub-minute failover.

### **3.5 GCP Architecture & Provisioning**

┌──────────────────────────────────────┐          ┌──────────────────────────────────────┐  
│          Customer VPC Network        │          │    Google Managed Service Network    │  
│                                      │          │                                      │  
│  \[ GKE Pod / GCE / Cloud Run \]       │          │   \[ Primary Node \]  \[ Replica Node \] │  
│                 │                    │          │        (RAM)            (RAM)        │  
│                 └──( VPC Peering )───┼──────────┼──────────┴────────────────┘          │  
│                                      │          │      Assigned IP: 10.0.0.15:6379     │  
└──────────────────────────────────────┘          └──────────────────────────────────────┘

> * **Tiers:** *Basic Tier* (single node, no SLA, dev/ephemeral) vs. *Standard Tier* (HA with automatic cross-zone failover).  
> * **Networking:** Exposes a private IP within the customer's VPC via Private Services Access.

### **3.6 Sample Structure & Example**

\# Key-Value Pair with Expiration (Session Store)  
SET session:user\_9841 '{"auth": true, "role": "admin"}' EX 3600

\# Sorted Set (Real-Time Leaderboard)  
ZADD leaderboard:weekly 1450 "user\_9841"  
ZADD leaderboard:weekly 2300 "user\_1022"

\# Retrieve Top 10 Users  
ZREVRANGE leaderboard:weekly 0 9 WITHSCORES

## **4\. Firestore**

### **4.1 Overview & Core Purpose**

Firestore is a fully managed, serverless NoSQL document database engineered for web, mobile, and serverless applications. It provides real-time client data synchronization, offline caching support, and automated horizontal scaling without manual sharding.

### **4.2 Caching Strategy & Access Pattern**

> * **Client-Side Local Cache:** Native client SDKs maintain a local persistent cache enabling full offline reads and writes that synchronize automatically upon network reconnection.  
> * **Read Indexing:** Every single field within a Firestore document is automatically indexed by default, enabling flexible compound filtering without manual index pre-allocation.

### **4.3 Storage Hierarchy & Comparisons**

| Feature | Firestore | Memorystore (Redis) | Cloud SQL |
| :---- | :---- | :---- | :---- |
| **Data Layout** | Hierarchical Collections & Documents | Flat Key-Value / In-Memory Structures | Relational Tables & Rows |
| **Internal Schema** | Schema-less JSON/BSON-like documents | Opaque string / Key values | Strict schema definitions |
| **Real-Time Sync** | Native WebSocket listener streams | Redis Pub/Sub | Not natively supported |

### **4.4 Under the Hood: Persistence & Durability**

> * **Replication & Consensus:** Built on Google's Spanner-based TrueTime and Paxos consensus layer. Writes are replicated synchronously across multiple availability zones (or multiple regions) before returning commit success.  
> * **ACID Transactions:** Full support for multi-document ACID transactions across collections.  
> * **Scalability Limits:** Scales horizontally to millions of concurrent connections; sustained writes to a single document are bounded at 1,000 writes/second.

### **4.5 GCP Architecture & Provisioning**

\[ Mobile / Web App / Cloud Run \]  
               │  
               ▼ (HTTPS / gRPC / WebSockets)  
┌────────────────────────────────────────────────────────────────────────┐  
│                        Firestore Service Layer                         │  
│  ┌──────────────────────────────────────────────────────────────────┐  │  
│  │ Regional / Multi-Regional Paxos Replication Group                │  │  
│  │   \[ Zone A Storage \] ─── \[ Zone B Storage \] ─── \[ Zone C Storage\] │  │  
│  └──────────────────────────────────────────────────────────────────┘  │  
└────────────────────────────────────────────────────────────────────────┘

> * **Location Types:** Regional (lower write latency) or Multi-Regional (maximum availability, 99.999% SLA).  
> * **Modes:** Native Mode (Standard document database with real-time listeners) vs. Datastore Mode (Backward compatibility for legacy backends).

### **4.6 Sample Structure & Example**

// Hierarchy: Collection ("users") \-\> Document ("user\_101") \-\> Subcollection ("orders")  
{  
  "document\_path": "users/user\_101",  
  "data": {  
    "name": "Dinesh Kumar",  
    "city": "Hyderabad",  
    "is\_active": true,  
    "skills": \["GCP", "Kubernetes", "PostgreSQL"\],  
    "profile\_metadata": {  
      "tier": "Enterprise",  
      "registered\_at": "2026-03-15T08:30:00Z"  
    }  
  }  
}

## **5\. Cloud Bigtable**

### **5.1 Overview & Core Purpose**

Cloud Bigtable is a sparsely populated, distributed wide-column NoSQL database engine capable of scaling to petabytes of data with consistent single-digit millisecond latency. It powers high-throughput workloads such as IoT sensor streaming, financial market data, fraud detection, and operational time-series feeds.

### **5.2 Caching Strategy & Access Pattern**

> * **Sequential Row-Key Indexing:** Data is indexed and lexicographically sorted exclusively by a single primary **Row Key**.  
> * **Scan Optimization:** Access patterns must be designed for single row-key lookups or contiguous row-key range scans. Full table scans must be avoided.  
> * **Write Throughput:** Optimized for append-heavy streaming pipelines directly from Pub/Sub, Dataflow, or Kafka.

### **5.3 Storage Hierarchy & Comparisons**

| Attribute | Cloud Bigtable | Firestore | BigQuery |
| :---- | :---- | :---- | :---- |
| **Architecture** | Distributed Wide-Column NoSQL | Distributed Document Store | Serverless Columnar Data Warehouse |
| **Ingestion Scale** | Millions of writes/sec (Terabytes/Petabytes) | Moderate throughput per document | Streaming buffer & bulk ingestion |
| **Query Engine** | Row-key lookup / prefix range scan | Indexed document field filters | Full SQL aggregations & analytical joins |

### **5.4 Under the Hood: Persistence & Durability**

> * **Disaggregated Compute/Storage:** Compute processing nodes are completely separated from the storage layer.  
> * **Colossus Storage System:** Physical data blocks are split into dynamic chunks called *Tablets* and persisted across Google's distributed Colossus file system.  
> * **Non-Disruptive Resizing:** Adding or removing compute nodes re-indexes tablet pointers instantly without requiring physical data rebalancing or sharding downtime.

### **5.5 GCP Architecture & Provisioning**

┌────────────────────────────────────────────────────────────────────────┐  
│                        Bigtable Instance / Cluster                     │  
│                                                                        │  
│   \[ Node Pool (Compute) \] ── (Pointers) ──► \[ Tablets (Row Ranges) \]   │  
│   Node 1  Node 2  Node 3                     Tablet A  Tablet B        │  
│      │       │       │                            │         │          │  
│   ═══╪═══════╪═══════╪════════════════════════════╪═════════╪════════  │  
│      ▼       ▼       ▼                            ▼         ▼          │  
│      Colossus Distributed File Storage Layer (Replicated SSD/HDD)      │  
└────────────────────────────────────────────────────────────────────────┘

> * **App Profiles:** Configure routing policies (Single-Cluster routing for strong consistency vs. Multi-Cluster routing for high availability and failover).  
> * **Disk Selection:** SSD (standard for low-latency operational workloads) or HDD (large batch-historical datasets \>10TB with relaxed latency).

### **5.6 Sample Structure & Example**

\-- Conceptual Table Structure with Column Families:  
Row Key: "DEVICE\#9021\#TIMESTAMP\#1774684800"  
\-------------------------------------------------------------------------  
Column Family: telemetry\_data  
  telemetry\_data:temperature \-\> 74.2F (Timestamp: 1774684800\)  
  telemetry\_data:pressure    \-\> 1.02bar (Timestamp: 1774684800\)  
\-------------------------------------------------------------------------  
Column Family: device\_metadata  
  device\_metadata:firmware\_ver \-\> "v3.4.1" (Timestamp: 1774684800\)

## **6\. BigQuery**

### **6.1 Overview & Core Purpose**

BigQuery is Google Cloud's serverless, highly scalable enterprise cloud data warehouse and analytical processing engine (OLAP). It allows SQL-based analysis across petabytes of historical structured and semi-structured data without requiring server infrastructure provisioning or database maintenance.

### **6.2 Caching Strategy & Access Pattern**

> * **Query Results Cache:** Exact-match queries executed within a 24-hour window return instantly from the free query cache if the underlying table data remains unchanged.  
> * **Bi Engine:** An in-memory analysis service that accelerates dashboards and BI queries (e.g., Looker, Tableau) down to sub-second response times.  
> * **Columnar Scans:** Scans only the specific columns referenced in the SQL query, significantly reducing processed byte volumes.

### **6.3 Storage Hierarchy & Comparisons**

| Dimension | BigQuery (OLAP) | Cloud SQL / AlloyDB (OLTP) | Cloud Bigtable (NoSQL) |
| :---- | :---- | :---- | :---- |
| **Data Orientation** | **Columnar** (Capacitor format) | **Row-based** (B-Trees / Pages) | **Wide-Column** (Key-Sorted) |
| **Query Profile** | Aggregations across millions/billions of rows | High-frequency point lookups & mutations | High-speed range scans & streaming appends |
| **Scaling Limit** | Petabytes / Exabytes (Serverless) | Gigabytes to low Terabytes | Petabytes |

### **6.4 Under the Hood: Persistence & Durability**

> * **Compute & Storage Decoupling:** The compute engine (**Dremel**) scales dynamically via dynamic compute units called *Slots*, connected over the multi-terabit **Jupiter** network to the **Colossus** distributed storage tier.  
> * **Storage Format:** Data is organized into **Capacitor**, a proprietary columnar file format optimized for deep compression and vectorized execution.  
> * **Partitioning & Clustering:** Tables can be partitioned (by ingestion time, date, or integer range) and clustered (by high-cardinality keys) to prune scanned data and optimize query costs.

### **6.5 GCP Architecture & Provisioning**

┌────────────────────────────────────────────────────────────────────────┐  
│                        BigQuery Serverless Architecture                │  
│                                                                        │  
│   \[ SQL Client / Looker \] ──► \[ Dremel Execution Engine (Slots) \]      │  
│                                           │                            │  
│   ════════════════════════════════════════╪═════════════════════════   │  
│                 Jupiter Network (Petabit-scale interconnect)           │  
│   ════════════════════════════════════════╪═════════════════════════   │  
│                                           ▼                            │  
│          Colossus Distributed Storage Tier (Capacitor Format)          │  
└────────────────────────────────────────────────────────────────────────┘

> * **Resource Hierarchy:** Project $\rightarrow$ Dataset (Access control boundary & region) $\rightarrow$ Tables / Views.  
> * **Billing Models:** On-Demand (pay per byte scanned) vs. Capacity-based Editions (Standard, Enterprise, Enterprise Plus with dedicated slot reservations).

### **6.6 Sample Structure & Example**

\-- Create Partitioned & Clustered Table  
CREATE TABLE \`project\_id.analytics\_dw.order\_events\`  
(  
    event\_id STRING,  
    event\_timestamp TIMESTAMP,  
    customer\_id INT64,  
    order\_amount NUMERIC,  
    country STRING  
)  
PARTITION BY DATE(event\_timestamp)  
CLUSTER BY country, customer\_id;

\-- Analytical Aggregation Query (Prunes partitions)  
SELECT   
    country,  
    COUNT(DISTINCT customer\_id) AS active\_buyers,  
    SUM(order\_amount) AS total\_revenue  
FROM \`project\_id.analytics\_dw.order\_events\`  
WHERE event\_timestamp \>= '2026-08-01'  
GROUP BY country  
ORDER BY total\_revenue DESC;

## **7\. Production Engineering Scenarios & Interview Playbook**

### **Scenario 1: Application Workload Migration to Cloud SQL**

**Structured 6-Pillar Answer:**

> 1. **Engine Compatibility:** Verify database engine versions, target SQL modes, and installed extensions (e.g., PostGIS for PostgreSQL).  
> 2. **Sizing & Capacity:** Calculate CPU cores, memory limits, and baseline disk I/O requirements based on active connections and working set size.  
> 3. **Network Architecture:** Provision Private Services Access (PSA) for private IP routing; place workloads inside the same GCP region to minimize round-trip latency.  
> 4. **Security:** Configure Cloud SQL Auth Proxy, enforce IAM Database Authentication, and store service account credentials inside Secret Manager.  
> 5. **HA & Disaster Recovery:** Enable regional High Availability (cross-zone primary/standby failover) and configure daily backups with point-in-time recovery (PITR).  
> 6. **Cutover Execution:** Establish continuous CDC (Change Data Capture) via Google Database Migration Service (DMS), sync historical state, and execute application cutover during a planned maintenance window.

### **Scenario 2: Transactional Database Bottleneck Resolution**

                       DATABASE BOTTLENECK DETECTED  
                                    │  
        ┌───────────────────────────┼───────────────────────────┐  
        ▼                           ▼                           ▼  
1\. Heavy Read Traffic       2\. Slow/Unindexed Queries    3\. Connection Starvation  
        │                           │                           │  
  Provision Read              Analyze Query Insights &     Deploy Connection  
  Replicas & Caching          Add Missing Indexes          Pooler (PgBouncer)  
  (Memorystore)

### **Scenario 3: Primary Instance Failure Recovery**

When a Cloud SQL HA primary instance fails:

> * The regional control plane detects a heartbeat loss and triggers automatic failover.  
> * The standby instance in the secondary zone mounts the replicated Persistent Disk volume.  
> * The instance IP address seamlessly maps to the new primary node, completing recovery within 60 seconds.

### **Scenario 4: Accidental Data Deletion Protection**

To recover from accidental data loss or logical table drops:

> * Use **Point-in-Time Recovery (PITR)** to restore a clone instance back to the exact second prior to the errant query.  
> * Extract the dropped data from the restored clone and re-insert it into the production instance.

### **Scenario 5: Secure Credential Management for GKE**

Never commit hardcoded database credentials into Kubernetes manifests or container images:

> * Attach a Google Service Account (GSA) to a Kubernetes Service Account (KSA) using **GKE Workload Identity**.  
> * Deploy the **Cloud SQL Auth Proxy** as a sidecar container to establish local mTLS connectivity.  
> * Inject database passwords dynamically at runtime using the **Secret Manager CSI Driver**.