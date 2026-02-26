# Phase 1: Discover GCP Resources

Lightweight orchestrator that detects available source types and delegates to domain-specific discoverers.

## Step 0: Initialize Migration State

Create `.migration/[MMDD-HHMM]/` directory (e.g., `.migration/0226-1430/`) using current timestamp.

Write initial `.phase-status.json`:

```json
{
  "phase": "discover",
  "status": "in-progress",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

## Step 1: Scan for Available Source Types

Recursively scan working directory for:

- **Terraform**: `.tf`, `.tfvars`, `.tfstate` files
- **Billing** (v1.2+): GCP billing CSV/JSON exports
- **App code** (v1.1+): Python/Node/Go with `google.cloud.*` imports

Report which sources are detected. If **zero** sources found: **STOP**. Output: "No GCP sources detected (no `.tf` files, billing exports, or app code). Provide at least one source type and try again."

## Step 2: Delegate to Domain-Specific Discoverers

For each source type found, invoke the corresponding discoverer:

- **IF Terraform found** → Call `discover-iac.md`
  - Produces: `iac_resources.json` (intermediate file)
- **IF Billing found** → Call `discover-billing.md` (skip if v1.2 not available; currently stub)
  - Produces: `billing_resources.json` (intermediate file)
- **IF App code found** → Call `discover-app-code.md` (skip if v1.1 not available; currently stub)
  - Produces: `app_code_resources.json` (intermediate file)

## Step 3: Combine Domain Outputs

Call `unify-resources.md` with all intermediate files found.

Produces:
- `gcp-resource-inventory.json` (final combined inventory)
- `gcp-resource-clusters.json` (if IaC path taken)

## Step 4: Update Phase Status

Update `.phase-status.json`:

```json
{
  "phase": "clarify",
  "status": "completed",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

Output to user: "Discovered X resources. Proceeding to Phase 2: Clarify."
