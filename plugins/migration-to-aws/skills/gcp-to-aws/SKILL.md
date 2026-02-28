---
name: gcp-to-aws
description: "Migrate workloads from Google Cloud Platform to AWS. Triggers on: migrate from GCP, GCP to AWS, move off Google Cloud, migrate Terraform to AWS, migrate Cloud SQL to RDS, migrate GKE to EKS, migrate Cloud Run to Fargate, Google Cloud migration. Runs a 5-phase process: discover GCP resources from Terraform files, clarify migration requirements, design AWS architecture, estimate costs, and plan execution."
---

# GCP-to-AWS Migration Skill

## Philosophy

- **Re-platform by default**: Select AWS services that match GCP workload types (e.g., Cloud Run → Fargate, Cloud SQL → RDS).
- **Dev sizing unless specified**: Default to development-tier capacity (e.g., db.t4g.micro, single AZ). Upgrade only on user direction.
- **Infrastructure-first approach**: v1.0 migrates Terraform-defined infrastructure only. App code scanning and billing import are v1.1+.

---

## Definitions

- **"Load"** = Read the file using the Read tool and follow its instructions. Do not summarize or skip sections.
- **`$MIGRATION_DIR`** = The run-specific directory under `.migration/` (e.g., `.migration/0226-1430/`). Set during Phase 1 (Discover).

---

## Prerequisites

User must provide GCP infrastructure-as-code:

- One or more `.tf` files (Terraform)
- Optional: `.tfvars` or `.tfstate` files

If no Terraform files are found, stop immediately and ask user to provide them.

---

## State Machine

This is the execution controller. After completing each phase, consult this table to determine the next action.

| Current State   | Condition | Next Action                                   |
| --------------- | --------- | --------------------------------------------- |
| `start`         | always    | Load `references/phases/discover/discover.md` |
| `discover_done` | always    | Load `references/phases/clarify.md`           |
| `clarify_done`  | always    | Load `references/phases/design.md`            |
| `design_done`   | always    | Load `references/phases/estimate.md`          |
| `estimate_done` | always    | Load `references/phases/execute.md`           |
| `execute_done`  | always    | Migration planning complete                   |

**How to determine current state:** Read `$MIGRATION_DIR/.phase-status.json` → check `phases` object → find the last phase with `status: "completed"`.

**Phase gate checks**: If prior phase incomplete, do not advance (e.g., cannot enter estimate without completed design).

---

## State Validation

When reading `$MIGRATION_DIR/.phase-status.json`, validate before proceeding:

1. **Multiple sessions**: If multiple directories exist under `.migration/`, STOP. Output: "Multiple migration sessions detected. Pick one to continue: [list]"
2. **Invalid JSON**: If `.phase-status.json` fails to parse, STOP. Output: "State file corrupted (invalid JSON). Delete the file and restart the current phase."
3. **Unrecognized phase**: If `phases` object contains a phase not in {discover, clarify, design, estimate, execute}, STOP. Output: "Unrecognized phase: [value]. Valid phases: discover, clarify, design, estimate, execute."
4. **Unrecognized status**: If any `phases.*.status` is not in {pending, in_progress, completed}, STOP. Output: "Unrecognized status: [value]. Valid values: pending, in_progress, completed."

---

## State Management

Migration state lives in `$MIGRATION_DIR` (`.migration/[MMDD-HHMM]/`), created by Phase 1 and persisted across invocations.

**.phase-status.json schema:**

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

**Status values:** `"pending"` → `"in_progress"` → `"completed"`. Never goes backward.

The `.migration/` directory is automatically protected by a `.gitignore` file created in Phase 1.

---

## Phase Summary Table

| Phase        | Inputs                                                      | Outputs                                                                                   | Reference                                |
| ------------ | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------- |
| **Discover** | `.tf` files                                                 | `gcp-resource-inventory.json`, `gcp-resource-clusters.json`, `.phase-status.json` updated | `references/phases/discover/discover.md` |
| **Clarify**  | `gcp-resource-inventory.json`, `gcp-resource-clusters.json` | `preferences.json`, `.phase-status.json` updated                                          | `references/phases/clarify.md`           |
| **Design**   | `gcp-resource-inventory.json`, `preferences.json`           | `aws-design.json`, `aws-design-report.md`, `.phase-status.json` updated                   | `references/phases/design.md`            |
| **Estimate** | `aws-design.json`, `preferences.json`                       | `estimation.json`, `estimation-report.md`, `.phase-status.json` updated                   | `references/phases/estimate.md`          |
| **Execute**  | `aws-design.json`, `preferences.json`                       | `execution.json`, `execution-timeline.md`, `.phase-status.json` updated                   | `references/phases/execute.md`           |

---

## MCP Servers

**awspricing** (for cost estimation):

- Provides `get_pricing`, `get_pricing_service_codes`, `get_pricing_service_attributes` tools
- Only needed during Estimate phase. Discover and Design do not require it.
- Fallback: if unavailable, uses `references/shared/pricing-fallback.json` (cached 2026 rates, ±15-25% accuracy)

---

## Files in This Skill

```
gcp-to-aws/
├── SKILL.md                                    ← You are here (orchestrator + state machine)
│
├── references/
│   ├── phases/
│   │   ├── discover/
│   │   │   ├── discover.md                     # Phase 1: Discover orchestrator
│   │   │   ├── discover-iac.md                 # Terraform/IaC discovery
│   │   │   ├── discover-app-code.md            # App code discovery (v1.1+)
│   │   │   └── discover-billing.md             # Billing data discovery (v1.2+)
│   │   ├── clarify.md                          # Phase 2: Clarify requirements
│   │   ├── design.md                           # Phase 3: Design AWS architecture
│   │   ├── estimate.md                         # Phase 4: Cost estimation
│   │   └── execute.md                          # Phase 5: Execution planning
│   │
│   ├── design-refs/
│   │   ├── index.md                            # Lookup table: GCP type → design-ref file
│   │   ├── fast-path.md                        # Deterministic 1:1 mappings (Pass 1)
│   │   ├── compute.md                          # Compute mappings (Cloud Run, GCE, GKE, etc.)
│   │   ├── database.md                         # Database mappings (Cloud SQL, Spanner, etc.)
│   │   ├── storage.md                          # Storage mappings (GCS, Filestore, etc.)
│   │   ├── networking.md                       # Networking mappings (VPC, LB, DNS, etc.)
│   │   ├── messaging.md                        # Messaging mappings (Pub/Sub, etc.)
│   │   └── ai.md                               # AI mappings (Vertex AI → Bedrock)
│   │
│   ├── clustering/terraform/
│   │   ├── classification-rules.md             # Primary/secondary classification
│   │   ├── clustering-algorithm.md             # Cluster formation rules
│   │   ├── depth-calculation.md                # Topological depth calculation
│   │   └── typed-edges-strategy.md             # Edge type assignment
│   │
│   └── shared/
│       ├── output-schema.md                    # JSON schemas for all output files
│       └── pricing-fallback.json               # Cached AWS pricing (±15-25%)
```

---

## Scope Notes

**v1.0 includes:**

- Terraform infrastructure discovery (no app code scanning)
- User requirement clarification (adaptive questions by category)
- Structured Design (cluster-based mapping from Terraform)
- AWS cost estimation (from pricing API or fallback)
- Execution timeline and risk assessment

**Deferred to v1.1+:**

- App code scanning (runtime detection of compute workload types)
- AI-only fast-track path in Clarify/Design
- Billing data import from GCP
- Flat Design path (for non-Terraform codebases)
