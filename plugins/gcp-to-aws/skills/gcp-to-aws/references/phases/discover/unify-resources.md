# Discover Phase: Unify Resources

Combines intermediate outputs from domain discoverers into final inventory and cluster JSON files.
**Execute ALL steps in order. Do not skip or deviate.**

## Step 1: Check for Intermediate Files

1. Check `.migration/[MMDD-HHMM]/` directory for intermediate files:
   - `iac_resources.json` → Present?
   - `billing_resources.json` → Present? (v1.2+)
   - `app_code_resources.json` → Present? (v1.1+)
2. If **ZERO** intermediate files found: **STOP**. Output: "No intermediate files found. No discoverers completed successfully. Check previous phase for errors."
3. Report to user: "Found [N] intermediate source files: [list]"

## Step 2: Determine Path

- **PATH A**: IF `iac_resources.json` exists → Use IaC-based clustering (write both inventory + clusters)
- **PATH B**: IF only billing/app-code files exist (no iac_resources.json) → Flat inventory only (write inventory only)

Report chosen path to user.

## Step 3: Load and Validate Intermediate Files

1. For each present intermediate file:
   - Read and validate JSON syntax (must be valid JSON)
   - Confirm required fields exist (see schema below)
2. If any file fails validation: **STOP**. Output: "Invalid JSON in [filename]. Cannot proceed."

### Required Intermediate File Schemas

**iac_resources.json** (if present):
```json
{
  "timestamp": "ISO 8601 string",
  "total_resources": "integer",
  "resources": [
    {
      "address": "string (e.g., google_compute_instance.web)",
      "type": "string (e.g., google_compute_instance)",
      "classification": "string (PRIMARY or SECONDARY)",
      "secondary_role": "string or null",
      "cluster_id": "string (assigned cluster)",
      "config": "object with resource attributes",
      "dependencies": "array of addresses",
      "typed_edges": "array of edge objects",
      "depth": "integer >= 0",
      "serves": "array of resource addresses"
    }
  ],
  "clusters": "object of cluster_id -> cluster definition"
}
```

**billing_resources.json** (if present, v1.2+):
```json
{
  "billing_resources": [
    {
      "service": "string",
      "monthly_cost_usd": "number",
      "evidence": "string",
      "confidence": "number 0-1"
    }
  ]
}
```

**app_code_resources.json** (if present, v1.1+):
```json
{
  "app_code_resources": [
    {
      "service": "string",
      "detected_imports": "array of import strings",
      "workload_type": "string",
      "evidence": "string",
      "confidence": "number 0-1"
    }
  ]
}
```

## Step 4: Merge Resources Across Sources (PATH A and B)

1. Start with resources from `iac_resources.json` (PATH A) or empty list (PATH B)
2. For each additional source (billing, app-code):
   - For each service in that source:
     - **IF** matching resource exists by type+address: merge by combining evidence and averaging confidence
     - **IF** NO matching resource: add as new resource with source-specific confidence
3. Result: unified resource list with evidence from all sources

## Step 5: Write Final Inventory (PATH A and B)

**This step is MANDATORY. Write file exactly as specified.**

1. Create file: `.migration/[MMDD-HHMM]/gcp-resource-inventory.json`
2. Write with exact schema:
   ```json
   {
     "timestamp": "2026-02-26T14:30:00Z",
     "total_resources": 24,
     "sources": ["iac", "billing", "app_code"],
     "resources": [
       {
         "address": "google_compute_instance.web",
         "type": "google_compute_instance",
         "classification": "PRIMARY",
         "secondary_role": null,
         "cluster_id": "compute_cloudrun_us-central1_001",
         "config": {
           "machine_type": "e2-medium",
           "zone": "us-central1-a"
         },
         "dependencies": ["google_compute_network.vpc"],
         "evidence": ["iac discovery"],
         "confidence": 0.95
       }
     ]
   }
   ```
3. **Validate**: Confirm file exists and contains ALL resources from Step 4

## Step 6: Write Cluster File (PATH A only)

**Execute ONLY if PATH A (iac_resources.json present).**

1. Create file: `.migration/[MMDD-HHMM]/gcp-resource-clusters.json`
2. Extract clusters from `iac_resources.json` and write with exact schema:
   ```json
   {
     "timestamp": "2026-02-26T14:30:00Z",
     "total_clusters": 5,
     "clusters": [
       {
         "cluster_id": "compute_cloudrun_us-central1_001",
         "name": "Cloud Run Application",
         "type": "compute",
         "description": "Serverless container workload with supporting infrastructure",
         "gcp_region": "us-central1",
         "primary_resources": ["google_cloud_run_service.app"],
         "secondary_resources": ["google_service_account.app_runner", "google_*_iam_member.app"],
         "network": "network_vpc_us-central1_000",
         "creation_order_depth": 2,
         "must_migrate_together": true,
         "dependencies": ["database_sql_us-central1_001"],
         "edges": [
           {
             "from": "google_cloud_run_service.app",
             "to": "google_sql_database_instance.db",
             "type": "data_dependency"
           }
         ]
       }
     ]
   }
   ```
3. **Validate**: Confirm file exists and contains all clusters

**If PATH B (no iac_resources.json)**: Skip Step 6, only `gcp-resource-inventory.json` is written.

## Step 7: Validate Output

1. Confirm BOTH required files exist (PATH A) or ONE file (PATH B):
   - PATH A: `gcp-resource-inventory.json` + `gcp-resource-clusters.json` (REQUIRED)
   - PATH B: `gcp-resource-inventory.json` (REQUIRED)
2. Confirm file sizes > 100 bytes (not empty)
3. If validation fails: **STOP**. Output: "Output files missing or invalid. Cannot complete discover phase."
4. Report to user: "✅ Unified resources into final inventory. Wrote [N] files."
