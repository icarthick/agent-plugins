# Phase 1: Discover GCP Resources

Lightweight orchestrator that delegates to domain-specific discoverers. Each sub-discovery file is self-contained — it scans for its own input, processes what it finds, and exits cleanly if nothing is relevant.
**Execute ALL steps in order. Do not skip or deviate.**

## Sub-Discovery Files

- **discover-iac.md** → `gcp-resource-inventory.json` + `gcp-resource-clusters.json` (if Terraform found)
- **discover-app-code.md** → `ai-workload-profile.json` (if source code with AI signals found)
- **discover-billing.md** → `billing-profile.json` (if billing data found)

Multiple artifacts can be produced in a single run — they are not mutually exclusive.

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
       "generate": { "status": "pending", "timestamp": null, "outputs": [] },
       "feedback": { "status": "pending", "timestamp": null, "outputs": [] }
     }
   }
   ```

5. Confirm both `.migration/.gitignore` and `.phase-status.json` exist before proceeding to Step 1.

## Step 1: Load Output Schemas

Load `references/shared/output-schema.md` once now. Sub-discovery files reference these schemas when writing output files — do not reload it from sub-files.

## Step 2: Scan for Input Sources and Run Sub-Discoveries

Scan the project directory for each input type. Only load sub-discovery files when their input files are present.

**2a. Check for Terraform files:**
Glob for: `**/*.tf`, `**/*.tfvars`, `**/*.tfstate`, `**/.terraform.lock.hcl`

- If found → Load `references/phases/discover/discover-iac.md`
- If not found → Skip. Log: "No Terraform files found — skipping IaC discovery."

**2b. Check for source code / dependency manifests:**
Glob for: `**/*.py`, `**/*.js`, `**/*.ts`, `**/*.jsx`, `**/*.tsx`, `**/*.go`, `**/*.java`, `**/*.scala`, `**/*.kotlin`, `**/*.rust`, `**/requirements.txt`, `**/setup.py`, `**/pyproject.toml`, `**/Pipfile`, `**/package.json`, `**/go.mod`, `**/pom.xml`, `**/build.gradle`

- If found → Load `references/phases/discover/discover-app-code.md`
- If not found → Skip. Log: "No source code found — skipping app code discovery."

**2c. Check for billing data:**
Glob for: `**/*billing*.csv`, `**/*billing*.json`, `**/*cost*.csv`, `**/*cost*.json`, `**/*usage*.csv`, `**/*usage*.json`

- If found → Load `references/phases/discover/discover-billing.md`
- If not found → Skip. Log: "No billing files found — skipping billing discovery."

**If NONE of the three checks found files**: STOP and output: "No GCP sources detected. Provide at least one source type (Terraform files, application code, or billing exports) and try again."

## Step 3: Check Outputs

After all loaded sub-discoveries complete, check what artifacts were produced in `$MIGRATION_DIR/`:

1. Check for output files:
   - `gcp-resource-inventory.json` — IaC discovery succeeded
   - `gcp-resource-clusters.json` — IaC discovery produced clusters
   - `ai-workload-profile.json` — App code discovery detected AI workloads
   - `billing-profile.json` — Billing data parsed
2. **If NO artifacts were produced** (sub-discoveries ran but produced no output): STOP and output: "Discovery ran but produced no artifacts. Check that your input files contain valid GCP resources and try again."
3. Record produced artifacts in `.phase-status.json` → `discovery_outputs`.

## Step 4: Update Phase Status

1. Update `.phase-status.json`:
   - Set `input_files_detected` with counts from sub-discoveries (e.g., `{"terraform_files": 12, "terraform_lines": 850, "app_code_languages": ["python"]}`)
   - Set `discovery_outputs` to the list of output files produced (e.g., `["gcp-resource-inventory.json", "gcp-resource-clusters.json"]`)
   - Set `phases.discover.status` to `"completed"`
   - Set `phases.discover.timestamp` to current ISO 8601 timestamp
   - Set `phases.discover.outputs` to same list as `discovery_outputs`
   - Update `last_updated` to current timestamp

2. Output to user: "Discover phase complete. Discovered X total resources across Y clusters. Proceeding to Phase 2: Clarify."

## Output Files

**Discover phase writes files to `$MIGRATION_DIR/`. Possible outputs (depending on what sub-discoverers find):**

1. `gcp-resource-inventory.json` — from discover-iac.md
2. `gcp-resource-clusters.json` — from discover-iac.md
3. `ai-workload-profile.json` — from discover-app-code.md
4. `billing-profile.json` — from discover-billing.md

**No other files should be created:**

- No README.md
- No discovery-summary.md
- No EXECUTION_REPORT.txt
- No discovery-log.md
- No documentation or report files

All user communication via output messages only.

## Error Handling

- **Missing `.migration` directory**: Create it (Step 0)
- **Missing `.migration/.gitignore`**: Create it automatically (Step 0) — prevents accidental commits
- **No input files found for any sub-discoverer**: STOP with error message (Step 2)
- **Sub-discoveries ran but produced no artifacts**: STOP with error message (Step 3)
- **Sub-discoverer fails**: STOP and report exact failure point and which sub-discoverer failed
- **Output file validation fails**: STOP and report schema errors
- **Extra files created (README, reports, etc.)**: Failure. Discover must produce ONLY the JSON artifact files.

## Scope Boundary

**This phase covers Discover & Analysis ONLY.**

FORBIDDEN — Do NOT include ANY of:

- AWS service names, recommendations, or equivalents
- Migration strategies, phases, or timelines
- Terraform generation for AWS
- Cost estimates or comparisons
- Effort estimates

**Your ONLY job: Inventory what exists in GCP. Nothing else.**
