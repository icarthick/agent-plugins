# Phase 3: Design AWS Architecture (Orchestrator)

**Execute ALL steps in order. Do not skip or optimize.**

## Prerequisites

Read `$MIGRATION_DIR/preferences.json`. If missing: **STOP**. Output: "Phase 2 (Clarify) not completed. Run Phase 2 first."

Check which discovery artifacts exist in `$MIGRATION_DIR/`:

- `gcp-resource-inventory.json` (IaC discovery ran)
- `gcp-resource-clusters.json` (IaC discovery ran)
- `billing-profile.json` (billing discovery ran)
- `ai-workload-profile.json` (AI workloads detected)

If **none** of these artifacts exist: **STOP**. Output: "No discovery artifacts found. Run Phase 1 (Discover) first."

## Routing Rules

### Infrastructure Design (IaC-based)

IF `gcp-resource-inventory.json` AND `gcp-resource-clusters.json` both exist:

→ Load `design-infra.md`

Produces: `aws-design.json`, `aws-design-report.md`

### Billing-Only Design (fallback)

IF `billing-profile.json` exists AND `gcp-resource-inventory.json` does **NOT** exist:

→ Load `design-billing.md`

Produces: `aws-design-billing.json`, `aws-design-billing-report.md`

### AI Workload Design

IF `ai-workload-profile.json` exists:

→ Load `design-ai.md`

Produces: `aws-design-ai.json`, `aws-design-ai-report.md`

### Mutual Exclusion

- **design-infra** and **design-billing** never both run (billing-only is the fallback when no IaC exists).
- **design-ai** runs independently alongside either design-infra or design-billing (if AI artifacts exist).

## Phase Completion

After all applicable sub-designs finish, update `$MIGRATION_DIR/.phase-status.json`:

- Set `phases.design.status` to `"completed"`
- Set `phases.design.timestamp` to current ISO 8601 timestamp
- Set `phases.design.outputs` to the combined list of output files produced (e.g., `["aws-design.json", "aws-design-report.md"]` or `["aws-design-billing.json", "aws-design-billing-report.md", "aws-design-ai.json", "aws-design-ai-report.md"]`)
- Update `last_updated` to current timestamp

Output to user: "AWS Architecture designed. Proceeding to Phase 4: Estimate Costs."

## Reference Files

Sub-design files may reference rubrics in `design-refs/`:

- `design-refs/index.md` — GCP type → rubric file lookup
- `design-refs/fast-path.md` — Deterministic 1:1 GCP→AWS mappings
- `design-refs/compute.md` — Compute service rubric
- `design-refs/database.md` — Database service rubric
- `design-refs/storage.md` — Storage service rubric
- `design-refs/networking.md` — Networking service rubric
- `design-refs/messaging.md` — Messaging service rubric
- `design-refs/ai.md` — AI/ML service rubric

## Scope Boundary

**This phase covers architecture mapping ONLY.**

FORBIDDEN — Do NOT include ANY of:

- Cost calculations or pricing estimates
- Execution timelines or migration schedules
- Terraform or IaC code generation
- Risk assessments or rollback procedures
- Team staffing or resource allocation

**Your ONLY job: Map GCP resources to AWS services. Nothing else.**
