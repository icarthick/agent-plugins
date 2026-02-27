# Reviewer Feedback - PR #73: migration-to-aws Plugin

**Date**: 2026-02-26
**PR**: [#73 - feat(migration-to-aws): Migration plugin](https://github.com/awslabs/agent-plugins/pull/73)
**Reviewer**: krokoko

---

## Issues Identified

### 🔴 CRITICAL - Phase Status Schema Bug

**Issue**: `.phase-status.json` records the NEXT phase as "completed", contradicting routing logic.

**Details**:
- After discover completes, discover.md writes: `{ "phase": "clarify", "status": "completed" }`
- This literally says "clarify is completed" when clarify hasn't started
- SKILL.md Phase Routing says: "If phase is completed, advance to next phase" — would skip clarify → design
- SKILL.md Workflow Execution Step 2 says: "Route based on phase field" — would go to clarify
- **These two instructions contradict each other**

**Affected Transitions**:
- discover → clarify
- clarify → design
- design → estimate
- estimate → execute

**Solution Options**:
1. Record what completed: `{ "phase": "discover", "status": "completed" }` (routing infers next)
2. Record next phase: `{ "phase": "clarify", "status": "pending" }`

**Status**: ⏳ PENDING

---

### 🔴 CRITICAL - Schema Inconsistency: gcp-resource-inventory.json

**Issue**: Two different schemas defined for same output file.

**Details**:
- **discover-iac.md schema**:
  ```json
  {
    "timestamp": "...",
    "metadata": {
      "total_resources": ...,
      "primary_resources": ...,
      ...
    },
    "resources": [...]
  }
  ```

- **output-schema.md schema**:
  ```json
  {
    "timestamp": "...",
    "total_resources": ...,
    "resources": [...]
  }
  ```

**Impact**: Agent following one file produces output incompatible with the other.

**Status**: ⏳ PENDING

---

### 🔴 CRITICAL - Cluster ID Naming Inconsistency

**Issue**: Two different formats specified for cluster IDs.

**Details**:
- **clustering-algorithm.md Rule 6**: `{service_category}_{service_type}_{gcp_region}_{sequence}`
  - Example: `compute_cloudrun_us-central1_001`

- **output-schema.md examples**: `web-us-central1`, `database-us-central1`
  - Uses hyphens, no sequence number
  - Different structure entirely

**Impact**: Agent receives contradictory instructions for same field.

**Status**: ⏳ PENDING

---

### 🟠 IMPORTANT - Factual Error in compute.md Example 2

**Issue**: Incorrect timeout calculation eliminates Lambda incorrectly.

**Details**:
- **GCP Function**: `google_cloudfunctions_function` (runtime=python39, timeout=540s)
- **Error**: Claims "540s > 15min limit" → cannot use Lambda
- **Fact**: 540 seconds = 9 minutes, which is **less than** 15 minutes (900s)
- **Correct conclusion**: Function WOULD work fine on Lambda

**File**: `plugins/migration-to-aws/skills/gcp-to-aws/references/design-refs/compute.md`

**Status**: ✅ RESOLVED
- Split into Example 2a (540s, short-running → Lambda) and Example 2b (1200s, long-running → Fargate)
- Teaches both scenarios correctly with accurate timeout math

---

### 🟠 IMPORTANT - .migration/ Directory Pollution

**Issue**: Skill creates state files without user guidance on `.gitignore`.

**Details**:
- Skill creates `.migration/[MMDD-HHMM]/` directory in working directory
- Contains phase state files (.phase-status.json, etc.)
- No instruction to add `.migration/` to `.gitignore`
- Users will likely commit these accidentally

**Solution Options**:
1. Advise users to add `.migration/` to `.gitignore`
2. Handle automatically (e.g., create .gitignore entry in skill)
3. Store state elsewhere (e.g., `.claude/plugin-state/`)

**Status**: ⏳ PENDING

---

### 🟠 IMPORTANT - Missing awsiac MCP Server

**Issue**: Execute phase promises IaC generation, but server not configured.

**Details**:
- `.mcp.json` only declares: `awsknowledge`, `awspricing`
- Execute phase completion message says: "Ready to generate IaC code. Say 'generate CDK code' or 'export Terraform' to proceed."
- SKILL.md scope notes: "v1.0 is design/estimate only" — no execute phase
- No `awsiac` MCP server configured

**Problem**: Sets expectation that IaC generation is available in v1.0, but it's not.

**Solution**:
- Either implement execute phase + awsiac server
- Or remove IaC generation message from execute completion

**Status**: ✅ RESOLVED
- Removed misleading IaC generation message from execute.md line 185
- Updated final output to clarify: "Use this plan to guide your migration. All phases of the GCP-to-AWS migration analysis are complete."
- Aligns with v1.0 scope: design/estimate/execute timeline only, not IaC generation
- IaC generation deferred to v1.1+

---

### 🟡 IMPORTANT - Duplicate Content Across Files

**Issue**: Schemas repeated in multiple files, risking drift.

**Details**:
- **clarified.json schema** appears in:
  - clarify.md
  - clarify-questions.md
  - output-schema.md

- **Inventory/cluster schemas** appear in:
  - discover-iac.md
  - output-schema.md

**Risk**: Drift already observed (see gcp-resource-inventory.json issue above)

**Solution**: Single source of truth for each schema
- Option 1: Consolidate all schemas in `output-schema.md` only
- Option 2: Create dedicated `schemas/` directory with individual schema files
- Option 3: Use $ref references to avoid duplication

**Status**: ⏳ PENDING

---

## Resolution Plan

### Priority 1 (Blocking):
- [ ] Fix phase status schema logic
- [ ] Resolve gcp-resource-inventory.json schema conflict
- [ ] Resolve cluster ID naming conflict

### Priority 2 (High):
- [ ] Fix compute.md factual error
- [ ] Address .migration/ directory pollution
- [ ] Clarify awsiac MCP server status

### Priority 3 (Medium):
- [ ] Consolidate duplicate schemas

---

## Notes

- These issues must be resolved before PR can be merged
- Some require coordination between multiple reference files
- Schema issues suggest need for better validation/testing during development
