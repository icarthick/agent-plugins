# Terraform Clustering: Deterministic Algorithm

Groups resources into named clusters using priority-ordered rules.

## Input

All resources with fields:

- `address`, `type`, `classification` (PRIMARY/SECONDARY)
- `secondary_role` (if SECONDARY)
- `typed_edges[]`, `depth`, `serves[]`

## Algorithm: Apply Rules in Priority Order

### Rule 1: Networking Cluster

**IF** `google_compute_network` resource exists:

- Group: `google_compute_network` + ALL network_path secondaries (subnetworks, firewalls, routers)
- Cluster ID: `network_{gcp_region}` (e.g., `network_us-central1`)
- **Reasoning**: Network is shared infrastructure; groups all config together

**Output**: 1 cluster (or 0 if no networks found)

**Mark these resources as clustered; remove from unassigned pool.**

### Rule 2: Same-Type Grouping

**FOR EACH** resource type with 2+ PRIMARY resources:

- Group: All PRIMARY resources of identical type
- Cluster ID: `{service_shortname}_{sequence}` (e.g., `cloudrun_001`, `cloudrun_002`)
- **Reasoning**: Identical workloads often migrate together

**Example**: 3× `google_container_cluster` → `k8s_001`, `k8s_002`, `k8s_003`

**Output**: M clusters (one per multi-resource type)

**Mark these resources as clustered; remove from unassigned pool.**

### Rule 3: Seed Clusters

**FOR EACH** remaining PRIMARY resource (unassigned):

- Create cluster seeded by this PRIMARY
- Add all SECONDARY resources in its `serves[]` array
- Cluster ID: `{service_type}_{gcp_region}_{sequence}` (e.g., `cloudrun_us-central1_001`)
- **Reasoning**: Primary + its supports = deployment unit

**Output**: N clusters (one per remaining PRIMARY)

**Mark all included resources as clustered.**

### Rule 4: Merge on Dependencies

**IF** two clusters have bidirectional or data_dependency edges between their PRIMARY resources:

- **AND** they form a single logical deployment unit (determined by: shared infrastructure, sequential deploy, business logic)
- **THEN** merge clusters

**Action**: Combine into one cluster; update ID to reflect both (e.g., `web-api_us-central1_001`)

**Reasoning**: Some workloads must deploy together (e.g., two Cloud Runs sharing database)

**Heuristic**: Merge if one PRIMARY depends on another's output (e.g., Function → Database). Do NOT merge independent workloads.

### Rule 5: Skip API Services

**IF** resource is `google_project_service`:

- Classify as orchestration secondary
- Do NOT create its own cluster
- Attach to cluster of service it enables (e.g., `google_project_service.cloud_run` attaches to Cloud Run cluster)

**Reasoning**: API enablement is prerequisite, not a deployable unit.

### Rule 6: Deterministic Naming

Apply consistent cluster naming:

- **Format**: `{service_category}_{service_type}_{gcp_region}_{sequence}`
- **service_category**: One of: `compute`, `database`, `storage`, `network`, `messaging`, `analytics`, `security`
- **service_type**: GCP service shortname (e.g., `cloudrun`, `sql`, `bucket`, `vpc`)
- **gcp_region**: Source region (e.g., `us-central1`)
- **sequence**: Zero-padded counter (e.g., `001`, `002`)

**Examples**:

- `compute_cloudrun_us-central1_001`
- `database_sql_us-west1_001`
- `storage_bucket_multi-region_001`
- `network_vpc_us-central1_000` (rule 1 network cluster)

**Reasoning**: Names reflect deployment intent; deterministic for reproducibility.

## Output Cluster Schema

Each cluster includes:

```json
{
  "cluster_id": "compute_cloudrun_us-central1_001",
  "name": "Cloud Run Application",
  "type": "compute",
  "description": "Primary: cloud_run_service.app, Secondary: service_account, iam_policy",
  "gcp_region": "us-central1",
  "primary_resources": ["google_cloud_run_service.app"],
  "secondary_resources": ["google_service_account.app_runner"],
  "network": "network_vpc_us-central1_000",
  "creation_order_depth": 2,
  "must_migrate_together": true,
  "dependencies": [],
  "edges": [{"from": "...", "to": "...", "type": "..."}]
}
```

## Determinism Guarantee

Given same Terraform input, algorithm produces same cluster structure every run:

1. Rules applied in fixed order
2. Sequence counters increment deterministically
3. Naming reflects source state, not random IDs
4. No LLM-based decisions (all heuristics deterministic)
