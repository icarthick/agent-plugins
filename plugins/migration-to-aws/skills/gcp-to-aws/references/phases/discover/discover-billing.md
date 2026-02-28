# Discover Phase: Billing Discovery

> Self-contained billing discovery sub-file. Scans for billing CSV/JSON files, parses billing data, and builds service usage profiles.
> If no billing files are found, exits cleanly with no output.

**Status**: Not yet implemented (v1.2 feature). Steps 1-3 below describe expected behavior when implemented.

## Step 0: Self-Scan for Billing Files

Scan the target directory for billing data:

- `**/*billing*.csv` — GCP billing export CSV
- `**/*billing*.json` — BigQuery billing export JSON
- `**/*cost*.csv`, `**/*cost*.json` — Cost report exports
- `**/*usage*.csv`, `**/*usage*.json` — Usage report exports

**Exit gate:** If NO billing files are found, **exit cleanly**. Return no output artifacts. Other sub-discovery files may still produce artifacts.

## Step 1: Parse Billing Data (v1.2+)

Supported formats:
- GCP billing export CSV
- BigQuery billing export JSON

Extract from each line item:
- `service_description` — GCP service name
- `sku_description` — Specific SKU/resource
- `cost` — Cost amount
- `usage_amount` — Usage quantity
- `usage_unit` — Usage unit (e.g., hours, bytes, requests)

Group by service and calculate monthly totals.

## Step 2: Build Service Usage Profile (v1.2+)

From the parsed billing data:

1. List all GCP services with non-zero spend
2. Calculate monthly cost per service
3. Identify top services by spend (sorted descending)
4. Note usage patterns (consistent vs bursty spend)

## Step 3: Write Output (v1.2+)

Write `$MIGRATION_DIR/billing_resources.json`:

```json
{
  "billing_resources": [
    {
      "service": "Cloud Run",
      "monthly_cost_usd": 150.50,
      "evidence": "billing export analysis",
      "confidence": 0.95
    }
  ]
}
```

## Scope Boundary

**This phase covers Discover & Analysis ONLY.**

FORBIDDEN — Do NOT include ANY of:
- AWS service names, recommendations, or equivalents
- Migration strategies, phases, or timelines
- Terraform generation for AWS
- Cost estimates or comparisons
- Effort estimates

**Your ONLY job: Inventory what exists in GCP. Nothing else.**
