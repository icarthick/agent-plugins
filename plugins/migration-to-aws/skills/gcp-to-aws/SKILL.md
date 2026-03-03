---
name: gcp-to-aws
description: "Migrate workloads from Google Cloud Platform to AWS. Triggers on: migrate from GCP, GCP to AWS, move off Google Cloud, migrate Terraform to AWS, migrate Cloud SQL to RDS, migrate GKE to EKS, migrate Cloud Run to Fargate, Google Cloud migration. Runs a 5-phase process: discover GCP resources from Terraform files, clarify migration requirements, design AWS architecture, estimate costs, and plan execution."
---

# GCP-to-AWS Migration Skill

## Philosophy

- **Re-platform by default**: Select AWS services that match GCP workload types (e.g., Cloud Run → Fargate, Cloud SQL → RDS).
- **Dev sizing unless specified**: Default to development-tier capacity (e.g., db.t4g.micro, single AZ). Upgrade only on user direction.
- **Multi-signal approach**: Design phase adapts based on available inputs — Terraform IaC for infrastructure, billing data for service mapping, and app code for AI workload detection.

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
| `clarify_done`  | always    | Load `references/phases/design/design.md`     |
| `design_done`   | always    | Load `references/phases/estimate/estimate.md` |
| `estimate_done` | always    | Load `references/phases/generate/generate.md` |
| `generate_done` | always    | Migration planning complete                   |

**How to determine current state:** Read `$MIGRATION_DIR/.phase-status.json` → check `phases` object → find the last phase with `status: "completed"`.

**Phase gate checks**: If prior phase incomplete, do not advance (e.g., cannot enter estimate without completed design).

---

## State Validation

When reading `$MIGRATION_DIR/.phase-status.json`, validate before proceeding:

