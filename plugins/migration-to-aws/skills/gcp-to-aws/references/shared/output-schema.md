# Output Schema Reference

Complete JSON schemas for all phase outputs and state files.

**Convention**: Values shown as `X|Y` in examples indicate allowed alternatives — use exactly one value per field, not the literal pipe character.

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
    "terraform_lines": 850,
    "app_code_languages": ["python"]
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

| Field                  | Type             | Set When                                                                                |
| ---------------------- | ---------------- | --------------------------------------------------------------------------------------- |
| `migration_id`         | string           | Created (matches folder name, never changes)                                            |
| `started_at`           | ISO 8601         | Created (never changes)                                                                 |
| `last_updated`         | ISO 8601         | After each phase update                                                                 |
| `project_directory`    | string           | Start of Discover                                                                       |
| `input_files_detected` | object           | End of Discover (terraform_files count, terraform_lines count, app_code_languages list) |
| `discovery_outputs`    | string[]         | End of Discover (list of artifact filenames produced)                                   |
| `phases.*.status`      | string           | Phase transitions: `"pending"` → `"in_progress"` → `"completed"`                        |
| `phases.*.timestamp`   | ISO 8601 or null | Phase completion                                                                        |
| `phases.*.outputs`     | string[]         | Phase completion (files created by that phase)                                          |

**Rules:**

- `phases.*.status` progresses: `"pending"` → `"in_progress"` → `"completed"`. Never goes backward.
- `discovery_outputs` is populated at the end of Discover. Subsequent phases read this to determine available artifacts.
- `input_files_detected` is populated during Discover and never changed.

---

## gcp-resource-inventory.json (Phase 1 output)

Complete inventory of discovered GCP resources with classification, dependencies, and AI detection.

```json
{
  "metadata": {
    "report_date": "2026-02-26",
    "project_directory": "/path/to/terraform",
    "terraform_version": ">= 1.0.0"
  },
  "summary": {
    "total_resources": 50,
    "primary_resources": 12,
    "secondary_resources": 38,
    "total_clusters": 6,
    "classification_coverage": "100%"
  },
  "resources": [
    {
      "address": "google_cloud_run_service.orders_api",
      "type": "google_cloud_run_service",
      "name": "orders_api",
      "file": "cloudrun.tf",
      "classification": "PRIMARY",
      "tier": "compute",
      "classification_source": "hardcoded_rules",
      "confidence": 0.99,
      "config": {
        "timeout": 60,
        "memory_mb": 512,
        "concurrency": 100
      },
      "variables_used": ["var.app_name"],
      "dynamic": false,
      "depth": 3,
      "cluster_id": "compute_cloudrun_us-central1_001"
    },
    {
      "address": "google_service_account.app",
      "type": "google_service_account",
      "name": "app",
      "file": "iam.tf",
      "classification": "SECONDARY",
      "tier": "identity",
      "classification_source": "hardcoded_rules",
      "confidence": 0.99,
      "secondary_role": "identity",
      "serves": ["google_cloud_run_service.orders_api", "google_cloud_run_service.products_api"],
      "config": {
        "account_id": "app-sa"
      },
      "depth": 2,
      "cluster_id": "compute_cloudrun_us-central1_001"
    },
    {
      "address": "google_compute_network.main",
      "type": "google_compute_network",
      "name": "main",
      "file": "vpc.tf",
      "classification": "PRIMARY",
      "tier": "networking",
      "classification_source": "hardcoded_rules",
      "confidence": 0.99,
      "config": {
        "auto_create_subnetworks": false
      },
      "depth": 0,
      "cluster_id": "networking_vpc_us-central1_001"
    }
  ],
  "variables": [],
  "outputs": [],
  "data_sources": [],
  "modules": [],
  "ai_detection": {
    "has_ai_workload": false,
    "confidence": 0,
    "confidence_level": "none",
    "signals_found": [],
    "ai_services": []
  }
}
```

**CRITICAL Field Names** (use EXACTLY these keys):

