# Design Phase: Billing-Only Service Mapping

> Loaded by `design.md` when `billing-profile.json` exists and `gcp-resource-inventory.json` does NOT exist.

**Execute ALL steps in order. Do not skip or optimize.**

This is the fallback design path when only billing data is available (no Terraform/IaC). Mappings are inferred from billing service names and SKU descriptions — confidence is always `billing_inferred`.

---

## Step 0: Load Inputs

Read `$MIGRATION_DIR/billing-profile.json`. This file contains:

- `services[]` — Each GCP service with monthly cost, SKU breakdown, and AI signals
- `summary` — Total monthly spend and service count

Read `$MIGRATION_DIR/preferences.json` → `design_constraints` (target region, compliance, etc.).

---

## Step 1: Load Billing Services

For each entry in `billing-profile.json` → `services[]`:

1. Extract `gcp_service` (display name, e.g., "Cloud Run")
2. Extract `gcp_service_type` (Terraform-style type, e.g., "google_cloud_run_service")
3. Extract `top_skus[]` for additional context (SKU descriptions hint at specific features)
4. Extract `monthly_cost` for cost context

---

## Step 2: Fast-Path Lookup Only

For each billing service, attempt a fast-path lookup:

1. Look up `gcp_service_type` in `design-refs/fast-path.md` → Direct Mappings table
2. If found: assign AWS service
3. Enrich with SKU hints:
   - If `top_skus` mention "PostgreSQL" → specify "RDS Aurora PostgreSQL"
   - If `top_skus` mention "MySQL" → specify "RDS Aurora MySQL"
   - If `top_skus` mention "CPU Allocation" → indicates compute (Fargate)
   - If `top_skus` mention "Storage" → check if object storage (S3) or block storage (EBS)
4. If not found in fast-path: add to `unknowns[]`

**No rubric evaluation** — without IaC config, there is insufficient data for the 6-criteria rubric. Fast-path only.

---

## Step 3: Flag Unknowns

For each service not found in fast-path:

1. Record in `unknowns[]` with:
   - `gcp_service` — Display name
   - `gcp_service_type` — Resource type
   - `monthly_cost` — How much is spent on this service
   - `reason` — "No IaC configuration available; billing name does not match any fast-path entry"
   - `suggestion` — "Provide Terraform files for accurate mapping, or manually specify the AWS equivalent"

<!-- TODO: Add heuristic mapping for common billing-only services (Cloud Armor → WAF, Cloud Scheduler → EventBridge, etc.) -->

---

## Step 4: Generate Output

**File 1: `aws-design-billing.json`**

Write to `$MIGRATION_DIR/aws-design-billing.json`:

```json
{
  "metadata": {
    "phase": "design",
    "design_source": "billing_only",
    "confidence_note": "All mappings inferred from billing data only — no IaC configuration available. Confidence is billing_inferred for all services.",
    "total_services": 8,
    "mapped_services": 6,
    "unmapped_services": 2,
    "timestamp": "2026-02-26T14:30:00Z"
  },
  "services": [
    {
      "gcp_service": "Cloud Run",
      "gcp_service_type": "google_cloud_run_service",
      "aws_service": "Fargate",
      "aws_config": {
        "region": "us-east-1"
      },
      "monthly_cost": 450.00,
      "confidence": "billing_inferred",
      "rationale": "Fast-path: Cloud Run → Fargate. SKU hints: CPU + Memory allocation.",
      "sku_hints": ["CPU Allocation Time", "Memory Allocation Time"]
    },
    {
      "gcp_service": "Cloud SQL",
      "gcp_service_type": "google_sql_database_instance",
      "aws_service": "RDS Aurora PostgreSQL",
      "aws_config": {
        "region": "us-east-1"
      },
      "monthly_cost": 800.00,
      "confidence": "billing_inferred",
      "rationale": "Fast-path: Cloud SQL → RDS Aurora. SKU hints: PostgreSQL engine.",
      "sku_hints": ["DB custom CORE", "DB custom RAM"]
    }
  ],
  "unknowns": [
    {
      "gcp_service": "Cloud Armor",
      "gcp_service_type": "google_compute_security_policy",
      "monthly_cost": 50.00,
      "reason": "No IaC configuration available; billing name does not match any fast-path entry",
      "suggestion": "Provide Terraform files for accurate mapping, or manually specify the AWS equivalent"
    }
  ]
}
```

**File 2: `aws-design-billing-report.md`**

```markdown
# Billing-Only Design Report

## Overview

Mapped X of Y GCP billing services to AWS equivalents.
Source: billing data only (no IaC/Terraform configuration available).
Confidence: billing_inferred for all mappings.

## ⚠️ Accuracy Notice

These mappings are inferred from GCP billing service names and SKU descriptions.
Without Terraform configuration, specific resource settings (instance sizes, regions,
feature flags) cannot be determined. Provide `.tf` files for higher-confidence mapping.

## Mapped Services

### [gcp_service] → [aws_service]

- Monthly GCP cost: $[monthly_cost]
- SKU hints: [sku_hints]
- Confidence: billing_inferred
- Rationale: [rationale]

[repeat per mapped service]

## Unmapped Services

### [gcp_service]

- Monthly GCP cost: $[monthly_cost]
- Reason: [reason]
- Suggestion: [suggestion]

[repeat per unknown]

## Total Monthly GCP Spend: $[total]
```

<!-- TODO: Add cost comparison column once Estimate phase supports billing-only designs -->

## Output Validation Checklist

### aws-design-billing.json

- `metadata.design_source` is `"billing_only"`
- `metadata.total_services` equals `mapped_services` + `unmapped_services`
- Every service from `billing-profile.json` appears in either `services[]` or `unknowns[]`
- All `confidence` values are `"billing_inferred"`
- Every `services[]` entry has `gcp_service`, `aws_service`, `monthly_cost`, `rationale`
- Every `unknowns[]` entry has `gcp_service`, `monthly_cost`, `reason`, `suggestion`
- Output is valid JSON

### aws-design-billing-report.md

- Overview lists mapped vs total count
- Accuracy notice is present
- Every mapped service has a section
- Every unmapped service has a section
