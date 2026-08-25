# RDS Proxy — How It Works, Why We Need It, and When to Use It

## What problem does it solve?

Without RDS Proxy, every application instance opens its **own direct connections** to the database. In production, this causes several real issues:

1. **Connection exhaustion from too many app instances**
   Databases have a hard limit on max connections (e.g., a `db.t3.medium` might support only ~170). Serverless functions (Lambda), auto-scaling app servers, or microservices — each opening their own connections — can easily blow past that limit. Lambda is the classic case: each concurrent invocation can open a new connection, and with thousands of concurrent invocations, the database's connection limit is exhausted almost instantly.

2. **Connection overhead**
   Establishing a new database connection isn't free — it involves a TCP handshake, TLS negotiation, and authentication. If the app opens/closes connections frequently instead of reusing them, that overhead adds directly to request latency.

3. **Failover downtime**
   During an RDS failover (Multi-AZ failover, or an Aurora writer change), every application instance has to detect the failure and re-establish new connections. This can take anywhere from several seconds to over a minute, during which requests fail.

4. **Idle connections wasting resources**
   Long-lived but mostly idle connections still consume memory and resources on the database instance.

## Why we need it

RDS Proxy exists to manage database connections *for* the application, so the application doesn't have to deal with connection scaling, pooling logic, or failover handling itself.

## How it works

Instead of every app instance opening its own connection to the database, all instances connect to **RDS Proxy** instead. The proxy maintains and reuses a small, warm pool of actual connections to the database. Many "logical" application connections get **multiplexed** onto far fewer "physical" database connections.
<img width="2720" height="1680" alt="rds_proxy_architecture" src="https://github.com/user-attachments/assets/dd5ed2d7-7460-4200-aacf-ceada5dfd047" />

### Key mechanics

- **Connection multiplexing**
  The proxy pins many client connections to a smaller number of backend database connections, reusing them across requests rather than opening/closing per request. This is why it can support thousands of Lambda invocations without exhausting the database's connection limit.

  > **Simple analogy:** Think of RDS Proxy like a taxi dispatch service instead of everyone owning their own car. A small fleet of taxis (backend connections) serves many passengers (app requests) — a taxi drops off one passenger and immediately picks up the next, instead of every passenger needing their own car.

- **Connection pinning (important caveat)**
  If the app uses session-level state (like transactions, temp tables, or session variables), the proxy has to "pin" that client to a specific backend connection for the rest of the session, because that state can't safely be shared. Heavy use of pinning reduces the multiplexing benefit — this is a common gotcha people hit after switching to RDS Proxy.

  > **Simple analogy:** A passenger who says *"wait for me, I have three stops to make and I need the same taxi the whole time"* forces that taxi to stay with them the entire trip. It can't pick up anyone else in between. If most requests are like this (long transactions), you lose most of the pooling benefit — you're effectively back to one taxi per passenger.

- **Fast failover**
  Because the proxy sits between the app and the database, during a Multi-AZ failover it can hold pending connections and transparently redirect them to the new primary — often cutting failover-related downtime from 60+ seconds down to single digits, since the app doesn't have to detect the failure and reconnect itself.

- **IAM authentication integration**
  Can enforce IAM-based authentication and pull credentials from Secrets Manager, so the app doesn't need to embed database passwords directly.

- **TLS termination**
  Can terminate TLS at the proxy, reducing per-connection handshake overhead for the app.

## Important: RDS Proxy is not a data cache

A common misunderstanding is that RDS Proxy "remembers" previous query results. It does not. It only reuses the **physical network connection (the pipe)** — not the data or the query result.

| | RDS Proxy | Cache (e.g., Redis/ElastiCache) |
|---|---|---|
| Reuses | Network connections | Query results / data |
| Query still runs on DB? | Yes, always | No, if cached |
| Solves | Connection exhaustion, failover time | Read latency, DB load |

Every query — whether from a "new" or "reused" connection — still goes all the way to the database to execute. If you want to skip re-running identical queries, that requires a separate caching layer (like Redis) in front of the database.

## When to use RDS Proxy

### Good fit
- Serverless/Lambda-based apps hitting RDS or Aurora directly (the most common use case — Lambda's connection-per-invocation pattern is exactly the problem RDS Proxy solves)
- Apps with many auto-scaling instances that spike connection counts unpredictably
- Applications where failover downtime is business-critical and faster recovery matters
- Microservices architectures where dozens of services each want their own small pool, collectively overwhelming the database

### Not a great fit / limited benefit
- Traditional apps with a small, fixed number of long-running app servers that already do proper connection pooling (e.g., PgBouncer, HikariCP) — added benefit may be minimal, plus it adds a small extra network hop
- Apps that rely heavily on session-level features (prepared statements tied to session state, temp tables, advisory locks) — pinning behavior can negate the pooling benefit
- Extremely latency-sensitive workloads where even the small added hop through the proxy matters more than the connection-management benefit

## One-line summary to remember

> **Multiplexing** = many passengers sharing few taxis (connection reuse).
> **Pinning** = a passenger locking a taxi to themselves for the whole trip, which cancels out the sharing benefit.
> RDS Proxy manages **connections**, not **data** — every query still runs fresh on the database.
