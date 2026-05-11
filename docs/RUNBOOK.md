# SwiftEats 2 AM Incident Runbook

Use this document in order. Do not skip steps. If the first symptom does not match the first root cause, move to the next branch exactly as written.

## STEP 1 - DETECT

These CloudWatch alarms must exist before launch.

| Metric | Threshold | Alert Type | Meaning |
|---|---:|---|---|
| `AWS/ApplicationELB HTTPCode_ELB_5XX_Count` | > 5% of requests for 2 minutes | Critical | The load balancer is serving errors or no healthy targets are left. |
| `AWS/ApplicationELB TargetResponseTime` p95 | > 1.5 seconds for 5 minutes | Warning | Users are feeling latency before outright failure. |
| `AWS/RDS DatabaseConnections` | > 80 | Critical | PostgreSQL is close to the `max_connections = 100` ceiling. |
| `AWS/RDS FreeableMemory` | < 512 MB | Warning | PostgreSQL is under memory pressure and will slow down soon. |
| `AWS/EC2 CPUUtilization` | > 80% for 3 minutes on all API nodes | Critical | The app tier is saturated and needs more instances immediately. |
| `AWS/EC2 StatusCheckFailed` | > 0 | Critical | One or more app nodes are unhealthy and should be replaced. |
| `AWS/SQS ApproximateNumberOfMessagesVisible` | > 10,000 | Warning | Payment backlog is building. |
| `AWS/SQS ApproximateAgeOfOldestMessage` | > 60 seconds | Critical | Payment workers are behind and customers are waiting on async settlement. |
| `Custom/Node EventLoopLagMs` | > 100 ms for 3 minutes | Warning | The Node.js event loop is backing up. |
| `AWS/ElastiCache CacheHitRate` | < 85% | Warning | Redis misses are forcing reads back to PostgreSQL. |

## STEP 2 - TRIAGE

Read top to bottom. The first red signal is the root cause.

```text
1. Is ALB 5xx > 5% AND RDS DatabaseConnections > 80?
   -> Yes: Step 3a, DB pool exhaustion.
   -> No: continue.

2. Are all API nodes at CPU > 80% OR is Node EventLoopLagMs > 100 ms?
   -> Yes: Step 3b, compute saturation.
   -> No: continue.

3. Is SQS ApproximateNumberOfMessagesVisible > 10,000 OR is ApproximateAgeOfOldestMessage > 60s?
   -> Yes: Step 3c, payment queue backup.
   -> No: continue.

4. Is Redis CacheHitRate < 85% OR are Redis evictions > 0?
   -> Yes: Step 3d, cache miss spike.
   -> No: continue.

5. Are static assets failing but API 5xx is normal?
   -> Yes: CloudFront/origin asset path problem. Escalate to Platform on-call.
```

## STEP 3 - RESPOND

### 3a. DB pool exhaustion

Owner: Data on-call first, Platform on-call second. Slack: `#swifteats-db-incident`.

1. Freeze extra load immediately.

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name swifteats-api-asg --desired-capacity 20
```

2. Drain the connection pooler if it is holding stale sessions.

```bash
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Role,Values=swifteats-pgbouncer" \
  --parameters 'commands=["sudo systemctl restart pgbouncer"]'
```

3. Clear idle transactions on PostgreSQL.

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND usename = 'app_user';
```

Success looks like this: `DatabaseConnections` drops below 65 within 5 minutes, and ALB 5xx falls below 1% within 10 minutes.

### 3b. Compute saturation

Owner: Platform on-call. Slack: `#swifteats-platform`.

1. Scale the API fleet immediately.

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name swifteats-api-asg --desired-capacity 20
```

2. If one node is pegged while others are idle, terminate the worst node so the ASG replaces it.

```bash
aws autoscaling terminate-instance-in-auto-scaling-group --instance-id i-xxxxxxxx --should-decrement-desired-capacity false
```

Success looks like this: average CPU drops below 60% and p95 response time falls below 250 ms within 5 minutes.

### 3c. Payment queue backup

Owner: Payments on-call. Slack: `#swifteats-payments`.

1. Scale the payment workers.

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name swifteats-payments-asg --desired-capacity 12
```

2. If the oldest message keeps growing, confirm workers are not crashing on a bad payload.

```bash
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/swifteats-payments-dlq --max-number-of-messages 10
```

Success looks like this: `ApproximateNumberOfMessagesVisible` drops below 5,000 within 10 minutes and `ApproximateAgeOfOldestMessage` drops below 60 seconds within 15 minutes.

### 3d. Redis cache miss spike

Owner: Platform on-call with Data on-call support. Slack: `#swifteats-platform`.

1. Open the ElastiCache console.
2. Select the Redis replication group.
3. Modify the node type to `cache.r6g.xlarge` or add a shard if cluster mode is enabled.
4. Run a cache warmup script from a bastion host or SSM session against the top 1,000 menu and restaurant URLs.

```bash
while read url; do curl -fsS "$url" >/dev/null; done < top-menu-urls.txt
```

Success looks like this: `CacheHitRate` rises above 90% and PostgreSQL read QPS drops within 10 minutes.

## STEP 4 - ROLLBACK

Rollback criteria:

- ALB 5xx stays above 20% for more than 5 minutes.
- A recent deploy happened in the last 2 hours.
- The root cause is application code, not database schema or infrastructure exhaustion.

Rollback command:

```bash
aws ecs update-service \
  --cluster swifteats-prod \
  --service api \
  --task-definition swifteats-api:PREVIOUS_STABLE_VERSION
```

Never roll back the database schema. Roll back application code only.

If the issue is capacity rather than bad code, scale forward instead of rolling back.

## STEP 5 - POSTMORTEM

Use this template exactly. Fill it in even if you were not in the incident.

### Incident Summary

Instructions: Write one paragraph that states when the incident started, what broke, and when service recovered.

### Timeline

Instructions: List every major event in order with timestamps from CloudWatch, ALB, RDS, and SQS.

### Root Cause

Instructions: Name the deepest technical cause, not the symptom. Example: "PgBouncer pool size was too small for the payment hold time."

### Customer Impact

Instructions: State duration, affected user count, and estimated revenue loss.

### What Worked

Instructions: Write the mitigations that reduced the blast radius. Be specific.

### What Failed

Instructions: Write the controls that did not work or were missing.

### Action Items

Instructions: Each item must have an owner, due date, and one sentence describing the fix.

| Action Item | Owner | Due Date | Notes |
|---|---|---|---|
| Example: Increase PgBouncer pool headroom | Data team | YYYY-MM-DD | Prevent the next connection exhaustion event. |
| Example: Add cache warmup for top menus | Platform team | YYYY-MM-DD | Reduce stampedes after cache eviction. |

## Final Check

If a first-year engineer can follow this without asking a question, the runbook is ready. If they need clarification, add the missing step before the next incident.