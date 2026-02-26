# Discover Phase: Unify Resources

Combines intermediate outputs from domain discoverers into final inventory and cluster JSON files.

## Step 1: Check for Intermediate Files

Determine which discoverers completed successfully:

- `iac_resources.json` → Present?
- `billing_resources.json` → Present? (v1.2+)
- `app_code_resources.json` → Present? (v1.1+)

If **zero** intermediate files exist: **STOP**. Output: "No discoverers completed. Check logs for errors."

## Step 2: Deduplicate Resources Across Sources

For resources found in multiple intermediate files (same GCP service type + address from different sources):

- Merge evidence fields: combine all evidence strings
- Increase confidence: average confidence scores if multiple sources agree
- Keep single entry in final inventory

## Step 3: Determine Clustering Path

- **PATH A**: IF `iac_resources.json` present → Use IaC-based cluster structure
  - Resources are pre-clustered via `references/clustering/terraform/`
  - Write both inventory and cluster JSON files
- **PATH B**: IF only billing/app-code resources present → Flat inventory (no clusters)
  - Write inventory only; skip cluster file
  - Clusters can be inferred in Design phase if user refines requirements

## Step 4: Write Final Inventory

Write `gcp-resource-inventory.json` containing all resources merged across sources:

```json
{
  "timestamp": "2026-02-26T14:30:00Z",
  "total_resources": 24,
  "resources": [
    {
      "address": "google_compute_instance.web",
      "type": "google_compute_instance",
      "classification": "PRIMARY",
      "cluster_id": "web-app-us-central1",
      "config": {},
      "dependencies": [],
      "evidence": ["iac discovery", "billing signals"],
      "confidence": 0.95
    }
  ]
}
```

## Step 5: Write Cluster File (PATH A only)

If PATH A taken, write `gcp-resource-clusters.json` from pre-computed clusters:

```json
{
  "clusters": [
    {
      "cluster_id": "web-app-us-central1",
      "gcp_region": "us-central1",
      "creation_order_depth": 0,
      "primary_resources": [],
      "secondary_resources": []
    }
  ]
}
```

If PATH B taken, skip this step.
