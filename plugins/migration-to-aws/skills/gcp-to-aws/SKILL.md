---
name: gcp-to-aws
description: "Migrate workloads from Google Cloud Platform to AWS. Triggers on: migrate from GCP, GCP to AWS, move off Google Cloud, migrate Terraform to AWS, migrate Cloud SQL to RDS, migrate GKE to EKS, migrate Cloud Run to Fargate, Google Cloud migration. Runs a 6-phase process: discover GCP resources from Terraform files, app code, or billing exports, clarify migration requirements, design AWS architecture, estimate costs, plan execution, and collect optional feedback. Do not use for: Azure or on-premises migrations to AWS, AWS-to-GCP reverse migration, general AWS architecture advice without migration intent, GCP-to-GCP refactoring, or multi-cloud deployments that do not involve migrating off GCP."
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

User must provide at least one GCP source:

- **Terraform IaC**: `.tf` files (with optional `.tfvars`, `.tfstate`)
- **Application code**: Source files with GCP SDK or AI framework imports
- **Billing data**: GCP billing/cost/usage export files (CSV or JSON)

If none of the above are found, stop and ask user to provide at least one source type.

---

## State Machine

This is the execution controller. After completing each phase, consult this table to determine the next action.

| Current State   | Condition | Next Action                                   |
| --------------- | --------- | --------------------------------------------- |
| `start`         | always    | Load `references/phases/discover/discover.md` |
| `discover_done` | always    | Load `references/phases/clarify/clarify.md`   |
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
3. **Unrecognized phase**: If `phases` object contains a phase not in {discover, clarify, design, estimate, generate, feedback}, STOP. Output: "Unrecognized phase: [value]. Valid phases: discover, clarify, design, estimate, generate, feedback."
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
    "generate": { "status": "pending", "timestamp": null, "outputs": [] },
    "feedback": { "status": "pending", "timestamp": null, "outputs": [] }
  }
}
```

**Status values:** `"pending"` → `"in_progress"` → `"completed"`. Never goes backward.

The `.migration/` directory is automatically protected by a `.gitignore` file created in Phase 1.

---

## Phase Summary Table

| Phase        | Inputs                                                                                                                                                                   | Outputs                                                                                                                                                                                   | Reference                                |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| **Discover** | `.tf` files, app source code, and/or billing exports (at least one required)                                                                                             | `gcp-resource-inventory.json`, `gcp-resource-clusters.json`, `ai-workload-profile.json`, `billing-profile.json`, `.phase-status.json` updated (outputs vary by input)                     | `references/phases/discover/discover.md` |
| **Clarify**  | Discovery artifacts (`gcp-resource-inventory.json`, `gcp-resource-clusters.json`, `ai-workload-profile.json`, `billing-profile.json` — whichever exist)                  | `preferences.json`, `.phase-status.json` updated                                                                                                                                          | `references/phases/clarify/clarify.md`   |
| **Design**   | `preferences.json` + discovery artifacts                                                                                                                                 | `aws-design.json` (infra), `aws-design-ai.json` (AI), `aws-design-billing.json` (billing-only)                                                                                            | `references/phases/design/design.md`     |
| **Estimate** | `aws-design.json` or `aws-design-billing.json` or `aws-design-ai.json`, `preferences.json`                                                                               | `estimation-infra.json` or `estimation-ai.json` or `estimation-billing.json`, `.phase-status.json` updated                                                                                | `references/phases/estimate/estimate.md` |
| **Generate** | `estimation-infra.json` or `estimation-ai.json` or `estimation-billing.json`, `aws-design.json` or `aws-design-billing.json` or `aws-design-ai.json`, `preferences.json` | `generation-infra.json` or `generation-ai.json` or `generation-billing.json` + `terraform/`, `scripts/`, `ai-migration/`, `MIGRATION_GUIDE.md`, `README.md`, `.phase-status.json` updated | `references/phases/generate/generate.md` |
| **Feedback** | `.phase-status.json` (discover completed minimum), all existing migration artifacts                                                                                      | `feedback.json`, `trace.json`, `.phase-status.json` updated                                                                                                                               | `references/phases/feedback/feedback.md` |

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
│   │   ├── clarify/
│   │   │   ├── clarify.md                     # Phase 2: Clarify orchestrator
│   │   │   ├── clarify-global.md              # Category A: Global/Strategic (Q1-Q7)
│   │   │   ├── clarify-compute.md             # Categories B+C: Config Gaps + Compute (Q8-Q11)
│   │   │   ├── clarify-database.md            # Category D: Database (Q12-Q13)
│   │   │   ├── clarify-ai.md                  # Category F: AI/Bedrock (Q14-Q22)
│   │   │   └── clarify-ai-only.md             # Standalone AI-only migration flow
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
│   │   ├── generate/
│   │   │   ├── generate.md                     # Phase 5: Generate orchestrator
│   │   │   ├── generate-infra.md               # Infrastructure migration plan
│   │   │   ├── generate-ai.md                  # AI migration plan
│   │   │   ├── generate-billing.md             # Billing-only migration plan
│   │   │   ├── generate-artifacts-infra.md     # Terraform configurations
│   │   │   ├── generate-artifacts-scripts.md  # Migration scripts
│   │   │   ├── generate-artifacts-ai.md        # Provider adapter + test harness
│   │   │   ├── generate-artifacts-billing.md   # Skeleton Terraform
│   │   │   └── generate-artifacts-docs.md      # MIGRATION_GUIDE.md + README.md
│   │   └── feedback/
│   │       ├── feedback.md                     # Phase 6: Feedback orchestrator
│   │       └── feedback-trace.md               # Anonymized trace builder
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
│       ├── schema-phase-status.md              # .phase-status.json schema (loaded by discover.md)
│       ├── schema-discover-iac.md              # gcp-resource-inventory + clusters schemas (loaded by discover-iac.md)
│       ├── schema-discover-ai.md               # ai-workload-profile schema (loaded by discover-app-code.md)
│       ├── schema-discover-billing.md          # billing-profile schema (loaded by discover-billing.md)
│       ├── schema-estimate-infra.md            # estimation-infra.json schema (loaded by estimate-infra.md at write time)
│       ├── cached_prices.json                  # Pre-fetched live AWS pricing (±5-10%, primary source)
│       └── pricing-fallback.json               # Broad coverage static cache (±15-25%, tertiary fallback)
│
├── assets/
│   ├── main.tf.template                        # Terraform provider/backend boilerplate
│   ├── variables.tf.template                   # Global variable declarations
│   ├── provider_adapter.py.template            # Feature-flag dual-provider adapter
│   ├── test_comparison.py.template             # A/B test harness (source vs Bedrock)
│   └── setup_bedrock.sh.template               # Bedrock IAM + model access setup
```

