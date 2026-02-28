# Output Schema Reference

Complete JSON schemas for all phase outputs and state files.

## .phase-status.json

Hierarchical phase tracking with per-phase metadata. This is the SINGLE source of truth for the `.phase-status.json` schema. All steering files reference this definition.

```json
{
  "migration_id": "0226-1430",
  "started_at": "2026-02-26T14:30:00Z",
  "last_updated": "2026-02-26T15:35:22Z",
  "project_directory": "/path/to/project",
  "input_files_detected": {
    "terraform_files": 12,
    "terraform_lines": 850
  },
  "discovery_outputs": [
    "gcp-resource-inventory.json",
    "gcp-resource-clusters.json"
  ],
  "phases": {
    "discover": {
      "status": "completed",
      "timestamp": "2026-02-26T14:31:00Z",
      "outputs": ["gcp-resource-inventory.json", "gcp-resource-clusters.json"]
    },
    "clarify": {
      "status": "completed",
      "timestamp": "2026-02-26T14:32:00Z",
      "outputs": ["preferences.json"]
    },
    "design": { "status": "in_progress", "timestamp": null, "outputs": [] },
    "estimate": { "status": "pending", "timestamp": null, "outputs": [] },
    "execute": { "status": "pending", "timestamp": null, "outputs": [] }
  }
}
```

**Field Definitions:**

| Field                  | Type             | Set When                                                         |
| ---------------------- | ---------------- | ---------------------------------------------------------------- |
| `migration_id`         | string           | Created (matches folder name, never changes)                     |
| `started_at`           | ISO 8601         | Created (never changes)                                          |
| `last_updated`         | ISO 8601         | After each phase update                                          |
| `project_directory`    | string           | Start of Discover                                                |
| `input_files_detected` | object           | End of Discover (terraform_files count, terraform_lines count)   |
| `discovery_outputs`    | string[]         | End of Discover (list of artifact filenames produced)            |
| `phases.*.status`      | string           | Phase transitions: `"pending"` → `"in_progress"` → `"completed"` |
| `phases.*.timestamp`   | ISO 8601 or null | Phase completion                                                 |
| `phases.*.outputs`     | string[]         | Phase completion (files created by that phase)                   |

**Rules:**

- `phases.*.status` progresses: `"pending"` → `"in_progress"` → `"completed"`. Never goes backward.
- `discovery_outputs` is populated at the end of Discover. Subsequent phases read this to determine available artifacts.
- `input_files_detected` is populated during Discover and never changed.

---

## gcp-resource-inventory.json (Phase 1 output)