- `address` — Terraform resource address (NOT `id`, `resource_address`)
- `type` — Resource type (NOT `resource_type`)
- `name` — Resource name component (NOT `resource_name`)
- `file` — Source file path (NOT `source_file`, `filename`)
- `classification` — `"PRIMARY"` or `"SECONDARY"` (NOT `class`, `category`)
- `tier` — Infrastructure layer: compute, database, storage, networking, identity, messaging, monitoring (NOT `layer`)
- `classification_source` — `"hardcoded_rules"` or `"llm_inference"` (NOT `source`)
- `confidence` — Classification confidence 0.0-1.0 (NOT `certainty`)
- `secondary_role` — For secondaries only: identity, access_control, network_path, configuration, encryption, orchestration
- `serves` — For secondaries only: array of primary resource addresses served
- `depth` — Dependency depth (0 = foundational, N = depends on depth N-1)
- `cluster_id` — Which cluster this resource belongs to
- `variables_used` — List of Terraform variables referenced
- `dynamic` — Boolean, whether resource uses count/for_each

**Key Sections:**

- `metadata` — Report metadata (report_date, project_directory, terraform_version)
- `summary` — Aggregate statistics (total_resources, primary/secondary counts, cluster count, classification_coverage)
- `resources[]` — All discovered resources with fields above
- `variables[]` — Extracted Terraform variable blocks
- `outputs[]` — Extracted Terraform output blocks
- `data_sources[]` — Extracted Terraform data sources
- `modules[]` — Extracted Terraform module blocks
- `ai_detection` — AI workload detection results:
  - `has_ai_workload` — boolean
  - `confidence` — 0.0-1.0
  - `confidence_level` — "very_high" (90%+), "high" (70-89%), "medium" (50-69%), "low" (< 50%), "none" (0%)
  - `signals_found[]` — array of detection signals with method, pattern, confidence, evidence
  - `ai_services[]` — list of AI services detected (vertex_ai, bigquery_ml, etc.)

---

## gcp-resource-clusters.json (Phase 1 output)

Resources grouped into logical clusters for migration with full dependency graph and creation order.

```json
{
  "clusters": [
    {
      "cluster_id": "networking_vpc_us-central1_001",
      "gcp_region": "us-central1",
      "primary_resources": [
        "google_compute_network.main"
      ],
      "secondary_resources": [
        "google_compute_subnetwork.app",
        "google_compute_firewall.app"
      ],
      "network": null,
      "creation_order_depth": 0,
      "must_migrate_together": true,
      "dependencies": [],
      "edges": []
    },
    {
      "cluster_id": "database_sql_us-central1_001",
      "gcp_region": "us-central1",
      "primary_resources": [
        "google_sql_database_instance.db"
      ],
      "secondary_resources": [
        "google_sql_database.main"
      ],
      "network": "networking_vpc_us-central1_001",
      "creation_order_depth": 2,
      "must_migrate_together": true,
      "dependencies": ["networking_vpc_us-central1_001"],
      "edges": [
        {
          "from": "google_sql_database_instance.db",
          "to": "google_compute_network.main",
          "relationship_type": "network_membership",
          "evidence": {
            "field_path": "settings.ip_configuration.private_network",
            "reference": "VPC membership"
          }
        }
      ]
    },
    {
      "cluster_id": "compute_cloudrun_us-central1_001",
      "gcp_region": "us-central1",
      "primary_resources": [
        "google_cloud_run_service.orders_api",
        "google_cloud_run_service.products_api"
      ],
      "secondary_resources": [
        "google_service_account.app"
      ],
      "network": "networking_vpc_us-central1_001",
      "creation_order_depth": 3,
      "must_migrate_together": true,
      "dependencies": ["database_sql_us-central1_001"],
      "edges": [
        {
          "from": "google_cloud_run_service.orders_api",
          "to": "google_sql_database_instance.db",
          "relationship_type": "data_dependency",
          "evidence": {
            "field_path": "template.spec.containers[0].env[].value",
            "reference": "DATABASE_URL"
          }
        }
      ]
    }
  ],
  "creation_order": [
    { "depth": 0, "clusters": ["networking_vpc_us-central1_001"] },
    { "depth": 2, "clusters": ["database_sql_us-central1_001"] },
    { "depth": 3, "clusters": ["compute_cloudrun_us-central1_001"] }
  ]
}
```

**Key Fields:**

