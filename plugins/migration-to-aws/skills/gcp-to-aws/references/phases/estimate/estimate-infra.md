# Estimate Phase: Infrastructure Cost Analysis

> Loaded by estimate.md when aws-design.json exists.

**Execute ALL steps in order. Do not skip or optimize.**

## Pricing Mode

The parent `estimate.md` selects the pricing mode before loading this file. Use whichever mode was selected:

- **Live (MCP Server)**: Real-time pricing from AWS API, accuracy ±5-10%
- **Fallback (Cached)**: Uses `pricing-fallback.json`, accuracy ±15-25%

The formulas and methodology below are identical for both modes. The only difference is the price source. Track the source for each service:

- `pricing_source: "live"` — Price came from MCP API
- `pricing_source: "fallback"` — Price came from `shared/pricing-fallback.json`
- `pricing_source: "estimated"` — Service not in fallback data; conservative estimate used

## Step 0: Validate Design Output

Before pricing queries, validate `aws-design.json`:

1. **File exists**: If missing, **STOP**. Output: "Phase 3 (Design) not completed. Run Phase 3 first."
2. **Valid JSON**: If parse fails, **STOP**. Output: "Design file corrupted (invalid JSON). Re-run Phase 3."
3. **Required fields**:
   - `clusters` array is not empty: If empty, **STOP**. Output: "No clusters in design. Re-run Phase 3."
   - Each cluster has `resources` array: If missing, **STOP**. Output: "Cluster [id] missing resources. Re-run Phase 3."
   - Each resource has `aws_service` field: If missing, **STOP**. Output: "Resource [address] missing aws_service. Re-run Phase 3."
   - Each resource has `aws_config` field: If missing, **STOP**. Output: "Resource [address] missing aws_config. Re-run Phase 3."

If all validations pass, proceed to Part 1.

## Part 1: Calculate Current GCP Costs

Determine your current GCP monthly infrastructure costs from the best available source.

### Source Priority

1. **`billing-profile.json` (preferred)** — If this file exists, use actual billing data as the GCP baseline. This is real spend data and provides the highest confidence baseline.
2. **`gcp-resource-inventory.json` (fallback)** — If no billing data is available, estimate costs from discovered resource configurations.
3. **`preferences.json` `gcp_monthly_spend`** — If the user provided their monthly GCP spend during clarification.
4. **Conservative default** — If none of the above: use `AWS monthly balanced * 1.25` (assume 25% GCP premium over AWS balanced estimate).

### If billing-profile.json exists (preferred)

Use the actual spend data as the authoritative GCP cost baseline:

```
FROM billing-profile.json:
  total_monthly_spend: $[amount]

Per-service breakdown:
  [service 1]: $[monthly_cost]
  [service 2]: $[monthly_cost]
  ...

Source: Actual billing data (high confidence)
```

### If no billing data (estimate from inventory)

Calculate from discovered services in `gcp-resource-inventory.json`:

```
FROM gcp-resource-inventory.json resources:

Compute (Cloud Run):
  - [N] instances x [vCPU] vCPU x [mem]GB x 730 hours/month
  - Estimated cost: $[calc]/month

Database (Cloud SQL):
  - [instance_tier] instance x 730 hours
  - Storage: [size]GB x $0.23/GB
  - Estimated cost: $[calc]/month

Storage (Cloud Storage):
  - [size] GB x $0.020/GB (Standard)
  - [size] GB x $0.004/GB (Coldline archive)
  - Estimated cost: $[calc]/month

Networking (Load Balancer):
  - HTTP(S) Load Balancer
  - Estimated cost: $[calc]/month

Total Current GCP Cost: $[total]/month
```

**Note:** Inventory-based estimates carry wider confidence ranges (±20-30%) compared to billing-based baselines (±5%).

## Part 2: Calculate Projected AWS Costs

Using the architecture from `aws-design.json`, calculate costs for 3 tiers. Track `pricing_source` per service.

Handle 3 cost tiers (to show optimization range):

- **Premium**: Latest generation, highest availability (e.g., db.r6g, Fargate Spot disabled)
- **Balanced**: Standard generation, typical setup (e.g., db.t4g, Fargate on-demand)
- **Optimized**: Cost-minimized (e.g., db.t4g with reserved, Fargate Spot 70%)

### A. Compute Costs

**Fargate (mapped from Cloud Run):**

