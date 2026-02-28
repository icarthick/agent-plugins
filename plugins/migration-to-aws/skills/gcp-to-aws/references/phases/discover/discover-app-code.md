# Discover Phase: App Code Discovery

> Self-contained application code discovery sub-file. Scans for source code, detects GCP SDK imports, and infers resources.
> If no source code files are found, exits cleanly with no output.

**Status**: Not yet implemented (v1.1 feature). Steps 1-3 below describe expected behavior when implemented.

## Step 0: Self-Scan for Source Code

Recursively scan the target directory for source code and dependency manifests:

**Source code:**

- `**/*.py` (Python)
- `**/*.js`, `**/*.ts` (JavaScript/TypeScript)
- `**/*.go` (Go)

**Dependency manifests:**

- `requirements.txt`, `setup.py`, `pyproject.toml` (Python)
- `package.json` (JavaScript)
- `go.mod` (Go)

**Exit gate:** If NO source code files or dependency manifests are found, **exit cleanly**. Return no output artifacts. Other sub-discovery files may still produce artifacts.

## Step 1: Detect GCP SDK Imports (v1.1+)

Scan source files for GCP service imports:

- `google.cloud` (Python: `from google.cloud import ...`)
- `@google-cloud/` (JS/TS: `import ... from '@google-cloud/...'`)
- `cloud.google.com/go` (Go: `import "cloud.google.com/go/..."`)

For each import found, record:

- `file_path` — Source file containing the import
- `import_statement` — The actual import line
- `inferred_gcp_service` — Which GCP service this maps to
- `confidence` — 0.60-0.80 (lower than IaC since inferring from code)

## Step 2: Infer Resources from Code (v1.1+)

Map SDK imports to likely GCP resources:

| Import pattern               | Inferred GCP resource |
| ---------------------------- | --------------------- |
| `google.cloud.storage`       | Cloud Storage bucket  |
| `google.cloud.firestore`     | Firestore database    |
| `google.cloud.pubsub`        | Pub/Sub topic         |
| `google.cloud.bigquery`      | BigQuery dataset      |
| `google.cloud.sql`           | Cloud SQL instance    |
| `google.cloud.run`           | Cloud Run service     |
| `google.cloud.functions`     | Cloud Functions       |
| `google.cloud.secretmanager` | Secret Manager        |

## Step 3: Write Output (v1.1+)

Write `$MIGRATION_DIR/app_code_resources.json`:

```json
{
  "app_code_resources": [
    {
      "service": "Cloud Run",
      "detected_imports": ["google.cloud.storage", "google.cloud.firestore"],
      "workload_type": "web",
      "evidence": "code analysis",
      "confidence": 0.92
    }
  ]
}
```

## Scope Boundary

**This phase covers Discover & Analysis ONLY.**

FORBIDDEN — Do NOT include ANY of:

- AWS service names, recommendations, or equivalents
- Migration strategies, phases, or timelines
- Terraform generation for AWS
- Cost estimates or comparisons
- Effort estimates

**Your ONLY job: Inventory what exists in GCP. Nothing else.**
