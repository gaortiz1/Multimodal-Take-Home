## 4. **Data Persistence**

### 4.1. Problem Definition  
An asynchronous task system requires a data store that can reliably:  
- **Persist tasks and their evolving state** (`PENDING` → `PROCESSING` → `COMPLETED`/`FAILED`)  
- **Store potentially large text** payloads and structured result data  
- **Support concurrent updates** (status transitions, retry counts) without conflicts  
- **Serve frequent reads** for status polling with low latency  
- **Scale** as the volume of tasks grows into the thousands per minute  

---

### 4.2. Data Model / Schema  
We model each task with the following fields:  

| Field         | Type        | Description                                           |
|---------------|-------------|-------------------------------------------------------|
| `task_id`     | UUID (PK)   | Unique identifier for the task                        |
| `status`      | Enum/String | One of `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED` |
| `text`        | Text        | Original input payload (up to configured max length)  |
| `result`      | JSONB       | Structured output (word count, top words…)            |
| `error`       | Text        | Error message if status=`FAILED`                      |
| `retry_count` | Integer     | Number of retry attempts performed                    |
| `created_at`  | Timestamp   | When the task was first enqueued                      |
| `updated_at`  | Timestamp   | Last status transition                                |

---

### 4.3. Candidate Datastores & CAP Considerations  
- **Relational (PostgreSQL)**  
  - **Pros:** ACID transactions guarantee consistent status updates; rich indexing on `status` and timestamps; JSONB for semi-structured `result`.  
  - **Cons:** Vertical scaling limits; requires careful sharding/partitioning under extreme write loads.  
  - **CAP:** Prioritizes **Consistency** and **Partition tolerance**, may sacrifice some Availability under network partitions (multi-AZ setups).  

- **NoSQL Document Store (MongoDB)**  
  - **Pros:** Flexible schema, easy horizontal sharding, good for high write throughput.  
  - **Cons:** Weaker multi-document transactional guarantees; secondary indexes can lag under heavy load.  
  - **CAP:** Tunable consistency; often eventual consistency for high Availability and Partition tolerance.

- **Key-Value Store (Redis)**  
  - **Pros:** Ultra-low latency, ideal for caching status or small metadata; built-in TTL for auto-expiry.  
  - **Cons:** In-memory by default (persistence via snapshots/AOF); not suited as primary store for large text or audit logs.  
  - **CAP:** Prioritizes **Availability** and **Partition tolerance**, with eventual consistency of persistence.

---

### 4.4. Chosen Solution & Justification  
**Primary Store:** PostgreSQL  
- **Why?**  
  - Guarantees transactional integrity for status transitions and retry updates.  
  - Mature JSONB support handles result payloads without sacrificing queryability.  
  - Simple to index and query for pagination, filtering by status, and sorting by timestamps.  

**Caching Layer (Optional):** Redis  
- Cache recent `task_id → status` lookups to offload read pressure from PostgreSQL for high-frequency polling.  
- Use Redis TTLs to expire old entries, minimizing memory growth.

---

### 4.5. Trade-Offs  
| Aspect          | PostgreSQL                            | MongoDB                               | Redis (Cache)                 |
|-----------------|---------------------------------------|---------------------------------------|-------------------------------|
| **Consistency** | Strong ACID                           | Tunable (eventual by default)         | Eventual (cache may be stale) |
| **Scalability** | Vertical + partitioning/sharding      | Horizontal out-of-the-box             | Horizontal via clustering     |
| **Complexity**  | Medium (schema, migrations)           | Medium-high (sharding config)         | Low (cache only)              |
| **Latency**     | Low-medium                            | Low                                   | Ultra-low                     |
| **Suitability** | Transactional workflows, audit, joins | Flexible schemas, large write volumes | Caching hot status lookups    |
 