```
Configuration (from aws-design.json):
  - [vCPU] vCPU + [mem]GB memory
  - 730 hours/month (24/7)
  - [N] instances (high availability)

Calculation:
  vCPU cost:    [vCPU] vCPU x $0.04048/hour x 730 hours = $[calc]
  Memory cost:  [mem] GB x $0.004445/hour x 730 hours = $[calc]
  Per instance: $[calc]/month
  [N] instances: $[total]/month

AWS Fargate Total: $[total]/month
```

#### Alternative: Lambda (if stateless services)

```
Configuration:
  - [N]M requests/month
  - [mem]MB memory
  - [duration] second average

Calculation:
  Request cost: [N]M x $0.0000002 = $[calc]
  Compute cost: [N]M x [duration]s x [mem_GB]GB = [X]K GB-seconds
              [X]K x $0.0000166667 = $[calc]

AWS Lambda Total: $[total]/month
```

### B. Database Costs

**Aurora PostgreSQL Multi-AZ (mapped from Cloud SQL):**

```
Configuration (from aws-design.json):
  - [instance_type] x [N] (primary + standby for HA)
  - [size]GB storage
  - Multi-AZ automatic backups

Calculation:
  Instance cost:  [instance_type] = $[rate]/hour x [N] instances x 730 hours
                                  = $[calc]
  Storage cost:   [size]GB x $0.1/GB/month = $[calc]
  I/O cost:       ~$0.2/million requests x [N]M = $[calc]

AWS Aurora Total: $[total]/month
```

#### Alternative: RDS Multi-AZ (if simpler database)

```
Configuration:
  - [instance_type] x [N] (Multi-AZ)
  - [size]GB storage

Calculation:
  Instance cost:  $[rate]/hour x [N] x 730 = $[calc]
  Storage cost:   [size]GB x $0.23/GB = $[calc]
  Backup cost:    ~$[calc]

AWS RDS Total: $[total]/month
```

### C. Storage Costs

**S3 with Tiering (mapped from Cloud Storage):**

```
Configuration (from aws-design.json):
  - Active data: [size] GB in S3 Standard
  - Archive data: [size] GB in S3 Intelligent-Tiering

Calculation:
  Standard:       [size] GB x $0.023/GB = $[calc]
  Intelligent:    [size] GB x $0.0125/GB (average) = $[calc]
  Requests:       [N]K puts x $0.005/1K = $[calc]
                  [N]M gets x $0.0004/1K = $[calc]

AWS S3 Total: $[total]/month
```

### D. Networking Costs

**Application Load Balancer:**

```
Configuration (from aws-design.json):
  - [N] ALB
  - ~[N]M requests/month

Calculation:
  Hourly charge:   $0.0225/hour x 730 hours = $16.43
  LCU charge:      [N]M requests = [X] LCUs x $0.006/hour x 730 = $[calc]

AWS ALB Total: $[total]/month
```

**NAT Gateway (if VPC with private subnets):**

```
Configuration:
  - [N] NAT Gateway(s)
  - [size] GB data processed/month

Calculation:
  NAT cost:   $32 x [N] = $[calc]
  Data cost:  [size] GB x $0.045 = $[calc]

AWS NAT Gateway Total: $[total]/month
```

### E. Supporting Services

**Secrets Manager:**

```
Configuration:
  - [N] secrets (DB password, API keys, etc.)
  - [N] secret retrievals/month

Calculation:
  Secrets:         [N] x $0.40/month = $[calc]
  Retrievals:      [N] x $0.05/million = negligible

AWS Secrets Manager: $[total]/month
```

**CloudWatch (Logs + Metrics):**

```
Configuration:
  - [size]GB of logs/month
  - [N] custom metrics

Calculation:
  Log ingestion:   [size]GB x $0.50 = $[calc]
  Log storage:     [size]GB x $0.03/GB = $[calc]
  Metrics:         [N] x $0.30 = $[calc]

AWS CloudWatch: $[total]/month
```

## Part 3: Total Cost Comparison

### AWS Architecture Total

```
Service                          Monthly Cost
-----------------------------------------------
Fargate (compute)               $[calc]
Aurora/RDS (database)           $[calc]
S3 (storage)                    $[calc]
ALB (networking)                $[calc]
NAT Gateway                     $[calc]
Secrets Manager                 $[calc]
CloudWatch                      $[calc]
-----------------------------------------------
TOTAL AWS (Base):               $[total]/month

------- COST OPTIMIZATION OPTIONS -------

Switch Fargate -> Lambda:       -$[calc] -> $[adjusted]
Switch Aurora -> RDS:           -$[calc] -> $[adjusted]
All optimizations:              $[lowest_total]/month
```

