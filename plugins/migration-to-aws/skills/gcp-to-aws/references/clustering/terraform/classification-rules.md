# Terraform Clustering: Classification Rules

Hardcoded lists for classifying GCP resources as PRIMARY or SECONDARY.

## Priority 1: PRIMARY Resources (Workload-Bearing)

These resource types are always PRIMARY:

- `google_cloud_run_service` — Serverless container workload
- `google_container_cluster` — Kubernetes cluster
- `google_compute_instance` — Virtual machine
- `google_cloudfunctions_function` — Serverless function
- `google_sql_database_instance` — Relational database
- `google_firestore_database` — Document database
- `google_bigquery_dataset` — Data warehouse
- `google_storage_bucket` — Object storage
- `google_redis_instance` — In-memory cache
- `google_pubsub_topic` — Message queue
- `google_compute_network` — Virtual network (VPC). Anchors the networking cluster (see clustering-algorithm.md Rule 1)
- `google_dns_managed_zone` — DNS zone
- `module.*` — Terraform module (treated as primary container)

**Action**: Mark as `PRIMARY`, classification done. No secondary_role.

## Priority 2: SECONDARY Resources by Role

Match resource type against secondary classification table. Each match assigns a `secondary_role`:

### Identity (`identity`)

- `google_service_account` — Workload identity

### Access Control (`access_control`)

- `google_*_iam_member` — IAM binding (all variants)
- `google_*_iam_policy` — IAM policy (all variants)

### Network Path (`network_path`)

- `google_vpc_access_connector` — VPC connector for serverless
- `google_compute_subnetwork` — Subnet
- `google_compute_firewall` — Firewall rule
- `google_compute_router` — Cloud router
- `google_compute_router_nat` — NAT rule
- `google_service_networking_connection` — VPC peering

### Configuration (`configuration`)

- `google_sql_database` — SQL schema
- `google_sql_user` — SQL user
- `google_secret_manager_secret` — Secret vault
- `google_dns_record_set` — DNS record
- `google_monitoring_alert_policy` — Alert rule (skipped in design; no AWS equivalent)

### Encryption (`encryption`)

- `google_kms_crypto_key` — KMS encryption key
- `google_kms_key_ring` — KMS key ring

### Orchestration (`orchestration`)

- `null_resource` — Terraform orchestration marker
- `time_sleep` — Orchestration delay
- `google_project_service` — API service enablement (prerequisite, not a deployable unit)

**Action**: Mark as `SECONDARY` with assigned role.

## Priority 3: LLM Inference Fallback

If resource type not in Priority 1 or 2, apply heuristic patterns:

- Name contains `scheduler`, `task`, `job` → `SECONDARY` / `orchestration`
- Name contains `log`, `metric`, `alert`, `trace` → `SECONDARY` / `configuration`
- Type contains `policy` or `binding` → `SECONDARY` / `access_control`
- Type contains `network` or `subnet` → `SECONDARY` / `network_path`

**Default**: If all heuristics fail: `SECONDARY` / `configuration` with confidence 0.5

## Serves[] Population

For SECONDARY resources, populate `serves[]` array (list of PRIMARY resources it supports):

1. Extract all outgoing references from this SECONDARY's config
2. Include direct references: `field = resource_type.name.id` patterns
3. Include transitive chains: if referenced resource is also SECONDARY, trace to PRIMARY

**Example**: `google_compute_firewall` → references `google_compute_network` (SECONDARY) → serves `google_compute_instance.web` (PRIMARY)

**Serves array**: Points back to PRIMARY workloads affected by this firewall rule. Trace through SECONDARY resources until a PRIMARY is reached.
