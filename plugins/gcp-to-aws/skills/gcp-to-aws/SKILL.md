---
name: gcp-to-aws
description: "Migrate workloads from Google Cloud Platform to AWS. Triggers on: migrate from GCP, GCP to AWS, move off Google Cloud, migrate Terraform to AWS, migrate Cloud SQL to RDS, migrate GKE to EKS, migrate Cloud Run to Fargate, Google Cloud migration. Runs a 5-phase process: discover GCP resources from Terraform files, clarify migration requirements, design AWS architecture, estimate costs, and plan execution."
---

# GCP-to-AWS Migration Skill

## Philosophy

- **Re-platform by default**: Select AWS services that match GCP workload types (e.g., Cloud Run → Fargate, Cloud SQL → RDS).
- **Dev sizing unless specified**: Default to development-tier capacity (e.g., db.t4g.micro, single AZ). Upgrade only on user direction.
- **Infrastructure-first approach**: v1.0 migrates Terraform-defined infrastructure only. App code scanning and billing import are v1.1+.

## Prerequisites

User must provide GCP infrastructure-as-code:

- One or more `.tf` files (Terraform)
- Optional: `.tfvars` or `.tfstate` files

If no Terraform files are found, stop immediately and ask user to provide them.

## State Management

Migration state lives in `.migration/[MMDD-HHMM]/` directory (created by Phase 1, persists across invocations):

```
.migration/
└── 0226-1430/                       # MMDD-HHMM timestamp
    ├── .phase-status.json           # Current phase tracking
    ├── gcp-resource-inventory.json  # All GCP resources found
    ├── gcp-resource-clusters.json   # Clustered resources by affinity
    ├── clarified.json               # User answers (Phase 2 output)
    ├── aws-design.json              # AWS services mapping (Phase 3 output)
    ├── estimation.json              # Cost breakdown (Phase 4 output)
    └── execution.json               # Timeline + risks (Phase 5 output)
```

**.phase-status.json schema:**

```json
{
  "phase": "discover|clarify|design|estimate|execute",
  "status": "in-progress|completed",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

If `.phase-status.json` exists and phase is `completed`, advance to next phase. If `in-progress`, resume from that phase.

## Phase Routing

1. **On skill invocation**: Check for `.migration/*/` directory
   - If none exist: Initialize Phase 1 (Discover)
   - If exists: Load `.phase-status.json`, resume from last completed phase

2. **After each phase completion**: Update `.phase-status.json` with next phase:
   - discover → clarify
   - clarify → design
   - design → estimate
   - estimate → execute
   - execute → (done, offer summary and cleanup options)

3. **Phase gate checks**: If prior phase incomplete, do not advance (e.g., cannot enter estimate without completed design)

## Phase Summary Table

| Phase        | Inputs                                          | Outputs                                                                                   | Reference                                |
| ------------ | ----------------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------- |
| **Discover** | `.tf` files                                     | `gcp-resource-inventory.json`, `gcp-resource-clusters.json`, `.phase-status.json` updated | `references/phases/discover/discover.md` |
| **Clarify**  | `gcp-resource-inventory.json`, user answers     | `clarified.json`, `.phase-status.json` updated                                            | `references/phases/clarify.md`  |
| **Design**   | `gcp-resource-inventory.json`, `clarified.json` | `aws-design.json`, `aws-design-report.md`, `.phase-status.json` updated                   | `references/phases/design.md`   |
| **Estimate** | `aws-design.json`, `clarified.json`             | `estimation.json` (cost tables), `.phase-status.json` updated                             | `references/phases/estimate.md` |
| **Execute**  | `aws-design.json`, `clarified.json`             | `execution.json` (timeline), `.phase-status.json` updated                                 | `references/phases/execute.md`  |

## MCP Servers

**awspricing** (for cost estimation):

1. Call `get_pricing_service_codes()` to detect availability
2. If success: use live AWS pricing
3. If timeout/error: fall back to `references/shared/pricing-fallback.json` (includes 2026 on-demand rates for major services)

**awsknowledge** (for design validation):

1. Use for service best practices, regional availability, feature parity checks
2. Fallback: none; skip validation if unavailable (non-critical)

## Error Handling

| Condition                                   | Action                                                                                                         |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| No `.tf` files found                        | Stop. Output: "No Terraform files detected. Please provide `.tf` files with your GCP resources and try again." |
| `.phase-status.json` missing phase gate     | Stop. Output: "Cannot enter Phase X: Phase Y-1 not completed. Start from Phase Y or resume Phase Y-1."         |
| awspricing timeout after 2 retries          | Log warning, use `pricing-fallback.json`. Continue estimation.                                                 |
| User does not answer all Q1-8               | Offer Mode C (defaults) or Mode D (free text). Phase 2 completes either way.                                   |
| `aws-design.json` missing required clusters | Stop Phase 4. Output: "Re-run Phase 3 to generate missing cluster designs."                                    |

## Defaults

- **IaC**: CDK TypeScript (no Terraform output; v1.0 is design/estimate only)
- **Region**: `us-east-1` (unless user specifies, or GCP region → AWS region mapping suggests otherwise)
- **Sizing**: Development tier (e.g., `db.t4g.micro` for databases, 0.5 CPU for Fargate)
- **Migration mode**: Full infrastructure path (no AI-only subset path in v1.0)
- **Cost currency**: USD
- **Timeline assumption**: 8-12 weeks total

## Workflow Execution

When invoked, the agent:

1. Calls the phase reference file matching current `.phase-status.json` phase
2. Passes all discovered state (JSON files) as context to that reference
3. Reference file executes its steps, writes outputs to `.migration/*/`
4. Agent updates `.phase-status.json` to mark phase complete
5. Agent displays summary and offers to proceed to next phase (or exit)

User can invoke the skill again at any time to resume or switch phases (read `.phase-status.json` for context).

## Scope Notes

**v1.0 includes:**

- Terraform infrastructure discovery (no app code scanning)
- User requirement clarification (8 structured questions)
- Structured Design (cluster-based mapping from Terraform)
- AWS cost estimation (from pricing API or fallback)
- Execution timeline and risk assessment

**Deferred to v1.1+:**

- App code scanning (runtime detection of compute workload types)
- AI-only fast-track path in Clarify/Design
- Billing data import from GCP
- Flat Design path (for non-Terraform codebases)
