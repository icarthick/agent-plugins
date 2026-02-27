# Discover Phase: IaC (Terraform) Discovery

Extracts and clusters GCP resources from Terraform files. Produces final inventory and clusters JSON files.
**Execute ALL steps in order. Do not skip or optimize.**

## Step 1: Parse Terraform Files

1. Read all `.tf`, `.tfvars`, and `.tfstate` files in working directory (recursively)
2. Extract all resources matching `google_*` pattern (e.g., `google_compute_instance`, `google_sql_database_instance`)
3. For each resource, capture exactly:
   - `address` (e.g., `google_compute_instance.web`)
   - `type` (e.g., `google_compute_instance`)
   - `config` (object with key attributes: `machine_type`, `name`, `region`, etc.)
   - `raw_hcl` (raw HCL text for this resource, needed for Step 3)
   - `depends_on` (array of addresses this resource depends on)
4. Report total resources found to user (e.g., "Parsed 50 GCP resources from Terraform")

## Step 2: Classify Resources (PRIMARY vs SECONDARY)

1. Read `references/clustering/terraform/classification-rules.md` completely
2. For EACH resource from Step 1, apply classification rules in priority order:
   - **Priority 1**: Check if in PRIMARY list → mark `classification: "PRIMARY"`, continue
   - **Priority 2**: Check if type matches SECONDARY patterns → mark `classification: "SECONDARY"` with `secondary_role` (one of: `identity`, `access_control`, `network_path`, `configuration`, `encryption`, `orchestration`)
   - **Priority 3**: Apply LLM inference heuristics → mark as SECONDARY with `secondary_role` and confidence field
   - **Default**: Mark as `SECONDARY` with `secondary_role: "configuration"` and `confidence: 0.5`
3. Confirm ALL resources have `classification` and (if SECONDARY) `secondary_role` fields
4. Report counts (e.g., "Classified: 12 PRIMARY, 38 SECONDARY")

## Step 3: Build Dependency Edges and Populate Serves

1. Read `references/clustering/terraform/typed-edges-strategy.md` completely
2. For EACH resource from Step 1, extract references from `raw_hcl`:
   - Extract all `google_*\.[\w\.]+` patterns
   - Classify edge type by field name/value context (see typed-edges-strategy.md)
   - Store as `{from, to, edge_type, evidence}` in `typed_edges[]` array
3. For SECONDARY resources, populate `serves[]` array:
   - Trace outgoing references to PRIMARY resources
   - Trace incoming `depends_on` references from PRIMARY resources
   - Include transitive chains (e.g., IAM → SA → Cloud Run)
4. Report dependency summary (e.g., "Found 45 typed edges, 38 secondaries populated serves arrays")

## Step 4: Calculate Topological Depth

1. Read `references/clustering/terraform/depth-calculation.md` completely
2. Use Kahn's algorithm (or equivalent topological sort) to assign `depth` field:
   - Depth 0: resources with no incoming dependencies
   - Depth N: resources where at least one dependency is depth N-1
3. **Detect cycles**: If any resource cannot be assigned depth, flag error: "Circular dependency detected between: [resources]. Breaking lowest-confidence edge."
4. Confirm ALL resources have `depth` field (integer ≥ 0)
5. Report depth summary (e.g., "Depth 0: 8 resources, Depth 1: 15 resources, ..., Max depth: 3")

## Step 5: Apply Clustering Algorithm

1. Read `references/clustering/terraform/clustering-algorithm.md` completely
2. Apply Rules 1-6 in exact priority order:
   - **Rule 1: Networking Cluster** — `google_compute_network` + all `network_path` secondaries → 1 cluster
   - **Rule 2: Same-Type Grouping** — ALL primaries of identical type → 1 cluster (not one per resource)
   - **Rule 3: Seed Clusters** — Each remaining PRIMARY gets cluster + its `serves[]` secondaries
   - **Rule 4: Merge on Dependencies** — Merge only if single deployment unit (rare)
   - **Rule 5: Skip API Services** — `google_project_service` never gets own cluster; attach to service it enables
   - **Rule 6: Deterministic Naming** — `{type}_{subtype}_{sequence}` (e.g., `compute_cloudrun_001`, `database_sql_001`)
