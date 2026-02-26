# Discover Phase: IaC (Terraform) Discovery

Extracts and clusters GCP resources from Terraform files. **Execute ALL steps in order. Do not skip or optimize.**

## Step 1: Parse Terraform Files

1. Read all `.tf`, `.tfvars`, and `.tfstate` files in working directory (recursively)
2. Extract all resources matching `google_*` pattern (e.g., `google_compute_instance`, `google_sql_database_instance`)
3. For each resource, capture exactly:
   - `address` (e.g., `google_compute_instance.web`)
   - `type` (e.g., `google_compute_instance`)
   - `config` (object with key attributes: `machine_type`, `name`, `region`, etc.)
   - `raw_hcl` (raw HCL text for this resource, needed for Step 3)
   - `depends_on` (array of addresses this resource depends on)
4. Report total resources found to user

## Step 2: Apply Classification Rules

1. Read `references/clustering/terraform/classification-rules.md` completely
2. For EACH resource from Step 1, apply classification rules in priority order:
   - **Priority 1**: Check if in PRIMARY list → mark `classification: "PRIMARY"`, skip to Step 3
   - **Priority 2**: Check if type matches SECONDARY patterns → mark `classification: "SECONDARY"` with `secondary_role` field (one of: `identity`, `access_control`, `network_path`, `configuration`, `encryption`, `orchestration`)
   - **Priority 3**: Apply LLM inference heuristics → mark as SECONDARY with `secondary_role` and confidence field
   - **Default**: Mark as `SECONDARY` with `secondary_role: "configuration"` and `confidence: 0.5`
3. Confirm ALL resources have `classification` and `secondary_role` (if SECONDARY) fields

## Step 3: Build Typed Dependency Edges

1. Read `references/clustering/terraform/typed-edges-strategy.md` completely
2. For EACH resource from Step 1, extract references from `raw_hcl`:
   - Extract all `google_*\.[\w\.]+` patterns
   - Classify edge type by field name context (see typed-edges-strategy.md rules):
     - Field name contains `DATABASE*`, `DB_*`, `SQL_*` → `data_dependency`
     - Field name contains `REDIS*`, `CACHE*` → `cache_dependency`
     - Field name contains `PUBSUB*`, `TOPIC*` → `publishes_to`
     - Field name contains `BUCKET*`, `STORAGE*` → `writes_to` or `reads_from`
     - Field name is `vpc_connector` → `network_path`
     - Field name is `kms_key_name` → `encryption`
     - In `depends_on` array → `orchestration`
   - Store edge with evidence (field_path, env_var_name, reference)
3. For SECONDARY resources, populate `serves[]` array (list of PRIMARY resources this secondary supports)
4. Add `typed_edges[]` array to each resource

## Step 4: Calculate Topological Depth

1. Read `references/clustering/terraform/depth-calculation.md` completely
2. Use Kahn's algorithm to assign `depth` field to EVERY resource:
   - Depth 0: resources with no incoming dependencies
   - Depth N: resources where at least one dependency is depth N-1
3. **Detect cycles**: If any resource cannot be assigned depth, flag error: "Circular dependency detected between: [resources]. Breaking lowest-confidence edge."
4. Confirm ALL resources have `depth` field (integer ≥ 0)

## Step 5: Apply Clustering Algorithm

1. Read `references/clustering/terraform/clustering-algorithm.md` completely
2. Apply Rules 1-6 in exact order:
   - **Rule 1**: Group `google_compute_network` + all `network_path` secondaries → cluster
   - **Rule 2**: Group each resource type with 2+ PRIMARY resources → clusters
   - **Rule 3**: Seed cluster from each remaining PRIMARY + its `serves[]` secondaries
   - **Rule 4**: Merge clusters only if single deployment unit
   - **Rule 5**: Skip `google_project_service`, attach to service it enables
   - **Rule 6**: Name deterministically: `{category}_{service_type}_{region}_{sequence}`
3. Assign `cluster_id` to EVERY resource (must match one of generated clusters)
4. Confirm ALL resources have `cluster_id` field

## Step 6: Write Intermediate Output File

**This step is MANDATORY. Write the file exactly as specified.**

1. Create file: `.migration/[MMDD-HHMM]/iac_resources.json`
2. Write with exact schema:
   ```json
   {
     "timestamp": "2026-02-26T14:30:00Z",
     "total_resources": 24,
     "resources": [
       {
         "address": "google_compute_instance.web",
         "type": "google_compute_instance",
         "classification": "PRIMARY",
         "secondary_role": null,
         "cluster_id": "compute_cloudrun_us-central1_001",
         "config": {
           "machine_type": "e2-medium",
           "zone": "us-central1-a",
           "region": "us-central1"
         },
         "dependencies": ["google_compute_network.vpc"],
         "typed_edges": [
           {
             "target": "google_sql_database_instance.prod",
             "edge_type": "data_dependency",
             "evidence": {
               "field_path": "env.DATABASE_URL",
               "reference": "google_sql_database_instance.prod.connection_name"
             }
           }
         ],
         "depth": 2,
         "serves": []
       }
     ],
     "clusters": {
       "compute_cloudrun_us-central1_001": {
         "name": "Cloud Run Application",
         "primary_resources": ["google_cloud_run_service.app"],
         "secondary_resources": ["google_service_account.app_runner"],
         "network": "network_vpc_us-central1_000",
         "creation_order_depth": 2
       }
     }
   }
   ```
3. **Validate file exists**: Confirm `.migration/[MMDD-HHMM]/iac_resources.json` is written before Step 6 completes
4. Report to user: "✅ Parsed X resources. Applied clustering rules. Wrote iac_resources.json."
