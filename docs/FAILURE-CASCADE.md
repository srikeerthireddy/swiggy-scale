# SwiftEats Failure Cascade Analysis

## System Under Analysis

The monolith has four hard limits that matter before anything else:

- One Node.js Express process on one CPU with 4 GB RAM.
- One PostgreSQL instance with `max_connections = 100`.
- Synchronous payment calls that hold a database connection while waiting 200-2000 ms.
- No cache, no CDN, no load balancer, and no auto-scaling.

The weakest component is PostgreSQL connection capacity. It is the first fixed ceiling that the burst traffic hits, and it is low enough to fail long before the Node.js process reaches its own CPU ceiling.

## 1. Traffic Simulation Math

### Raw demand

- Notification recipients: 180,000,000
- Click-through rate: 8%
- Users who open the app: 180,000,000 x 0.08 = 14,400,000

### Burst model used for capacity planning

The operational burst is the first 60 seconds after delivery, when the app-open wave is still building. For the failure model below, the conservative peak is 10,000,000 concurrent users in that 60-second window.

- Active users in the spike window: 10,000,000
- API calls per user in the first minute: 3
- Peak RPS = (10,000,000 x 3) / 60
- Peak RPS = 500,000 requests/second

That is 41.7x above the single-process Node.js saturation point and 5,000x above the PostgreSQL connection ceiling.

## 2. Component Capacity Numbers

### Hard limits

- PostgreSQL `max_connections = 100`.
- Node.js event loop saturation: approximately 12,000-15,000 RPS on a single t3.medium before the callback queue backs up.
- Synchronous payment calls: 200-2000 ms hold time per database connection.
- Node.js heap limit: 4 GB on a t3.medium, with OOM likely around 15,000 queued requests.

### DB pool exhaustion math

Use the observed burst mix from the promo event:

- 70% non-payment requests
- 30% payment requests
- Average non-payment query time = 0.02 s
- Average payment hold time = 0.8 s

Connections held at any moment = (% non-payment RPS x query_time_s) + (% payment RPS x payment_hold_time_s)

Connections held = (0.70 x RPS x 0.02) + (0.30 x RPS x 0.8)

Connections held = 0.014RPS + 0.24RPS

Connections held = 0.254RPS

Pool exhaustion when connections held = 100:

100 = 0.254RPS

RPS = 100 / 0.254 = 393.7

The PostgreSQL pool exhausts at approximately 394 RPS.

If payment hold time spikes to the top of the observed range, the threshold is even lower:

- 30% payment mix x 2.0 s hold time + 70% x 0.02 s query time
- Connections held = 0.614RPS
- 100 / 0.614 = 162.9 RPS

That is why synchronous payments are an amplifier, not just a latency bug.

## 3. The Cascade

### Failure 1: PostgreSQL connection pool exhaustion

- Severity: CRITICAL
- Trigger: ~394 RPS mixed traffic, or as low as ~163 RPS if payment wait time drifts toward 2 seconds
- User-visible failure: 500s, 502s, and timeouts on login, restaurant browsing, checkout, and order tracking
- What breaks next: Node requests pile up waiting for a free connection, and the event loop starts buffering work instead of serving users

### Failure 2: Synchronous payment call amplification

- Severity: HIGH
- Trigger: ~245 RPS of checkout-heavy traffic, where payment requests become 50% of the mix and hold a connection for 0.8 seconds
- User-visible failure: order placement hangs at the payment step, then fails after long retries or gateway timeouts
- What breaks next: database connections stay pinned while the gateway call is open, so the effective DB capacity collapses even faster than the raw request rate suggests

### Failure 3: Node.js event loop saturation

- Severity: CRITICAL
- Trigger: approximately 12,000 RPS on one t3.medium
- User-visible failure: response time climbs from tens of milliseconds to multiple seconds; eventually requests time out and the process stops accepting work
- What breaks next: the pending request queue expands, memory rises, GC pauses get longer, and the single process moves toward OOM

### Failure 4: Promo code race condition

- Severity: HIGH
- Trigger: about 20 RPS of concurrent promo-redemption requests, because the code checks then updates in separate statements
- User-visible failure: users see "promo applied" even after the budget is already gone, while later users see inconsistent validation errors
- What breaks next: oversubscribed discounts create a financial loss spike and extra write load on PostgreSQL, which further slows the already-exhausted pool

### Failure 5: Static asset NIC saturation (no CDN)

- Severity: HIGH
- Trigger: about 125 image requests/second if the average response is 1 MB and the origin NIC is 1 Gbps
- User-visible failure: restaurant images, app shell assets, and JS bundles stall or fail to load; the app appears blank even before API calls complete
- What breaks next: the same Node.js process that should be serving APIs is now busy shipping static bytes, so API latency increases for every user

## 4. Incident Timeline

- T+0s: Push notification is delivered to the user base and the first app opens begin.
- T+3s: PostgreSQL reaches 100/100 connections; new requests start waiting in the Node.js process.
- T+5s: P99 response time jumps from sub-second to multi-second as the request queue grows.
- T+8s: Checkout traffic begins holding database connections for the full payment round-trip; the pool no longer drains fast enough.
- T+10s: Promo validations race; some users see successful redemptions after the budget is already overdrawn.
- T+12s: The origin NIC starts saturating under image and static-file traffic; API requests are competing with asset transfer.
- T+18s: Memory pressure and queue buildup push the Node.js process toward OOM and restart behavior.
- T+20s: The single instance is effectively unavailable; the load balancer-less IP has nowhere healthy to send traffic.
- T+45m: On-call has enough logs and manual checks to identify the database pool and synchronous payment path as the dominant failure mode.
- T+1h: Emergency mitigation is applied: promo traffic is disabled, payment retries are throttled, and the app is restarted after flushing stuck connections.
- T+2h: Service stabilizes, but only after traffic has fallen and the pool has been manually drained.

## Summary

The first hard limit is PostgreSQL connections, not CPU. The second is the Node.js event loop, which becomes the failure once database waits accumulate. If this architecture sees the stated promo burst again, it does not degrade gracefully; it fails in a predictable sequence.