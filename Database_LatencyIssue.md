# Diagnosing Increased Latency in RDS Reads

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

## How to Diagnose

The single most common root cause in practice is **missing indexes combined with data growth** — a query that was fast at 10,000 rows becomes slow at 10 million rows if it's doing a full table scan.  

The second most common is **connection pool exhaustion** under increased traffic.  

---

## Next Steps

If you can share specifics:
- Is it a specific query that's slow, or is everything slower uniformly?  
- Is this a sudden spike or gradual degradation?  
- Is it read-heavy or mixed read/write traffic?  

With those details, we can narrow down which of these is most likely in your case.
