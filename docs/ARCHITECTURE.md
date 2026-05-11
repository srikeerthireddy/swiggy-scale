# SwiftEats Architecture Redesign

## 1. Current Architecture Diagram

```text
10M users
   |
   v
┌─────────────────────────────────────────────┐
│ Single Node.js Express server               │
│ 1 process, 1 CPU, 4 GB RAM                  │
│                                             │
│ - Restaurant browsing                       │
│ - Order placement                           │
│ - Payment calls (sync)                      │
│ - Promo validation                          │
│ - Static assets                             │
└─────────────────────────────────────────────┘
   |   ^   ^   ^
   |   |   |   └── ← Failure 5: Static assets consume the same NIC as APIs
   |   |   └─────── ← Failure 4: Promo validation is split SELECT then UPDATE
   |   └─────────── ← Failure 2: Payment calls hold DB connections open
   └─────────────── ← Failure 3: One process saturates around 12K RPS
          |
          v
┌─────────────────────────────────────────────┐
│ Single PostgreSQL instance                  │
│ max_connections = 100                       │
│ no replicas, no pooler                      │
└─────────────────────────────────────────────┘
          ^
          └──────────── ← Failure 1: Pool exhausts at ~394 RPS
```

## 2. New Architecture Diagram

```text
10M users
   |
   v
┌─────────────────────────────────────────────────────────────┐
│ CloudFront CDN                                                │
│ Caches JS/CSS/images for 1 day; menu bootstrap JSON for 5m    │
│ Stale-while-revalidate: 60s for menu cards and restaurant tiles│
└─────────────────────────────────────────────────────────────┘
   |
   v
┌─────────────────────────────────────────────────────────────┐
│ Application Load Balancer + AWS WAF                          │
│ SSL termination, health checks every 5s, 100 req/IP/min      │
└─────────────────────────────────────────────────────────────┘
   |
   +-------------------+-------------------+-------------------+
   |                   |                   |
   v                   v                   v
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Node.js #1   │  │ Node.js #2   │  │ Node.js #3   │   ... 4-20 stateless
│ baseline     │  │ baseline     │  │ baseline     │         instances
└──────────────┘  └──────────────┘  └──────────────┘
   |                   |                   |
   +-------------------+-------------------+
                       |
                       v
                ┌───────────────────┐
                │ Redis Cluster      │
                │ menu cache TTL 5m  │
                │ promo lock: SETNX  │
                │ idempotency: 24h   │
                └───────────────────┘
                       |
                       v
                ┌───────────────────┐
                │ PgBouncer          │
                │ transaction pool   │
                │ 100 DB conns ->    │
                │ thousands of app   │
                │ clients            │
                └───────────────────┘
                       |
              +--------+--------+
              |                 |
              v                 v
   ┌───────────────────┐  ┌───────────────────┐
   │ PostgreSQL primary  │  │ Read replica #1   │
   │ writes only         │  │ restaurant reads  │
   └───────────────────┘  └───────────────────┘
              |
              v
   ┌───────────────────┐  ┌───────────────────┐
   │ Read replica #2    │  │ SQS payment queue │
   │ order history      │  │ + DLQ             │
   └───────────────────┘  └───────────────────┘
                                              |
                                              v
                                   ┌────────────────────────┐
                                   │ Payment worker service │
                                   │ async gateway calls    │
                                   │ writes payment status  │
                                   └────────────────────────┘
```

### Scaling policy

- Baseline Node.js fleet: 4 instances.
- Peak Node.js fleet: 20 instances.
- Scale-out trigger: average CPU > 60% for 3 minutes or p95 latency > 250 ms for 3 minutes.
- Scale-in trigger: average CPU < 35% and p95 latency < 120 ms for 10 minutes.

### Redis behavior

- Cache restaurant menus, restaurant cards, and promo eligibility decisions with a 5 minute TTL.
- Use `SETNX` plus `DECR` for promo redemption so the check and decrement happen atomically.
- Keep an idempotency key for each checkout attempt for 24 hours so retries cannot double-charge or double-apply a promo.

### Database routing

- Writes go to the PostgreSQL primary.
- Read-heavy restaurant browsing goes to replica #1.
- Order-history and analytics reads go to replica #2.
- PgBouncer keeps the database at 100 server connections while allowing many more app-side clients.

### Payment flow

1. The API writes the order intent to PostgreSQL.
2. The API publishes a payment job to SQS and returns a pending state.
3. The payment worker consumes the queue and calls the payment gateway.
4. The worker writes the payment outcome back to PostgreSQL.

That removes the synchronous 200-2000 ms hold from the request path.

## 3. Component Justification Table

| Component | Failure It Prevents | How It Prevents It |
|---|---|---|
| CloudFront CDN | Failure 5: Static asset NIC saturation | Serves images, JS, and CSS from the edge so Node never spends origin bandwidth on static bytes. |
| ALB + AWS WAF | Failure 3: Node hot-spot collapse | Terminates TLS, removes the single-IP bottleneck, health-checks dead targets, and rate-limits abusive bursts before they hit the app. |
| Auto-scaling Node.js fleet | Failure 3: Node.js event loop saturation | Spreads the 500,000 RPS spike across many stateless instances so one callback queue cannot bring down the whole tier. |
| Redis Cluster | Failure 1 and Failure 4: DB pool exhaustion and promo race condition | Caches menu reads to remove most DB lookups and uses atomic promo locking so concurrent redemptions cannot oversell the offer. |
| PgBouncer | Failure 1: PostgreSQL connection pool exhaustion | Converts many short-lived app connections into a small, stable pool of database connections and keeps the DB at its configured limit. |
| PostgreSQL primary + read replicas | Failure 1 and write/read contention | Separates write traffic from read-heavy browsing so browsing does not compete with checkout on the primary. |
| SQS payment queue | Failure 2: Synchronous payment call amplification | Moves the 200-2000 ms gateway wait off the request path so API threads do not sit on database connections while waiting. |
| Payment worker service | Failure 2: Synchronous payment call amplification | Performs the gateway call asynchronously and retries from the queue, so spikes in payment latency do not block the frontend request path. |

## 4. Why this architecture survives the event

The new design removes the three original hard bottlenecks:

- Static content is served at the edge, so the origin NIC no longer becomes the first failure.
- Reads are cached and split across Redis and replicas, so PostgreSQL is no longer the first choke point.
- Payments are asynchronous, so checkout latency no longer pins a database connection for hundreds or thousands of milliseconds.

The design still has finite capacity, but it fails by scaling out, not by collapsing into one dead process.