- `cluster_id` — Unique cluster identifier (deterministic format: `{type}_{subtype}_{region}_{sequence}`)
- `gcp_region` — GCP region for this cluster
- `primary_resources` — GCP resources that map independently
- `secondary_resources` — GCP resources that support primary resources
- `network` — Which VPC cluster this cluster belongs to (cluster ID reference, or null if networking cluster itself)
- `creation_order_depth` — Depth level in topological sort (0 = foundational)
- `must_migrate_together` — Boolean indicating if cluster is an atomic deployment unit
- `dependencies` — Other cluster IDs this cluster depends on (derived from cross-cluster Primary→Primary edges)
- `edges` — Typed relationships between resources with structured evidence
- `creation_order` — Global ordering of clusters by depth level (for migration sequencing)

---

## ai-workload-profile.json (Phase 1 output)

Focused profile of AI/ML workloads including models, capabilities, integration patterns, and supporting infrastructure. Generated by `discover-app-code.md` when AI confidence >= 70%.

```json
{
  "metadata": {
    "report_date": "2026-02-26",
    "project_directory": "/path/to/project",
    "sources_analyzed": {
      "terraform": true,
      "application_code": true,
      "billing_data": false
    }
  },

  "summary": {
    "overall_confidence": 0.95,
    "confidence_level": "very_high",
    "total_models_detected": 2,
    "languages_found": ["python"]
  },

  "models": [
    {
      "model_id": "gemini-pro",
      "service": "vertex_ai_generative",
      "detected_via": ["code", "terraform"],
      "evidence": [
        {
          "source": "code",
          "file": "src/ai/client.py",
          "line": 12,
          "pattern": "GenerativeModel(\"gemini-pro\")"
        },
        {
          "source": "terraform",
          "file": "vertex.tf",
          "resource": "google_vertex_ai_endpoint.main",
          "pattern": "Vertex AI endpoint resource"
        }
      ],
      "capabilities_used": ["text_generation", "streaming"],
      "usage_context": "Recommendation engine - generates product recommendations from user profiles"
    },
    {
      "model_id": "text-embedding-004",
      "service": "vertex_ai_embeddings",
      "detected_via": ["code"],
      "evidence": [
        {
          "source": "code",
          "file": "src/embeddings/indexer.py",
          "line": 5,
          "pattern": "VertexAIEmbeddings()"
        }
      ],
      "capabilities_used": ["embeddings"],
      "usage_context": "Document indexing for semantic search"
    }
  ],

  "integration": {
    "primary_sdk": "google-cloud-aiplatform",
    "sdk_version": "1.38.0",
    "frameworks": ["langchain"],
    "languages": ["python"],
    "pattern": "direct_sdk",
    "capabilities_summary": {
      "text_generation": true,
      "streaming": true,
      "function_calling": false,
      "vision": false,
      "embeddings": true,
      "batch_processing": false
    }
  },

  "infrastructure": [
    {
      "address": "google_vertex_ai_endpoint.main",
      "type": "google_vertex_ai_endpoint",
      "file": "vertex.tf",
      "config": {
        "display_name": "recommendation-endpoint",
        "location": "us-central1"
      }
    },
    {
      "address": "google_service_account.vertex_sa",
      "type": "google_service_account",
      "file": "iam.tf",
      "role": "supports_ai",
      "config": {
        "account_id": "vertex-ai-sa"
      }
    }
  ],

  "current_costs": {
    "monthly_ai_spend": 450,
    "services_detected": ["Vertex AI Predictions", "Generative AI API"]
  },

  "detection_signals": [
    {
      "method": "terraform",
      "pattern": "google_vertex_ai_endpoint resource",
      "confidence": 0.95,
      "evidence": "Found resource 'main' in vertex.tf"
    },
    {
      "method": "code",
      "pattern": "google.cloud.aiplatform import",
      "confidence": 0.95,
      "evidence": "Found in src/ai/client.py line 3"
    }
  ]
}
```

**CRITICAL Field Names** (use EXACTLY these keys):

- `model_id` — Model identifier string (NOT `model_name`, `name`)
- `service` — GCP service category (NOT `service_type`, `gcp_service`)
- `detected_via` — Array of detection sources (NOT `detection_method`, `source`)
- `capabilities_used` — Array of capability strings per model (NOT `capabilities`, `features`)
- `usage_context` — Human-readable description of what the model does (NOT `description`, `purpose`)
- `pattern` — Integration pattern in `integration` object (NOT `integration_type`, `method`)
- `capabilities_summary` — Boolean map in `integration` object (NOT `capabilities`, `feature_flags`)

**Key Fields:**