## Part 4: Current vs Projected Cost

### Side-by-Side Comparison

```
CURRENT GCP          | $[total]/month (Cloud Run, Cloud SQL, Storage, LB, Other)

PROJECTED AWS Options:
                     | Compute   | Database  | Storage  | Net+Other | TOTAL     | vs GCP
Option A (Premium)   | Fargate   | Aurora    | S3       | ALB+CW    | $[total]  | [+/-]$[diff]
Option B (Balanced)  | Lambda    | RDS       | S3       | ALB+CW    | $[total]  | [+/-]$[diff]
Option C (Optimized) | Lambda    | RDS       | S3-IA    | ALB+CW    | $[total]  | [+/-]$[diff]
```

## Part 5: One-Time Migration Costs

In addition to monthly costs, migration requires one-time effort:

### Development & Testing

```
Component              | Hours | Rate  | Cost
-----------------------+-------+-------+----------
Architecture design    | 16    | $150  | $2,400
Code migration/updates | 40    | $150  | $6,000
Testing & validation   | 24    | $150  | $3,600
Data migration         | 16    | $150  | $2,400
Deployment & cutover   | 16    | $150  | $2,400
-----------------------+-------+-------+----------
TOTAL Development      |       |       | $16,800
```

Adjust based on complexity from `preferences.json`:

- `complexity_level = "low"` -> Reduce hours by 30%
- `complexity_level = "medium"` -> Use baseline hours
- `complexity_level = "high"` -> Increase hours by 50%

### Infrastructure Setup

```
Component                   | Cost
----------------------------+---------
AWS account setup           | $0 (free)
Data transfer (GCP -> AWS)  | $500-1,000
Initial testing             | $50-200
----------------------------+---------
TOTAL Infrastructure        | ~$1,000
```

### Training & Documentation

```
Component              | Cost
-----------------------+---------
Team AWS training      | $2,000
Documentation          | $1,000
Runbooks & guides      | $1,000
-----------------------+---------
TOTAL Training         | $4,000
```

**Total One-Time Migration Cost: ~$21,800** (Adjust total based on complexity scaling above.)

## Part 6: ROI Analysis

### Payback Period Calculation

### Scenario: Balanced Option

```
Monthly Savings:        $[GCP total] - $[AWS balanced] = $[savings or cost]/month
One-Time Cost:          $[one_time_total]
Payback Period:         $[one_time_total] / $[monthly savings] = [N] months

If AWS is cheaper:
  Result: Payback in [N] months

If AWS is more expensive:
  No payback from cloud costs alone -- justify with operational savings
```

### Scenario: Cost-Optimized Option

```
Monthly Savings:        $[GCP total] - $[AWS optimized] = $[savings]/month
One-Time Cost:          $[one_time_total]
Payback Period:         $[one_time_total] / $[monthly savings] = [N] months
```

### Non-Cost Benefits

Present these alongside the financial analysis:

- **Operational efficiency** — AWS managed services (Fargate, RDS) require less operational overhead
- **Better global reach** — More AWS regions available for expansion
- **Service breadth** — Access to broader AWS service catalog
- **Enterprise integration** — Better integration with enterprise tools
- **Vendor diversification** — Reduces single-vendor risk
- **Scaling flexibility** — Auto-scaling, spot instances, savings plans

## Part 7: Hidden Savings

While direct cloud costs may or may not justify migration, operational savings often do:

```
GCP Operational Overhead:     [N] engineers managing GCP infra  = $[cost]/year
AWS with Managed Services:    [N-1] engineers managing AWS      = $[cost]/year
Operational Savings:          $[diff]/year = $[monthly]/month

Payback (with ops savings):   $[one_time_total] / ($[cloud_savings] + $[ops_savings]) = [N] months

Additional Business Value:
  - Better global reach ([N] AWS regions vs [N] GCP regions)
  - More service options for future workloads
  - Better integration with enterprise tools
  - Reduced vendor lock-in risk
```

## Part 8: Cost Optimization Opportunities

If proceeding with migration, here are ways to save:

