# Infrastructure Estimate Schema

Schema for `estimation-infra.json`, produced by `estimate-infra.md`.

---

## estimation-infra.json schema

```json
{
  "phase": "estimate",
  "design_source": "infrastructure",
  "timestamp": "2026-02-24T14:00:00Z",
  "pricing_source": {
    "status": "cached|live|fallback",
    "message": "Using cached prices from 2026-03-04 (±5-10% accuracy)|Using live AWS pricing API|Using cached rates from 2026-02-24 (±15-25% accuracy)",
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
      "medium: 6 services with database migration",
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
    "five_year_total_savings": 484540,
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

## Output Validation Checklist

- `design_source` is `"infrastructure"`
- `pricing_source.status` is `"cached"`, `"live"`, or `"fallback"`
- `accuracy_confidence` matches the pricing mode (±5-10% for cached/live, ±15-25% for fallback)
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
