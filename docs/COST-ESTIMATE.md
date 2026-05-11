# SwiftEats AWS Cost Estimate

## Assumptions

- Month length: 720 hours.
- Pricing uses the on-demand rates referenced in AWS pricing pages.
- Baseline traffic is steady-state daily usage.
- Peak traffic is a 4-hour World Cup Final window.

## Baseline Monthly Cost

| Service | Instance / Unit | Formula | Monthly Cost |
|---|---|---:|---:|
| EC2 app fleet | t3.medium x 4 | $0.0416/hr x 720 hr x 4 | $119.81 |
| RDS primary | db.r6g.large x 1 | $0.182/hr x 720 hr x 1 | $131.04 |
| RDS read replicas | db.r6g.large x 2 | $0.182/hr x 720 hr x 2 | $262.08 |
| ElastiCache Redis | cache.r6g.large x 3 | $0.166/hr x 720 hr x 3 | $358.56 |
| ALB base | Application Load Balancer | $0.0225/hr x 720 hr | $16.20 |
| ALB LCU usage | LCU hours | $0.008/hr x 5,000 LCU-hr | $40.00 |
| CloudFront transfer | 10 TB/month | $0.0085/GB x 10,000 GB | $85.00 |
| CloudFront requests | 50,000,000 HTTPS requests | $0.0075 / 10,000 requests x 50,000,000 | $37.50 |
| SQS requests | 30,000,000 requests/month | $0.40 / 1,000,000 x 30,000,000 | $12.00 |

### Baseline total

$119.81 + $131.04 + $262.08 + $358.56 + $16.20 + $40.00 + $85.00 + $37.50 + $12.00 = $1,062.19/month

## Peak Event Cost

The peak window is handled by pre-scaling the app tier and absorbing the surge in CloudFront egress.

| Service | Instance / Unit | Formula | Extra Cost for 4 Hours |
|---|---|---:|---:|
| Extra EC2 app instances | t3.2xlarge x 16 additional instances | $0.3328/hr x 4 hr x 16 | $21.30 |
| CloudFront surge transfer | 50 TB extra | $0.0085/GB x 50,000 GB | $425.00 |

### Peak extra total

$21.30 + $425.00 = $446.30 for the 4-hour peak window

## Business Justification

Baseline infrastructure costs about $1,062.19/month. The World Cup Final peak adds only $446.30 for four hours of extra capacity. That spend is small compared with a 45-minute outage at ₹4.2 crore per minute:

- 45 minutes x ₹4.2 crore/minute = ₹189 crore in lost orders

The infrastructure cost is not the risk; the outage is. The ROI is obvious: spending roughly one thousand dollars per month to avoid a nine-figure rupee incident is a trivial trade.