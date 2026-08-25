# Diagnosing Increased Latency in RDS Reads

**Use AWS Cloudwatch Database Insights for metrics to identify the issue.** https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html

When an application reads from RDS and users experience increased latency, the cause usually falls into one of several categories. Let's go through the likely culprits systematically.

---

## 1. Database-side issues (most common)

### Query performance
- Missing or unused indexes on frequently queried columns
- Query plans changed due to stale statistics (needs `ANALYZE`)
- Table scans instead of index lookups as data volume grew
- N+1 query patterns from the application (many small queries instead of one efficient one)

### Resource contention on the RDS instance
- High CPU utilization (check CloudWatch `CPUUtilization`)
- Memory pressure causing increased disk reads (check `FreeableMemory`, buffer cache hit ratio)
- Disk I/O bottleneck — especially if using gp2/gp3 storage and burst credits (`IOPS`) are exhausted
- High `DatabaseConnections` count nearing max — connection pool exhaustion causes queueing

### Locking/blocking
- A write operation locks the data, making read operations wait in line.
- Long-running transactions holding locks
- Deadlocks or lock waits from concurrent writes blocking reads

---

## 2. Connection-related issues

- **Connection pool exhaustion**: application opening too many connections, or not reusing them efficiently  
- **New connection overhead**: if the app isn't pooling connections properly, every request pays the TCP + TLS + auth handshake cost  
- **Connection pool misconfigured** (too small = queueing; too large = overwhelming the DB)

---

## 3. Network-side issues

- Application server and RDS instance in different Availability Zones (adds cross-AZ latency)  
- Security group / route table misconfigurations causing suboptimal paths  
- If using RDS Proxy or a load balancer in front, added hop latency  
- DNS resolution delays if not cached properly  

---

## 4. Application-side issues

- Inefficient ORM-generated queries (common with lazy loading triggering many round trips)  
- Lack of caching layer (e.g., no Redis/ElastiCache in front of frequent reads)  
- Application server itself under CPU/memory pressure, delaying query dispatch  
- Serialization/deserialization overhead for large result sets  

---

## 5. Scaling and growth-related

- Data volume grew but indexing strategy didn't adapt  
- Read replica not used — all reads still hitting the primary, competing with writes  
- Instance class outgrown (e.g., still on `db.t3.medium` but traffic has grown significantly)  

---

# Real-World Causes of Increased RDS Latency

In real production environments, the increased latency almost never has one single cause — it's usually a slow-building combination of things that compound each other. Here's what actually happens most often, roughly in order of how frequently I've seen each one be the real culprit.

---

## 1. The "it worked fine in testing" trap — data growth outpacing indexing

This is the #1 real-world cause. A query is written and tested against a table with a few thousand rows. It works instantly. Six months into production, that table has 50 million rows. The query that used to hit an index now falls back to a sequential scan because:

- A new `WHERE` clause or `JOIN` was added later without updating indexes  
- The query planner's statistics are stale, so it picks a bad execution plan even though an index exists  
- The index exists but isn't *selective* enough at the new data volume (e.g., indexing a boolean column with only two possible values)  

**Key insight:** This doesn't cause a sudden outage. It's a slow, creeping degradation — a query that took 5ms takes 50ms, then 500ms, over months — so nobody notices until users start complaining.

---

## 2. Connection pool exhaustion under real traffic patterns

In testing, you have 1 user. In production, you have 500 concurrent users, and every one of them is trying to grab a connection from a pool sized for a much smaller load. What actually happens:

- Requests start **queueing silently** waiting for a free connection  
- The app logs show the query itself only took 10ms, but the user experienced 2 seconds of latency — because 1.99s of that was spent *waiting for a connection*, not running the query  
- This is the classic case where people look at slow query logs, see nothing wrong, and get confused — because the bottleneck isn't the query, it's the wait to even start the query  

**Note:** This gets worse under traffic spikes (marketing email, flash sale) because pools are usually sized for average load, not peak load.

---

## 3. The "noisy neighbor" query

One bad query — often a reporting query, an admin dashboard, or a batch job — starts consuming disproportionate resources (CPU, I/O, or locks) and slows down *every other* query on the same instance.  

- Users experiencing latency are doing something completely different from whoever triggered the bad query  
- It's hard to trace because the slow user-facing query looks innocent in isolation — the problem is contention, not the query itself  

---

## 4. Lock contention from mixed read/write workloads

Reads getting blocked by writes (or vice versa) is extremely common and often invisible until you specifically look for it. A batch update or a long-running transaction holds locks longer than expected, and reads that should take milliseconds sit in a queue waiting for the lock to release.  

Typical triggers:
- Large data import/migration jobs  
- ORMs that open a transaction and forget to commit/close it promptly  
- Ad-hoc `UPDATE` queries run directly on production without realizing their impact  

---

## 5. Read replica lag being silently ignored

If reads are routed to a read replica for scaling, and the replica falls behind (replication lag), the app either:

- Serves stale data quickly (correctness bug, not latency)  
- Or waits for the replica to catch up, manifesting as *latency* instead of an error  

**Gotcha:** Dev/staging environments rarely have replication lag, so this issue only surfaces in production.

---

## 6. The N+1 query problem hiding behind an ORM

Code like "get all orders, then for each order get the customer" looks clean in the ORM but generates 1 + N actual database round trips.  

- In dev with 10 test records, this is invisible  
- In production with 500 orders, it's 501 queries  
- Even if each query is fast, the *cumulative round-trip latency* adds up quickly, especially with any network hop between app and DB  

---

## Incident Diagnosis Tip

The fastest way to narrow down the root cause is to answer:  
**Is the latency in the query execution itself, or in waiting to get a connection/lock before the query even starts?**  

These point to almost entirely different fixes, and most monitoring tools (like **RDS Performance Insights**) can show you which one it is within minutes.

