# Phase 4: Estimate AWS Costs (Orchestrator)

**Execute ALL steps in order. Do not skip or optimize.**

## Step 0: Pricing Mode Selection

Before running any sub-estimate file, determine the pricing source.

### MCP Retry Logic

Attempt to reach awspricing with **up to 2 retries** (3 total attempts):

1. **Attempt 1**: Call `get_pricing_service_codes()`
2. **If timeout/error**: Wait 1 second, retry (Attempt 2)
3. **If still fails**: Wait 2 seconds, retry (Attempt 3)
4. **If all 3 attempts fail**: Proceed to fallback

### Live Pricing (pricing_mode = "live")

- **If any attempt succeeds**: Use live AWS pricing for all estimates
- Accuracy: ±5-10%
- Pass `pricing_mode: "live"` to all sub-estimate files

### Fallback Pricing (pricing_mode = "fallback")

- **If all 3 attempts fail**:
  1. Load `shared/pricing-fallback.json`
  2. **Check staleness**:
     - Read `metadata.last_updated` (e.g., "2026-02-24")
     - Calculate days since update: `today - last_updated`
     - If > 90 days: Add warning: "Cached pricing data is >90 days old; accuracy may be significantly degraded"
     - If <= 90 days: Add note: "Using cached 2026 rates (±15-25% accuracy)"
  3. Log warning: "AWS pricing API unavailable; using cached rates from [last_updated]"
  4. **Display to user**: Add visible warning to estimation report with staleness notice
- Accuracy: ±15-25%
- Pass `pricing_mode: "fallback"` to all sub-estimate files

## Step 1: Prerequisites

Read `$MIGRATION_DIR/preferences.json`. If missing: **STOP**. Output: "Phase 2 (Clarify) not completed. Run Phase 2 first."

Check which design artifacts exist in `$MIGRATION_DIR/`:

- `aws-design.json` (infrastructure design from IaC)
- `aws-design-ai.json` (AI workload design)
- `aws-design-billing.json` (billing-only design)

If **none** of these artifacts exist: **STOP**. Output: "No design artifacts found. Run Phase 3 (Design) first."

## Step 2: Routing Rules

### Infrastructure Estimate

IF `aws-design.json` exists:

> Load `estimate-infra.md`

Produces: `estimation-infra.json`, `estimation-infra-report.md`

### Billing-Only Estimate

IF `aws-design-billing.json` exists AND `aws-design.json` does **NOT** exist:

> Load `estimate-billing.md`

Produces: `estimation-billing.json`, `estimation-billing-report.md`

### AI Estimate

IF `aws-design-ai.json` exists:

> Load `estimate-ai.md`

Produces: `estimation-ai.json`, `estimation-ai-report.md`

### Mutual Exclusion

- **estimate-infra** and **estimate-billing** never both run (billing-only is the fallback when no IaC exists).
- **estimate-ai** runs independently alongside either estimate-infra or estimate-billing (if AI artifacts exist).

## Phase Completion

After all applicable sub-estimates finish, update `$MIGRATION_DIR/.phase-status.json`:

- Set `phases.estimate.status` to `"completed"`
- Set `phases.estimate.timestamp` to current ISO 8601 timestamp
- Set `phases.estimate.outputs` to the combined list of output files produced (e.g., `["estimation-infra.json", "estimation-infra-report.md"]` or `["estimation-billing.json", "estimation-billing-report.md", "estimation-ai.json", "estimation-ai-report.md"]`)
- Update `last_updated` to current timestamp

Output to user: "Cost estimation complete. Proceeding to Phase 5: Execution Plan."

## Reference Files

- `shared/pricing-fallback.json` — Cached AWS pricing (±15-25%)
- `shared/output-schema.md` — JSON schemas for estimation output files

## Scope Boundary

**This phase covers financial analysis ONLY.**

FORBIDDEN — Do NOT include ANY of:

- Changes to architecture mappings from the Design phase
- Execution timelines or migration schedules
- Terraform or IaC code generation
- Detailed migration procedures or runbooks
- Team staffing or resource allocation

**Your ONLY job: Show the financial picture of moving to AWS. Nothing else.**
