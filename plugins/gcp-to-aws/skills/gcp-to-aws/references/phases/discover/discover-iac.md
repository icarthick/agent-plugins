# Discover Phase: IaC (Terraform) Discovery

Extracts and clusters GCP resources from Terraform files.

## Step 1: Parse Terraform Files

Read all `.tf`, `.tfvars`, and `.tfstate` files. Extract all resources matching `google_*` type (e.g., `google_compute_instance`, `google_sql_database_instance`).

For each resource, capture:

- Resource type (e.g., `google_compute_instance`)
- Logical address (e.g., `google_compute_instance.web`)
- Configuration (key attributes like `machine_type`, `name`, `region`)
- Raw HCL text (for reference extraction)
- Dependencies (explicit `depends_on` and references)

## Step 2: Apply Classification Rules

See `references/clustering/terraform/classification-rules.md`.

Mark each resource as:

- `PRIMARY`: Workload-bearing resources (compute, database, storage, messaging)
- `SECONDARY`: Supporting infrastructure (network, IAM, monitoring) with `secondary_role` field

Assign `secondary_role` from: `identity`, `access_control`, `network_path`, `configuration`, `encryption`, `orchestration`.

## Step 3: Build Typed Dependency Edges

See `references/clustering/terraform/typed-edges-strategy.md`.

For each resource, extract all references to other resources. Classify edge type by field context:

- `DATABASE*`, `DB_*` fields → `data_dependency`
- `REDIS*`, `CACHE*` fields → `cache_dependency`
- `PUBSUB*`, `TOPIC*` fields → `publishes_to`
- `BUCKET*`, `STORAGE*` fields → `writes_to`/`reads_from`
- `vpc_connector` field → `network_path`
- `kms_key_name` field → `encryption`
- `depends_on` → `orchestration`

Populate `serves[]` array for secondaries (outgoing refs + transitive chains).

## Step 4: Calculate Topological Depth

See `references/clustering/terraform/depth-calculation.md`.

Assign `depth` field via topological sort:

- Depth 0: resources with no dependencies
- Depth N: resources depending on depth N-1 resources

Detect cycles; flag warnings and break at lowest-confidence edges.

## Step 5: Apply Clustering Algorithm

See `references/clustering/terraform/clustering-algorithm.md`.

Apply Rules 1-6 to group resources into named clusters:

1. Networking cluster: `google_compute_network` + network_path secondaries
2. Same-type grouping: multiple primaries of identical type
3. Seed clusters: each remaining primary + its secondaries via `serves[]`
4. Merge on dependencies: only if single deployment unit
5. Skip API services: attach to cluster of enabled service
6. Deterministic naming: `{type}_{subtype}_{sequence}`

## Step 6: Write Intermediate Output

Write `iac_resources.json` containing all resources with fields:

- `address`, `type`, `classification`, `secondary_role`
- `config`, `dependencies`, `typed_edges`, `depth`
- `cluster_id`, `serves[]`

This intermediate file is consumed by `unify-resources.md`.
