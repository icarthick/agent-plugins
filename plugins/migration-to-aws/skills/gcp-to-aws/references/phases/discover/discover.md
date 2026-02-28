# Phase 1: Discover GCP Resources

Lightweight orchestrator that detects available source types and delegates to domain-specific discoverers.
**Execute ALL steps in order. Do not skip or deviate.**

## Step 0: Initialize Migration State

1. Check for existing `.migration/` directory at the project root.
   - **If existing runs found:** List them with their phase status and ask:
     - `[A] Resume: Continue with [latest run]`
     - `[B] Fresh: Create new migration run`
     - `[C] Cancel`
   - **If resuming:** Set `$MIGRATION_DIR` to the selected run's directory. Read its `.phase-status.json` and skip to the appropriate phase per the State Machine in SKILL.md.
   - **If fresh or no existing runs:** Continue to step 2.
2. Create `.migration/[MMDD-HHMM]/` directory (e.g., `.migration/0226-1430/`) using current timestamp (MMDD = month/day, HHMM = hour/minute). Set `$MIGRATION_DIR` to this new directory.
3. Create `.migration/.gitignore` file (if not already present) with exact content:

   ```
   # Auto-generated migration state (temporary, should not be committed)
   *
   !.gitignore
   ```

   This prevents accidental commits of migration artifacts.

4. Write `.phase-status.json` with exact schema:

   ```json
   {
     "migration_id": "[MMDD-HHMM]",
     "started_at": "[ISO 8601 timestamp]",
     "last_updated": "[ISO 8601 timestamp]",
     "project_directory": "[absolute path to scanned project]",
     "input_files_detected": {},
     "discovery_outputs": [],
     "phases": {
       "discover": { "status": "in_progress", "timestamp": null, "outputs": [] },
       "clarify": { "status": "pending", "timestamp": null, "outputs": [] },
       "design": { "status": "pending", "timestamp": null, "outputs": [] },
       "estimate": { "status": "pending", "timestamp": null, "outputs": [] },
       "execute": { "status": "pending", "timestamp": null, "outputs": [] }
     }
   }
   ```

   **Schema reference:** See `references/shared/output-schema.md` > ".phase-status.json" for the canonical schema.

5. Confirm both `.migration/.gitignore` and `.phase-status.json` exist before proceeding to Step 1.

## Step 1: Scan for Available Source Types

1. Recursively scan working directory for source files:
   - **Terraform**: `.tf`, `.tfvars`, `.tfstate` files (primary v1.0)
   - **Billing** (v1.2+): GCP billing CSV/JSON exports (skip if not available)
   - **App code** (v1.1+): Python/Node/Go with `google.cloud.*` imports (skip if not available)
2. **IF zero sources found**: STOP and output: "No GCP sources detected (no `.tf` files, billing exports, or app code). Provide at least one source type and try again."
3. Report detected sources to user (e.g., "Found Terraform files in: [list]")

## Step 2: Invoke Terraform Discoverer (v1.0 — REQUIRED)

**This step is MANDATORY for v1.0. Produces final outputs directly.**

1. **IF Terraform files found** (from Step 1):
   - Load `references/phases/discover/discover-iac.md`
   - **WAIT for completion**: Confirm BOTH output files exist in `$MIGRATION_DIR/`:
     - `gcp-resource-inventory.json` (REQUIRED)
     - `gcp-resource-clusters.json` (REQUIRED)
   - **Validate schemas**: Confirm files contain all required fields
   - Proceed to Step 3
2. **IF Terraform files NOT found**:
   - Skip this step. Terraform is required for v1.0. If other sources available, defer to v1.1/v1.2 handling.

## Step 3: Update Phase Status

1. Update `.phase-status.json`:
   - Set `input_files_detected` with counts from Step 1 (e.g., `{"terraform_files": 12, "terraform_lines": 850}`)
   - Set `discovery_outputs` to the list of output files produced (e.g., `["gcp-resource-inventory.json", "gcp-resource-clusters.json"]`)
   - Set `phases.discover.status` to `"completed"`
   - Set `phases.discover.timestamp` to current ISO 8601 timestamp
   - Set `phases.discover.outputs` to same list as `discovery_outputs`
   - Update `last_updated` to current timestamp

2. Output to user: "Discover phase complete. Discovered X total resources across Y clusters. Proceeding to Phase 2: Clarify."

## Output Files ONLY

**Discover phase produces EXACTLY 2 files in `.migration/[MMDD-HHMM]/`:**

1. `gcp-resource-inventory.json` (REQUIRED)
2. `gcp-resource-clusters.json` (REQUIRED)

**No other files should be created:**

- ❌ README.md
- ❌ discovery-summary.md
- ❌ EXECUTION_REPORT.txt
- ❌ discovery-log.md
- ❌ Any documentation or report files

All user communication via output messages only.

## Error Handling

- **Missing `.migration` directory**: Create it (Step 0)
- **Missing `.migration/.gitignore`**: Create it automatically (Step 0) — prevents accidental commits
- **No Terraform files found**: STOP with error message (Step 1). Terraform is required for v1.0.
- **discover-iac.md fails**: STOP and report exact failure point
- **discover-iac.md completes but output files missing**: STOP with error listing missing files
- **Output file validation fails**: STOP and report schema errors
- **Extra files created (README, reports, etc.)**: Failure. Discover must produce ONLY the two JSON files.

## Future Versions (v1.1+, v1.2+)

**v1.1 (App Code Discovery):**

- Implement `discover-app-code.md` to scan Python/Node/Go imports
- Merge strategy with Terraform results: TBD

**v1.2 (Billing Discovery):**

- Implement `discover-billing.md` to parse GCP billing exports
- Merge strategy with other sources: TBD

## Scope Boundary

**This phase covers Discover & Analysis ONLY.**

FORBIDDEN — Do NOT include ANY of:
- AWS service names, recommendations, or equivalents
- Migration strategies, phases, or timelines
- Terraform generation for AWS
- Cost estimates or comparisons
- Effort estimates

**Your ONLY job: Inventory what exists in GCP. Nothing else.**
