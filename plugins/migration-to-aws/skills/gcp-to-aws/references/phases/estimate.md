# Phase 4: Estimate AWS Costs

## Step 1: Check Pricing Availability

Call MCP `awspricing` with `get_pricing_service_codes()`:

- **If success**: Use live AWS pricing for all estimates
- **If timeout or error**: Log warning, fall back to `shared/pricing-fallback.json`

## Step 2: Extract Services from Design

Read `aws-design.json`. Build list of unique AWS services mapped:

- Fargate
- RDS (Aurora Serverless v2)
- S3
- ALB
- Lambda
- ECS
- VPC / NAT Gateway
- etc.

## Step 3: Query Pricing

For each service:

1. Determine usage scenario from `aws-design.json` config (e.g., Fargate: 0.5 CPU, 1 GB memory, assumed 24/7)
2. Call awspricing with appropriate filters:
   - Region: extracted from design (default `us-east-1`)
   - Service attributes: CPU, memory, storage, etc.
3. Calculate monthly cost per service

Handle 3 cost tiers (to show optimization range):

- **Premium**: Latest generation, highest availability (e.g., db.r6g, Fargate Spot disabled)
- **Balanced**: Standard generation, typical setup (e.g., db.t4g, Fargate on-demand)
- **Optimized**: Cost-minimized (e.g., db.t4g with reserved, Fargate Spot 70%)

## Step 4: Calculate Summary

**Monthly operational cost:**

```
Sum of all service monthly costs (Balanced tier)
```

**One-time migration cost:**

```
Development hours: X hours × $150/hour (assume 10-15 weeks)
Data transfer: Y GB × $0.02/GB (egress from GCP)
```

**ROI (vs GCP):**

```
Monthly GCP cost: (estimate from inventory or ask user)
AWS monthly cost: (from Step 3)
Monthly savings: GCP - AWS
Payback period: One-time cost / Monthly savings (in months)
5-year savings: (Monthly savings × 60) - One-time cost
```

## Step 5: Write Estimation Output

Write `estimation.json`:

```json
{
  "monthly_costs": {
    "premium": { "total": 5000, "breakdown": {"Fargate": 1200, "RDS": 2500, ...} },
    "balanced": { "total": 3500, "breakdown": {"Fargate": 800, "RDS": 1800, ...} },
    "optimized": { "total": 2200, "breakdown": {"Fargate": 400, "RDS": 1200, ...} }
  },
  "one_time_costs": {
    "dev_hours": "150 hours @ $150/hr = $22,500",
    "data_transfer": "500 GB @ $0.02/GB = $10,000",
    "total": 32500
  },
  "roi": {
    "assumed_gcp_monthly": 4500,
    "aws_monthly_balanced": 3500,
    "monthly_savings": 1000,
    "payback_months": 32.5,
    "five_year_savings": 28500
  },
  "assumptions": [
    "24/7 workload usage",
    "No Reserved Instances",
    "No Spot instances (Balanced tier)",
    "Region: us-east-1",
    "GCP monthly cost: user estimate"
  ]
}
```

Write `estimation-report.md`:

```
# AWS Cost Estimation

## Monthly Operating Costs

### Balanced Tier (Recommended)
- Fargate: $800
- RDS Aurora: $1,800
- S3: $500
- ALB: $200
- **Total: $3,500/month**

### Comparison Tiers
- Premium: $5,000/month
- Optimized: $2,200/month

## One-Time Migration Costs
- Dev: 150 hours @ $150/hr = $22,500
- Data transfer: $10,000
- **Total: $32,500**

## ROI Analysis
- Assumed GCP cost: $4,500/month
- AWS Balanced: $3,500/month
- **Savings: $1,000/month**
- **Payback: 32.5 months**
- **5-year savings: $28,500**
```

## Step 6: Update Phase Status

Update `.phase-status.json`:

```json
{
  "phase": "execute",
  "status": "completed",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

Output to user: "Cost estimation complete. Balanced tier: $X/month, Payback: X months. Proceeding to Phase 5: Execution Plan."