- `metadata.sources_analyzed` — Which data sources were provided (affects which sections are populated)
- `summary.overall_confidence` — Combined detection confidence from all signals
- `models[]` — Each distinct AI model/service detected, with evidence and capabilities
- `integration.pattern` — How the app connects to AI (`direct_sdk`, `framework`, `rest_api`, `mixed`)
- `integration.capabilities_summary` — Union of all capabilities across all models
- `infrastructure[]` — Terraform resources related to AI (empty array if no Terraform provided)
- `current_costs` — Present ONLY if billing data was provided; omitted entirely otherwise
- `detection_signals[]` — Raw signals from AI detection for transparency

**Conditional sections:**

- `current_costs` — Include ONLY if billing data was provided (billing discovery ran). Omit entirely if no billing data.
- `infrastructure` — Set to `[]` if no Terraform files were provided (IaC discovery did not run).

---

## billing-profile.json (Phase 1 output)

Cost breakdown derived from GCP billing export CSV. Provides service-level spend and AI signal detection from billing data alone.

```json
{
  "metadata": {
    "report_date": "2026-02-24",
    "project_directory": "/path/to/project",
    "billing_source": "gcp-billing-export.csv",
    "billing_period": "2026-01"
  },
  "summary": {
    "total_monthly_spend": 2450.00,
    "service_count": 8,
    "currency": "USD"
  },
  "services": [
    {
      "gcp_service": "Cloud Run",
      "gcp_service_type": "google_cloud_run_service",
      "monthly_cost": 450.00,
      "percentage_of_total": 0.18,
      "top_skus": [
        {
          "sku_description": "Cloud Run - CPU Allocation Time",
          "monthly_cost": 300.00
        },
        {
          "sku_description": "Cloud Run - Memory Allocation Time",
          "monthly_cost": 150.00
        }
      ],
      "ai_signals": []
    },
    {
      "gcp_service": "Cloud SQL",
      "gcp_service_type": "google_sql_database_instance",
      "monthly_cost": 800.00,
      "percentage_of_total": 0.33,
      "top_skus": [
        {
          "sku_description": "Cloud SQL for PostgreSQL - DB custom CORE",
          "monthly_cost": 500.00
        },
        {
          "sku_description": "Cloud SQL for PostgreSQL - DB custom RAM",
          "monthly_cost": 300.00
        }
      ],
      "ai_signals": []
    },
    {
      "gcp_service": "Vertex AI",
      "gcp_service_type": "google_vertex_ai_endpoint",
      "monthly_cost": 600.00,
      "percentage_of_total": 0.24,
      "top_skus": [
        {
          "sku_description": "Vertex AI Prediction - Online Prediction",
          "monthly_cost": 400.00
        },
        {
          "sku_description": "Generative AI - Gemini Pro Input Tokens",
          "monthly_cost": 200.00
        }
      ],
      "ai_signals": ["vertex_ai", "generative_ai"]
    }
  ],
  "ai_signals": {
    "detected": true,
    "confidence": 0.85,
    "services": ["Vertex AI"]
  }
}
```

**Key Fields:**

- `summary.total_monthly_spend` — Total monthly GCP spend from the billing export
- `summary.service_count` — Number of distinct GCP services with charges
- `services[].gcp_service_type` — Terraform resource type equivalent for the service (used by downstream phases)
- `services[].monthly_cost` — Monthly cost for this service
- `services[].top_skus` — Highest-cost line items within the service
- `services[].ai_signals` — AI-related keywords found in SKU descriptions for this service
- `ai_signals.detected` — Whether any AI/ML services were found in the billing data
- `ai_signals.confidence` — Confidence that the project uses AI (derived from billing SKU analysis)
- `ai_signals.services` — List of AI-related GCP services found

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
    "message": "All services validated for regional availability and feature parity|Validation unavailable (awsknowledge MCP unreachable)"
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
          "rationale": "Compute mapping; always-on; Fargate for simplicity",
          "rubric_applied": [
            "Eliminators: PASS",
            "Operational Model: Managed Fargate",
            "User Preference: Cost (q2)",
            "Feature Parity: Full (always-on compute)",
            "Cluster Context: Standalone compute tier",
            "Simplicity: Fargate (managed, no EC2)"
          ]
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
    "message": "Using live AWS pricing API|Using cached rates from 2026-02-24 (±15-25% accuracy due to API unavailability)",
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