### 1. Use Reserved Instances (40-60% Savings)

```
Before (On-Demand):
  Fargate: $[cost]/month x 12 months = $[annual]/year
  RDS:     $[cost]/month x 12 months = $[annual]/year
  Total:   $[annual_total]/year

After (1-Year Reserved):
  Fargate: $[cost] x 40% discount = $[calc]/month = $[annual]/year
  RDS:     $[cost] x 40% discount = $[calc]/month = $[annual]/year
  Total:   $[annual_total]/year (save $[diff] = 40%)
```

### 2. Move Data to S3-IA Earlier (38-50% Storage Savings)

```
Before (All S3 Standard):
  [size] GB Standard  = $[cost]/month
  Total: $[total]/month

After (Tiering with IA):
  [size] GB Standard  = $[cost]/month
  [size] GB S3-IA     = $[cost]/month
  Total: $[total]/month (save $[diff]/month = [N]%)
```

### 3. Use Spot Instances for Batch (60-90% Savings)

If you have batch jobs:

```
On-Demand EC2: $[rate]/hour
Spot EC2:      $[rate]/hour (70% discount)

Monthly batch job ([N] hours):
  On-Demand: [N] x $[rate] = $[cost]
  Spot:      [N] x $[rate] = $[cost] (save $[diff]/month)
```

### 4. Enable Compute Savings Plans (20-50% Savings)

AWS Savings Plans cover Fargate and Lambda:

```
On-Demand: $[cost]/month
Savings Plan (1-year): $[cost] x 25% discount = $[calc]/month (save $[diff])
```

## Part 9: Financial Summary

### Executive Summary

```
                              Direct Cloud    With Ops Savings    With All Optimizations
Current GCP:                  $[total]/mo     $[total]/mo         $[total]/mo
Projected AWS:                $[total]/mo     $[total]/mo         $[total]/mo
Monthly Difference:           [+/-]$[diff]    [+/-]$[diff]        [+/-]$[diff]
Annual Benefit:               ~$[annual]      ~$[annual]          ~$[annual]
Payback Period:               ~[N] months     ~[N] months         ~[N] months
5-Year Savings:               ~$[total]       ~$[total]           ~$[total]
```

## Part 10: Recommendation

### When to Migrate

**MIGRATE IF:**

- You value operational efficiency (fast payback with ops savings)
- Your team wants to use AWS-specific services
- You have batch workloads (use Spot for 60-90% savings)
- Long-term AWS strategy aligns with business
- Your infrastructure is growing (leverage AWS scale)

**STAY ON GCP IF:**

- Cloud costs are your only metric and AWS is more expensive
- Team is highly experienced with GCP
- Lowest possible spend is paramount
- No need for AWS-specific services

### Recommended Path

### 1. Migrate with Optimizations (Best ROI)

```
Path: [optimized service choices based on design]
Monthly Cost: $[total]
Operational Savings: $[monthly]
5-Year TCO: $[savings] savings
Timeline: [N] weeks
```

### 2. Phased Migration (Lower Risk)

```
Path: Migrate cluster-by-cluster per design evaluation order
Risk: Lower (validate each cluster before proceeding)
Decision: Proceed to next cluster after validation
```

### 3. Stay on GCP (Lowest Cost)

```
Path: Keep current GCP infrastructure
Monthly Cost: $[total]
Risk: Higher cost if scale increases
Decision: Only if costs are non-negotiable
```

## Output

Write `estimation-infra.json` and `estimation-infra-report.md`.

### estimation-infra.json schema