3. Assign `cluster_id` to EVERY resource (must match one of generated clusters)
4. Confirm ALL resources have `cluster_id` field
5. Report clustering results (e.g., "Generated 6 clusters from 50 resources")

## Step 6: Write Final Output Files

**This step is MANDATORY. Write both files with exact schemas.**

### 6a: Write gcp-resource-inventory.json

1. Create file: `.migration/[MMDD-HHMM]/gcp-resource-inventory.json`
2. Write with exact schema (see below):

   ```json
   {
     "timestamp": "2026-02-26T14:30:00Z",
     "metadata": {
       "total_resources": 50,
       "primary_resources": 12,
       "secondary_resources": 38,
       "total_clusters": 6,
       "terraform_available": true
     },
     "resources": [
       {
         "address": "google_cloud_run_service.api",
         "type": "google_cloud_run_service",
         "classification": "PRIMARY",
         "secondary_role": null,
         "cluster_id": "compute_cloudrun_001",
         "config": {
           "region": "us-central1",
           "name": "api"
         },
         "dependencies": ["google_sql_database_instance.main"],
         "depth": 2,
         "serves": []
       },
       {
         "address": "google_service_account.app",
         "type": "google_service_account",
         "classification": "SECONDARY",
         "secondary_role": "identity",
         "cluster_id": "compute_cloudrun_001",
         "config": {},
         "dependencies": [],
         "depth": 1,
         "serves": ["google_cloud_run_service.api"]
       }
     ]
   }
   ```

**CRITICAL field names (use EXACTLY these):**

- `address` (resource Terraform address)
- `type` (resource Terraform type)
- `classification` (PRIMARY or SECONDARY)
- `secondary_role` (for secondaries only)
- `cluster_id` (assigned cluster)
- `depth` (topological depth)
- `serves` (for secondaries only)

### 6b: Write gcp-resource-clusters.json

1. Create file: `.migration/[MMDD-HHMM]/gcp-resource-clusters.json`
2. Write with exact schema:

   ```json
   {
     "timestamp": "2026-02-26T14:30:00Z",
     "metadata": {
       "total_clusters": 6
     },
     "clusters": [
       {
         "cluster_id": "compute_cloudrun_001",
         "name": "Cloud Run Services",
         "type": "compute",
         "description": "Primary: cloud_run_service, Secondary: service_account, iam_member",
         "gcp_region": "us-central1",
         "primary_resources": ["google_cloud_run_service.api", "google_cloud_run_service.worker"],
         "secondary_resources": [
           "google_service_account.app",
           "google_cloud_run_service_iam_member.*"
         ],
         "network": "network_vpc_001",
         "creation_order_depth": 2,
         "must_migrate_together": true,
         "dependencies": ["database_sql_001"],
         "edges": [
           {
             "from": "google_cloud_run_service.api",
             "to": "google_sql_database_instance.main",
             "relationship_type": "data_dependency"
           }
         ]
       }
     ]
   }
   ```

**CRITICAL field names (use EXACTLY these):**

- `cluster_id` (matches resources' cluster_id)
- `primary_resources` (array of addresses)
- `secondary_resources` (array of addresses)
- `creation_order_depth` (matches resource depths)
- `edges` (array of {from, to, relationship_type})

### 6c: Validate Both Files Exist

1. Confirm `.migration/[MMDD-HHMM]/gcp-resource-inventory.json` exists and is valid JSON
2. Confirm `.migration/[MMDD-HHMM]/gcp-resource-clusters.json` exists and is valid JSON
3. Verify all resource addresses in inventory appear in exactly one cluster
4. Verify all cluster IDs match resource cluster_id assignments
5. Report to user: "✅ Wrote gcp-resource-inventory.json (X resources) and gcp-resource-clusters.json (Y clusters)"