All discovered GCP resources with full configuration and dependencies.

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
      "address": "google_sql_database_instance.prod_postgres",
      "type": "google_sql_database_instance",
      "classification": "PRIMARY",
      "secondary_role": null,
      "cluster_id": "database_sql_us-central1_001",
      "config": {
        "database_version": "POSTGRES_13",
        "region": "us-central1",
        "tier": "db-custom-2-7680"
      },
      "dependencies": [],
      "depth": 0,
      "serves": []
    },
    {
      "address": "google_compute_instance.web",
      "type": "google_compute_instance",
      "classification": "PRIMARY",
      "secondary_role": null,
      "cluster_id": "compute_instance_us-central1_001",
      "config": {
        "machine_type": "e2-medium",
        "zone": "us-central1-a",
        "image": "debian-11"
      },
      "dependencies": ["google_compute_network.vpc"],
      "depth": 1,
      "serves": []
    },
    {
      "address": "google_compute_network.vpc",
      "type": "google_compute_network",
      "classification": "SECONDARY",
      "secondary_role": "network_path",
      "cluster_id": "compute_instance_us-central1_001",
      "config": {},
      "dependencies": [],
      "depth": 0,
      "serves": ["google_compute_instance.web"]
    }
  ]
}
```

**Schema fields:**

- `metadata`: Summary statistics (total_resources, primary/secondary counts, cluster count, terraform_available)
- `resources`: Array of all discovered resources with fields:
  - `address`: Terraform resource address
  - `type`: Terraform resource type
  - `classification`: PRIMARY or SECONDARY
  - `secondary_role`: Role if SECONDARY (identity, access_control, network_path, configuration, encryption, orchestration); null for PRIMARY
  - `cluster_id`: Assigned cluster identifier
  - `config`: Resource configuration (varies by type)
  - `dependencies`: List of Terraform addresses this resource depends on
  - `depth`: Topological depth (0 = no dependencies, N = depends on depth N-1)
  - `serves`: List of resources this secondary supports (for SECONDARY only)

---

## gcp-resource-clusters.json (Phase 1 output)

Clustered resources by affinity and deployment order.

```json
{
  "clusters": [
    {
      "cluster_id": "compute_instance_us-central1_001",
      "gcp_region": "us-central1",
      "creation_order_depth": 1,
      "primary_resources": [
        "google_compute_instance.web"
      ],
      "secondary_resources": [
        "google_compute_network.vpc",
        "google_compute_firewall.web-allow-http"
      ],
      "edges": [
        {
          "from": "google_compute_instance.web",
          "to": "google_compute_network.vpc",
          "relationship_type": "network_path"
        },
        {
          "from": "google_compute_instance.web",
          "to": "google_compute_firewall.web-allow-http",
          "relationship_type": "network_path"
        }
      ]
    },
    {
      "cluster_id": "database_sql_us-central1_001",
      "gcp_region": "us-central1",
      "creation_order_depth": 0,
      "primary_resources": [
        "google_sql_database_instance.prod_postgres"
      ],
      "secondary_resources": [],
      "edges": []
    }
  ]
}
```

---

## preferences.json (Phase 2 output)

Adaptive migration preferences organized as design constraints. Each constraint tracks its provenance via `chosen_by`. Questions are generated by category (A-F) based on which discovery artifacts exist and which GCP services are detected.

```json
{
  "metadata": {
    "timestamp": "2026-02-26T14:30:00Z",
    "discovery_artifacts": ["gcp-resource-inventory.json"],
    "questions_asked": ["Q1", "Q2", "Q3", "Q4", "Q5", "Q6", "Q7", "Q11", "Q12"],
    "questions_defaulted": ["Q8"],
    "questions_skipped_extracted": [],
    "questions_skipped_early_exit": [],
    "questions_skipped_not_applicable": ["Q9", "Q10"],
    "category_e_enabled": false,
    "inventory_clarifications": {}
  },
  "design_constraints": {
    "target_region": { "value": "us-east-1", "chosen_by": "user" },
    "compliance": { "value": ["hipaa"], "chosen_by": "user" },
    "gcp_monthly_spend": { "value": "$5K-$20K", "chosen_by": "user" },
    "availability": { "value": "multi-az", "chosen_by": "default" },
    "cutover_strategy": { "value": "maintenance-window-weekly", "chosen_by": "user" },
    "kubernetes": { "value": "eks-or-ecs", "chosen_by": "user" },
    "database_tier": { "value": "standard", "chosen_by": "user" },
    "db_io_workload": { "value": "medium", "chosen_by": "user" }
  }
}
```

### Field Definitions

| Field                                       | Description                                                                                                                                                                                     |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `metadata.timestamp`                        | ISO 8601 timestamp                                                                                                                                                                              |
| `metadata.discovery_artifacts`              | List of discovery files that existed when clarify ran                                                                                                                                           |
| `metadata.questions_asked`                  | Questions presented to the user                                                                                                                                                                 |
| `metadata.questions_defaulted`              | Questions where the user skipped and default was applied                                                                                                                                        |
| `metadata.questions_skipped_extracted`      | Questions skipped because inventory already provided the answer                                                                                                                                 |
| `metadata.questions_skipped_early_exit`     | Questions skipped due to early-exit logic (e.g., Q7 skipped because Q4=multi-cloud)                                                                                                             |
| `metadata.questions_skipped_not_applicable` | Questions skipped because the relevant service wasn't in the inventory                                                                                                                          |
| `metadata.category_e_enabled`               | Whether the user opted into Category E (Migration Posture)                                                                                                                                      |
| `metadata.inventory_clarifications`         | Category B answers for billing-source inventories (v1.2)                                                                                                                                        |
| `design_constraints.<key>.value`            | The constraint value                                                                                                                                                                            |
| `design_constraints.<key>.chosen_by`        | Provenance: `"user"` (explicitly answered), `"default"` (system default — includes "I don't know"), `"extracted"` (inferred from inventory), `"derived"` (computed from combination of answers) |
| `ai_constraints`                            | _(v1.1)_ Present ONLY if Category F fired (AI discovery). Omit entirely otherwise                                                                                                               |

### Rules

- Every entry in `design_constraints` is an object with `value` and `chosen_by` fields
- Only write a key to `design_constraints` if the answer produces a constraint; absent keys mean "no constraint — Design decides"
- No null values in `design_constraints`
- `ai_constraints` section is present ONLY if Category F fired (v1.1: currently always omitted)

---

## aws-design.json (Phase 3 output)

AWS services mapped from GCP resources, clustered by affinity.

```json
{
  "validation_status": {
    "status": "completed|skipped",
    "message": "All services validated for regional availability and feature parity" | "Validation unavailable (awsknowledge MCP unreachable)"
  },
  "clusters": [
    {
      "cluster_id": "compute_instance_us-central1_001",
      "gcp_region": "us-central1",
      "aws_region": "us-east-1",
      "resources": [
        {
          "gcp_address": "google_compute_instance.web",
          "gcp_type": "google_compute_instance",
          "aws_service": "Fargate",
          "aws_config": {
            "cpu": "0.5",
            "memory": "1024",
            "region": "us-east-1"
          },
          "confidence": "inferred",
          "rationale": "Compute mapping; always-on; Fargate for simplicity"
        }
      ]
    }
  ],
  "warnings": [
    "Service X not available in us-east-1; feature parity check deferred to us-west-2"
  ],
  "timestamp": "2026-02-26T14:30:00Z"
}
```

---

## estimation.json (Phase 4 output)

Monthly operating costs, one-time migration costs, and ROI analysis.

```json
{
  "pricing_source": {
    "status": "live|fallback",
    "message": "Using live AWS pricing API" | "Using cached rates from 2026-02-24 (±15-25% accuracy due to API unavailability)",
    "fallback_staleness": {
      "last_updated": "2026-02-24",
      "days_old": 3,
      "is_stale": false,
      "staleness_warning": null | "⚠️ Cached pricing data is >90 days old; accuracy may be significantly degraded"
    },
    "services_by_source": {
      "live": ["Fargate", "RDS Aurora", "S3", "ALB"],
      "fallback": ["NAT Gateway"],
      "estimated": []
    },
    "services_with_missing_fallback": []
  },
  "monthly_costs": {
    "premium": {
      "total": 5000,
      "breakdown": {
        "Fargate": 1200,
        "RDS Aurora": 2500,
        "S3": 500,
        "ALB": 200,
        "NAT Gateway": 300,
        "Data Transfer": 300
      }
    },
    "balanced": {
      "total": 3500,
      "breakdown": {
        "Fargate": 800,
        "RDS Aurora Serverless": 1800,
        "S3": 500,
        "ALB": 200,
        "NAT Gateway": 200
      }
    },
    "optimized": {
      "total": 2200,
      "breakdown": {
        "Fargate Spot": 300,
        "RDS Aurora Serverless": 1200,
        "S3": 500,
        "NAT Gateway": 200
      }
    }
  },
  "one_time_costs": {
    "dev_hours": "150 hours @ $150/hr = $22,500",
    "data_transfer": "500 GB @ $0.02/GB = $10",
    "training": "Team AWS training = $5,000",
    "total": 27510
  },
  "roi": {
    "assumed_gcp_monthly": 4500,
    "aws_monthly_balanced": 3500,
    "monthly_savings": 1000,
    "payback_months": 27.51,
    "five_year_savings": 32490
  },
  "assumptions": [
    "24/7 workload operation",
    "us-east-1 region selection",
    "No Reserved Instances purchased",
    "No Spot instances in Balanced tier",
    "GCP monthly cost: user estimate"
  ],
  "timestamp": "2026-02-26T14:30:00Z"
}
```

---

## execution.json (Phase 5 output)

Timeline, risk assessment, and rollback procedures.

```json
{
  "timeline_weeks": 12,
  "critical_path": [
    "VPC setup (Week 1)",
    "PoC deployment (Week 3-5)",
    "Data migration (Week 9-10)",
    "DNS cutover (Week 11)"
  ],
  "risks": [
    {
      "category": "data_loss",
      "probability": "low",
      "impact": "critical",
      "mitigation": "Dual-write replication for 2 weeks; full backup before cutover"
    },
    {
      "category": "performance_regression",
      "probability": "medium",
      "impact": "high",
      "mitigation": "PoC testing (Week 3-5); load testing (Week 6)"
    },
    {
      "category": "team_capacity",
      "probability": "medium",
      "impact": "medium",
      "mitigation": "Allocate 2 FTE engineers; external support if needed"
    }
  ],
  "rollback_window": "Reversible until DNS cutover (Week 11); manual after",
  "gcp_teardown_week": 14,
  "timestamp": "2026-02-26T14:30:00Z"
}
```

---

## Design Resource Schema (aws-design.json resource object)

Template for individual resource mappings in aws-design.json.

```json
{
  "gcp_address": "google_sql_database_instance.prod_postgres",
  "gcp_type": "google_sql_database_instance",
  "gcp_config": {
    "database_version": "POSTGRES_13",
    "region": "us-central1",
    "tier": "db-custom-2-7680"
  },
  "aws_service": "RDS Aurora PostgreSQL",
  "aws_config": {
    "engine_version": "14.7",
    "instance_class": "db.r6g.xlarge",
    "multi_az": true,
    "region": "us-east-1"
  },
  "confidence": "deterministic|inferred",
  "rationale": "1:1 Cloud SQL → RDS Aurora; Multi-AZ for production HA",
  "rubric_applied": [
    "Eliminators: PASS",
    "Operational Model: Managed RDS Aurora",
    "User Preference: database_tier=standard, db_io_workload=medium",
    "Feature Parity: Full (binary logs, replication)",
    "Cluster Context: Consistent with app tier",
    "Simplicity: RDS Aurora (managed)"
  ]
}
```