```json
{
  "phase": "estimate",
  "design_source": "infrastructure",
  "timestamp": "2026-02-24T14:00:00Z",
  "pricing_source": {
    "status": "live|fallback",
    "message": "Using live AWS pricing API|Using cached rates from 2026-02-24 (±15-25% accuracy)",
    "fallback_staleness": {
      "last_updated": "2026-02-24",
      "days_old": 3,
      "is_stale": false,
      "staleness_warning": null
    },
    "services_by_source": {
      "live": ["Fargate", "RDS Aurora", "S3", "ALB"],
      "fallback": ["NAT Gateway"],
      "estimated": []
    },
    "services_with_missing_fallback": []
  },
  "accuracy_confidence": "±5-10%|±15-25%",

  "current_costs": {
    "source": "billing_data|inventory_estimate|preferences|default",
    "gcp_monthly": 300,
    "gcp_annual": 3600,
    "baseline_note": "From billing-profile.json actual spend data",
    "breakdown": { "compute": 75, "database": 50, "storage": 40, "networking": 20, "other": 15 }
  },

  "projected_costs": {
    "aws_monthly_premium": 1003,
    "aws_monthly_balanced": 265,
    "aws_monthly_optimized": 194,
    "aws_annual_optimized": 2328,
    "breakdown": {
      "compute": {
        "service": "Fargate",
        "monthly": 71,
        "alternative": { "service": "Lambda", "monthly": 9, "savings": 62 }
      },
      "database": {
        "service": "Aurora PostgreSQL",
        "monthly": 269,
        "alternative": { "service": "RDS PostgreSQL", "monthly": 75, "savings": 194 }
      },
      "storage": {
        "service": "S3 Standard + Intelligent-Tiering",
        "monthly": 86,
        "alternative": { "service": "S3-IA", "monthly": 65, "savings": 21 }
      },
      "networking": { "service": "ALB + NAT Gateway", "monthly": 53 },
      "supporting": { "secrets_manager": 1.20, "cloudwatch": 35.30 }
    }
  },

  "cost_comparison": {
    "gcp_monthly_baseline": 300,
    "option_a_premium": {
      "aws_monthly": 1003,
      "monthly_difference": 703,
      "annual_difference": 8436,
      "percent_change": "+234%"
    },
    "option_b_balanced": {
      "aws_monthly": 265,
      "monthly_difference": -35,
      "annual_difference": -420,
      "percent_change": "-12%"
    },
    "option_c_optimized": {
      "aws_monthly": 194,
      "monthly_difference": -106,
      "annual_difference": -1272,
      "percent_change": "-35%"
    }
  },

  "one_time_costs": {
    "development": 16800,
    "infrastructure": 1000,
    "training": 4000,
    "total_one_time": 21800,
    "complexity_adjustment": "none|reduced_30|increased_50",
    "complexity_factors": [
      "medium complexity from preferences.json",
      "standard database migration",
      "container-based compute migration"
    ]
  },

  "roi_analysis": {
    "direct_cloud_savings": {
      "monthly": 106,
      "annual": 1272,
      "payback_period_months": 205,
      "note": "Based on optimized option vs GCP baseline"
    },
    "with_operational_efficiency": {
      "operational_savings_monthly": 8333,
      "cloud_savings_monthly": 106,
      "total_monthly_benefit": 8439,
      "annual_benefit": 101268,
      "payback_period_months": 2.6,
      "note": "Includes reduction in operational overhead from managed services"
    },
    "five_year_total_savings": 475000,
    "non_cost_benefits": [
      "Operational efficiency (fewer engineers needed for managed services)",
      "Better global reach (more AWS regions)",
      "Broader service catalog for future workloads",
      "Better enterprise tool integration",
      "Vendor diversification (reduce single-vendor risk)",
      "Auto-scaling, spot instances, savings plans flexibility"
    ]
  },

  "optimization_opportunities": [
    {
      "opportunity": "Reserved Instances",
      "target_services": ["Fargate", "RDS"],
      "savings_monthly": 58,
      "savings_percent": "40%",
      "commitment": "1-year",
      "implementation_effort": "low",
      "description": "Commit to 1-year reserved capacity for predictable workloads"
    },
    {
      "opportunity": "S3 Infrequent Access",
      "target_services": ["S3"],
      "savings_monthly": 52,
      "savings_percent": "38%",
      "commitment": "none",
      "implementation_effort": "low",
      "description": "Move infrequently accessed data to S3-IA storage class"
    },
    {
      "opportunity": "Spot Instances for Batch",
      "target_services": ["EC2"],
      "savings_monthly": 6,
      "savings_percent": "70%",
      "commitment": "none",
      "implementation_effort": "medium",
      "description": "Use Spot instances for fault-tolerant batch processing jobs"
    },
    {
      "opportunity": "Compute Savings Plans",
      "target_services": ["Fargate", "Lambda"],
      "savings_monthly": 20,
      "savings_percent": "25%",
      "commitment": "1-year",
      "implementation_effort": "low",
      "description": "AWS Savings Plans covering Fargate and Lambda usage"
    }
  ],

  "financial_summary": {
    "current_gcp_monthly": 300,
    "projected_aws_balanced_monthly": 265,
    "projected_aws_optimized_monthly": 194,
    "one_time_cost": 21800,
    "payback_cloud_only_months": 205,
    "payback_with_ops_months": 2.6,
    "five_year_savings_with_ops": 475000,
    "recommendation": "Migrate with optimizations for best ROI"
  },

  "recommendation": {
    "path": "Full Infrastructure with Optimizations",
    "roi_justification": "2.6 month payback with operational efficiency; $475K 5-year savings",
    "confidence": "high",
    "next_steps": [
      "Review financial case with stakeholders",
      "Confirm service tier selections (Aurora vs RDS, Fargate vs Lambda)",
      "Get approval to proceed to Execute phase",
      "Schedule migration timeline per cluster evaluation order"
    ]
  }
}
```