1. **Multiple sessions**: If multiple directories exist under `.migration/`, STOP. Output: "Multiple migration sessions detected. Pick one to continue: [list]"
2. **Invalid JSON**: If `.phase-status.json` fails to parse, STOP. Output: "State file corrupted (invalid JSON). Delete the file and restart the current phase."
3. **Unrecognized phase**: If `phases` object contains a phase not in {discover, clarify, design, estimate, generate}, STOP. Output: "Unrecognized phase: [value]. Valid phases: discover, clarify, design, estimate, generate."
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
    "generate": { "status": "pending", "timestamp": null, "outputs": [] }
  }
}
```

**Status values:** `"pending"` → `"in_progress"` → `"completed"`. Never goes backward.

The `.migration/` directory is automatically protected by a `.gitignore` file created in Phase 1.

---

## Phase Summary Table

| Phase        | Inputs                                                                                                                                                                   | Outputs                                                                                                                                                                                   | Reference                                |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| **Discover** | `.tf` files                                                                                                                                                              | `gcp-resource-inventory.json`, `gcp-resource-clusters.json`, `.phase-status.json` updated                                                                                                 | `references/phases/discover/discover.md` |
| **Clarify**  | `gcp-resource-inventory.json`, `gcp-resource-clusters.json`                                                                                                              | `preferences.json`, `.phase-status.json` updated                                                                                                                                          | `references/phases/clarify.md`           |
| **Design**   | `preferences.json` + discovery artifacts                                                                                                                                 | `aws-design.json` + `aws-design-report.md` (infra), `aws-design-ai.json` + `aws-design-ai-report.md` (AI), `aws-design-billing.json` + `aws-design-billing-report.md` (billing-only)      | `references/phases/design/design.md`     |
| **Estimate** | `aws-design.json` or `aws-design-billing.json` or `aws-design-ai.json`, `preferences.json`                                                                               | `estimation-infra.json` or `estimation-ai.json` or `estimation-billing.json` + reports, `.phase-status.json` updated                                                                      | `references/phases/estimate/estimate.md` |
| **Generate** | `estimation-infra.json` or `estimation-ai.json` or `estimation-billing.json`, `aws-design.json` or `aws-design-billing.json` or `aws-design-ai.json`, `preferences.json` | `generation-infra.json` or `generation-ai.json` or `generation-billing.json` + `terraform/`, `scripts/`, `ai-migration/`, `MIGRATION_GUIDE.md`, `README.md`, `.phase-status.json` updated | `references/phases/generate/generate.md` |

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
│   │   │   ├── discover-app-code.md            # App code discovery
│   │   │   └── discover-billing.md             # Billing data discovery
│   │   ├── clarify.md                          # Phase 2: Clarify requirements
│   │   ├── design/
│   │   │   ├── design.md                       # Phase 3: Design orchestrator
│   │   │   ├── design-infra.md                 # Infrastructure design (IaC-based)
│   │   │   ├── design-ai.md                    # AI workload design (Bedrock)
│   │   │   └── design-billing.md               # Billing-only design (fallback)
│   │   ├── estimate/
│   │   │   ├── estimate.md                     # Phase 4: Estimate orchestrator
│   │   │   ├── estimate-infra.md               # Infrastructure cost analysis
│   │   │   ├── estimate-ai.md                  # AI workload cost analysis
│   │   │   └── estimate-billing.md             # Billing-only cost analysis
│   │   └── generate/
│   │       ├── generate.md                     # Phase 5: Generate orchestrator
│   │       ├── generate-infra.md               # Infrastructure migration plan
│   │       ├── generate-ai.md                  # AI migration plan
│   │       ├── generate-billing.md             # Billing-only migration plan
│   │       ├── generate-artifacts-infra.md     # Terraform + migration scripts
│   │       ├── generate-artifacts-ai.md        # Provider adapter + test harness
│   │       ├── generate-artifacts-billing.md   # Skeleton Terraform
│   │       └── generate-artifacts-docs.md      # MIGRATION_GUIDE.md + README.md
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

| Condition                                   | Action                                                                                                                       |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| No `.tf` files found                        | Stop. Output: "No Terraform files detected. Please provide `.tf` files with your GCP resources and try again."               |
| `.phase-status.json` missing phase gate     | Stop. Output: "Cannot enter Phase X: Phase Y-1 not completed. Start from Phase Y or resume Phase Y-1."                       |
| awspricing unavailable after 3 attempts     | Display user warning about ±15-25% accuracy. Use `pricing-fallback.json`. Add `pricing_source: fallback` to estimation.json. |
| User does not answer all Q1-8               | Offer Mode C (defaults) or Mode D (free text). Phase 2 completes either way.                                                 |
| `aws-design.json` missing required clusters | Stop Phase 4. Output: "Re-run Phase 3 to generate missing cluster designs."                                                  |

## Defaults

- **IaC output**: Terraform configurations, migration scripts, AI migration code, and documentation
- **Region**: `us-east-1` (unless user specifies, or GCP region → AWS region mapping suggests otherwise)
- **Sizing**: Development tier (e.g., `db.t4g.micro` for databases, 0.5 CPU for Fargate)
- **Migration mode**: Adapts based on available inputs (infrastructure, AI, or billing-only)
- **Cost currency**: USD
- **Timeline assumption**: 8-12 weeks total

## Workflow Execution

When invoked, the agent **MUST follow this exact sequence**:

1. **Load phase status**: Read `.phase-status.json` from `.migration/*/`.
   - If missing: Initialize for Phase 1 (Discover)
   - If exists: Determine current phase based on phase field and status value

2. **Determine phase to execute**:
   - If status is `in-progress`: Resume that phase (read corresponding reference file)
   - If status is `completed`: Advance to next phase (read next reference file)
   - Phase mapping for advancement:
     - discover (completed) → Execute clarify (read `references/phases/clarify.md`)
     - clarify (completed) → Execute design (read `references/phases/design/design.md`)
     - design (completed) → Execute estimate (read `references/phases/estimate/estimate.md`)
     - estimate (completed) → Execute generate (read `references/phases/generate/generate.md`)
     - generate (completed) → Migration complete

3. **Read phase reference**: Load the full reference file for the target phase.

4. **Execute ALL steps in order**: Follow every numbered step in the reference file. **Do not skip, optimize, or deviate.**

5. **Validate outputs**: Confirm all required output files exist with correct schema before proceeding.

6. **Update phase status**: Each phase reference file specifies the final `.phase-status.json` update (records the phase that just completed).

7. **Display summary**: Show user what was accomplished, highlight next phase, or confirm migration completion.

**Critical constraint**: Agent must strictly adhere to the reference file's workflow. If unable to complete a step, stop and report the exact step that failed.

User can invoke the skill again to resume from last completed phase.

## Scope Notes

**v1.0 includes:**

- Terraform infrastructure discovery
- App code scanning (AI workload detection)
- Billing data import from GCP
- User requirement clarification (adaptive questions by category)
- Multi-path Design (infrastructure, AI workloads, billing-only fallback)
- AWS cost estimation (from pricing API or fallback)
- Migration artifact generation (Terraform, scripts, AI adapters, documentation)
