# Phase 1: Discover GCP Resources

## Step 0: Initialize Migration State

Create `.migration/[MMDD-HHMM]/` directory (e.g., `.migration/0226-1430/`) using current timestamp.

Write initial `.phase-status.json`:

```json
{
  "phase": "discover",
  "status": "in-progress",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

## Step 1: Locate Terraform Files

Scan working directory recursively for:

- `.tf` files
- `.tfvars` files
- `.tfstate` files

If **zero** files found: **STOP**. Output: "No Terraform files detected. Provide `.tf` files in your project root and try again."

If found: List files and continue.

## Step 2: Parse GCP Resources

Read all `.tf` files. Extract all resources matching `google_*` type (e.g., `google_compute_instance`, `google_sql_database_instance`).

For each resource, capture:

- Resource type (`google_compute_instance`, `google_cloudfunctions_function`, etc.)
- Logical address (e.g., `google_compute_instance.web`)
- Configuration (key attributes like `machine_type`, `name`, `region`)
- Dependencies (explicit `depends_on`, and inferred from references)

## Step 3: Classify and Cluster Resources

**Classification**: Mark each resource as:

- `PRIMARY`: Workload-bearing (compute, database, storage, messaging)
- `SECONDARY`: Supporting infrastructure (network, IAM, monitoring)

**Clustering**: Group resources by:

1. GCP region (source of truth)
2. Workload affinity (resources in same Terraform module or by manual grouping)
3. Dependency order (topological sort by `depends_on`)

Assign a `cluster_id` to each group (e.g., `web-app-us-central1`, `api-backend-us-west1`).

## Step 4: Build Dependency Graph

For each resource, determine:

- Direct dependencies (from `depends_on` or references to other resources)
- Order of operations (which resources must be created first)
- Critical path (longest chain of dependencies)

## Step 5: Write Outputs

**File 1: `gcp-resource-inventory.json`**

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
      "config": {
        "machine_type": "e2-medium",
        "zone": "us-central1-a",
        "image": "debian-11"
      },
      "dependencies": ["google_compute_network.vpc"]
    }
  ]
}
```

**File 2: `gcp-resource-clusters.json`**

```json
{
  "clusters": [
    {
      "cluster_id": "web-app-us-central1",
      "gcp_region": "us-central1",
      "creation_order_depth": 0,
      "primary_resources": [
        "google_compute_instance.web",
        "google_storage_bucket.assets"
      ],
      "secondary_resources": [
        "google_compute_network.vpc"
      ]
    }
  ]
}
```

## Step 6: Update Phase Status

Update `.phase-status.json`:

```json
{
  "phase": "clarify",
  "status": "completed",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

Output to user: "Discovered X resources across Y clusters. Proceeding to Phase 2: Clarify."
