# Phase 1: Discover GCP Resources

Lightweight orchestrator that detects available source types and delegates to domain-specific discoverers.
**Execute ALL steps in order. Do not skip or deviate.**

## Step 0: Initialize Migration State

1. Create `.migration/[MMDD-HHMM]/` directory (e.g., `.migration/0226-1430/`) using current timestamp (MMDD = month/day, HHMM = hour/minute)
2. Write `.phase-status.json` with exact schema:
   ```json
   {
     "phase": "discover",
     "status": "in-progress",
     "timestamp": "2026-02-26T14:30:00Z",
     "version": "1.0.0"
   }
   ```
3. Confirm file exists before proceeding to Step 1.

## Step 1: Scan for Available Source Types

1. Recursively scan working directory for source files:
   - **Terraform**: `.tf`, `.tfvars`, `.tfstate` files (primary v1.0)
   - **Billing** (v1.2+): GCP billing CSV/JSON exports (skip if not available)
   - **App code** (v1.1+): Python/Node/Go with `google.cloud.*` imports (skip if not available)
2. **IF zero sources found**: STOP and output: "No GCP sources detected (no `.tf` files, billing exports, or app code). Provide at least one source type and try again."
3. Report detected sources to user (e.g., "Found Terraform files in: [list]")

## Step 2: Invoke Terraform Discoverer

**This step is MANDATORY for v1.0. Execute exactly as written.**

1. **IF Terraform files found** (from Step 1):
   - Read `references/phases/discover/discover-iac.md` completely
   - Follow ALL steps in discover-iac.md exactly as written
   - **WAIT for completion**: Confirm `iac_resources.json` exists in `.migration/[MMDD-HHMM]/` before proceeding
   - **Validate schema**: Confirm file contains all required resource fields (see discover-iac.md output schema)
   - Proceed to Step 3
2. **IF Terraform files NOT found**:
   - Skip this step, proceed to Step 3

## Step 3: Invoke Other Discoverers (v1.1+, v1.2+)

**Note**: Currently stubs. Skip these in v1.0.

1. **IF Billing files found** (v1.2+):
   - Read `references/phases/discover/discover-billing.md`
   - Follow all steps
   - **WAIT for completion**: Confirm `billing_resources.json` exists
   - (Currently stub; skip in v1.0)
2. **IF App code found** (v1.1+):
   - Read `references/phases/discover/discover-app-code.md`
   - Follow all steps
   - **WAIT for completion**: Confirm `app_code_resources.json` exists
   - (Currently stub; skip in v1.0)

## Step 4: Unify Resources from All Sources

**Execute exactly as written.**

1. Read `references/phases/discover/unify-resources.md` completely
2. Follow ALL steps in unify-resources.md exactly as written
3. **WAIT for completion**: Confirm both output files exist:
   - `gcp-resource-inventory.json` (REQUIRED)
   - `gcp-resource-clusters.json` (REQUIRED if IaC discoverer ran)
4. **Validate schemas**: Confirm files match exact schemas defined in unify-resources.md

## Step 5: Update Phase Status

1. Update `.phase-status.json` with exact schema:
   ```json
   {
     "phase": "clarify",
     "status": "completed",
     "timestamp": "2026-02-26T14:30:00Z",
     "version": "1.0.0"
   }
   ```
2. Output to user: "✅ Discover phase complete. Discovered X total resources across Y clusters. Proceeding to Phase 2: Clarify."

## Error Handling

- **Missing `.migration` directory**: Create it (Step 0)
- **No sources found**: STOP with error message (Step 1)
- **discover-iac.md fails**: STOP and report exact failure point
- **discover-iac.md completes but `iac_resources.json` missing**: STOP with error: "discover-iac.md did not produce `iac_resources.json`. Verify discover-iac.md execution."
- **unify-resources.md fails**: STOP and report exact failure point
- **Output files missing after unify-resources.md**: STOP with error listing missing files
