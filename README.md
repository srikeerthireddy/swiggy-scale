# Swiggy is Down - Scale Simulation & Incident Architecture

This repository models what happens when a Swiggy-like backend faces a World Cup Final promo burst and then redesigns the system so it can survive the load.

## Scenario

India vs Pakistan World Cup Final. 8 PM IST. A 50% off promo is sent to 180 million users while the backend is still a single Node.js process talking to one PostgreSQL database. The question is not whether it gets slow; the question is which component fails first and what it takes to make the next version survive.

## Document Summary
| Document | What it contains |
|---|---|
| [FAILURE-CASCADE.md](docs/FAILURE-CASCADE.md) | Traffic math, component capacity limits, failure triggers, and the incident timeline. |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | The current monolith, the redesigned multi-tier architecture, and the justification for each new component. |
| [COST-ESTIMATE.md](docs/COST-ESTIMATE.md) | Baseline and peak AWS cost calculations using concrete instance types and formulas. |
| [RUNBOOK.md](docs/RUNBOOK.md) | A 5-step 2 AM incident runbook with alarms, triage, response, rollback, and postmortem template. |

## Key Findings

- PostgreSQL connection capacity is the first hard stop: the pool exhausts at about 394 RPS, while the promo burst targets 500,000 RPS.
- If payment hold time drifts toward 2 seconds, the database ceiling drops to about 163 RPS.
- The single Node.js process saturates around 12,000 RPS, so even after caching improves the database path, the app tier still needs horizontal scale.
- Serving static assets from the origin is not a side issue: a 1 Gbps NIC can saturate at roughly 125 image requests per second when assets are large.
- A 45-minute outage at ₹4.2 crore per minute costs ₹189 crore, which dwarfs the monthly infrastructure spend.

## Architecture Overview

The redesigned system moves static content to CloudFront, puts an ALB and WAF in front of a stateless Node.js fleet, and shifts hot reads into Redis and PostgreSQL replicas. Synchronous payment calls are removed from the request path by pushing them onto SQS and processing them in worker services, while PgBouncer keeps the database at its configured connection limit.

## Tech Stack Context

This analysis covers Node.js, PostgreSQL, Redis, and AWS infrastructure primitives such as CloudFront, ALB, RDS, ElastiCache, SQS, and Auto Scaling Groups.