| Condition                                                     | Action                                                                                                                                           |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| No GCP sources found (no `.tf`, no app code, no billing data) | Stop. Output: "No GCP sources detected. Provide at least one source type (Terraform files, application code, or billing exports) and try again." |
| `.phase-status.json` missing phase gate                       | Stop. Output: "Cannot enter Phase X: Phase Y-1 not completed. Start from Phase Y or resume Phase Y-1."                                           |
| awspricing unavailable after 3 attempts                       | Display user warning about ±15-25% accuracy. Use `pricing-fallback.json`. Add `pricing_source: fallback` to estimation.json.                     |
| User does not answer all Q1-8                                 | Offer Mode C (defaults) or Mode D (free text). Phase 2 completes either way.                                                                     |
| `aws-design.json` missing required clusters                   | Stop Phase 4. Output: "Re-run Phase 3 to generate missing cluster designs."                                                                      |

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
     - discover (completed) → Execute clarify (read `references/phases/clarify/clarify.md`)
     - clarify (completed) → Execute design (read `references/phases/design/design.md`)
     - design (completed) → Execute estimate (read `references/phases/estimate/estimate.md`)
     - estimate (completed) → Execute generate (read `references/phases/generate/generate.md`)
     - generate (completed) → Migration complete

3. **Read phase reference**: Load the full reference file for the target phase.

4. **Execute ALL steps in order**: Follow every numbered step in the reference file. **Do not skip, optimize, or deviate.**

5. **Validate outputs**: Confirm all required output files exist with correct schema before proceeding.

6. **Update phase status**: Each phase reference file specifies the final `.phase-status.json` update (records the phase that just completed).

7. **Feedback checkpoint**: After a phase completes, check if feedback should be offered. This runs **before** advancing to the next phase.

   - **After Discover** (if `phases.feedback.status` is `"pending"`): Output to user:
     "Would you like to share quick feedback (5 optional questions + anonymized usage data) to help improve this tool? Your data never includes resource names, file paths, or account IDs.
     [N] Send feedback now
     [L] Wait until after the Estimate phase"
     - If user picks **N** → Load `references/phases/feedback/feedback.md`, execute it, then continue to Clarify.
     - If user picks **L** → Continue to Clarify (feedback stays `"pending"`).

   - **After Estimate** (if `phases.feedback.status` is `"pending"`): Output to user:
     "Would you like to share quick feedback now? (5 optional questions + anonymized usage data)
     [Y] Yes, share feedback
     [N] No thanks, continue to Generate"
     - If user picks **Y** → Load `references/phases/feedback/feedback.md`, execute it, then continue to Generate.
     - If user picks **N** → Set `phases.feedback.status` to `"completed"`, `phases.feedback.timestamp` to current ISO 8601 timestamp, `phases.feedback.outputs` to `[]`, update `last_updated`. Continue to Generate.

   - **After Generate**: No feedback offer. If `phases.feedback.status` is still `"pending"`, set it to `"completed"` with empty outputs (user had two chances and chose to defer/skip).

8. **Display summary**: Show user what was accomplished, highlight next phase, or confirm migration completion.

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
- Optional feedback collection with anonymized telemetry