### estimation-infra-report.md

Present the financial analysis as a readable report covering:

- Monthly operating costs (balanced tier recommended, with premium and optimized for comparison)
- Per-service cost breakdown
- One-time migration costs
- ROI analysis with payback period
- Cost optimization opportunities
- Executive summary and recommendation

## Output Validation Checklist

### estimation-infra.json

- `design_source` is `"infrastructure"`
- `pricing_source.status` is `"live"` or `"fallback"`
- `accuracy_confidence` matches the pricing mode (±5-10% for live, ±15-25% for fallback)
- `current_costs.source` is `"billing_data"` if `billing-profile.json` was used, `"inventory_estimate"`, `"preferences"`, or `"default"` otherwise
- `current_costs.gcp_monthly` matches billing-profile.json total (if used) or is a reasonable estimate
- `projected_costs` has all three tiers (premium, balanced, optimized)
- `projected_costs.breakdown` covers compute, database, storage, networking, and supporting services
- Every service in `aws-design.json` is represented in the cost breakdown
- `cost_comparison` shows all three options with monthly and annual differences
- `one_time_costs` adjusted for complexity from `preferences.json`
- `one_time_costs.total_one_time` equals the sum of development + infrastructure + training
- `roi_analysis` includes both cloud-only and operational-efficiency payback periods
- `roi_analysis` is honest — if migration increases cost, say so and justify with non-cost benefits
- `optimization_opportunities` only includes strategies relevant to the designed architecture
- `financial_summary` provides a clear executive-level view
- `recommendation.next_steps` includes actionable items
- No references to AI-specific costs (those belong in `estimate-ai.md`)
- No references to billing-only estimates (those belong in `estimate-billing.md`)
- All cost values are numbers, not strings
- Output is valid JSON

### estimation-infra-report.md

- Balanced tier section shows per-service breakdown with total
- Comparison tiers section lists premium and optimized totals
- One-time costs section present
- ROI analysis section present with payback period
- Cost optimization section present
- Executive summary present

## Execute Phase Integration

The Execute phase (`execute.md`) uses `estimation-infra.json` as follows:

1. **`projected_costs.breakdown`** — Budget allocation per cluster migration phase
2. **`one_time_costs`** — Total budget for the migration effort and resource planning
3. **`optimization_opportunities`** — Which optimizations to implement and when (some during initial migration, some post-migration)
4. **`cost_comparison`** — Set cost monitoring targets and alerts for each migrated cluster
5. **`recommendation.next_steps`** — Prerequisites for starting execution
6. **`financial_summary.payback_with_ops_months`** — Timeline pressure for realizing ROI

The execution plan should reference the cost estimates to set per-cluster cost monitoring thresholds and validate that actual AWS spend aligns with projections after each cluster migration.

## Key Metrics Summary

| Metric                    | Value          | Note                                            |
| ------------------------- | -------------- | ----------------------------------------------- |
| Current GCP Cost          | $[total]/month | From billing-profile.json or inventory estimate |
| AWS Cost (Premium)        | $[total]/month | Full-featured, no optimizations                 |
| AWS Cost (Balanced)       | $[total]/month | Balanced performance and cost                   |
| AWS Cost (Optimized)      | $[total]/month | Maximum savings with trade-offs                 |
| One-Time Migration        | $[total]       | Development + infrastructure + training         |
| Payback Period (Cloud)    | [N] months     | Based on cloud cost difference alone            |
| Payback Period (With Ops) | [N] months     | Including operational efficiency gains          |
| 5-Year Savings            | $[total]       | With optimizations and operational efficiency   |
