# Discover Phase: App Code Discovery

> Self-contained application code discovery sub-file. Scans for source code, detects GCP SDK imports, infers resources, flags AI signals, and if AI confidence >= 70%, extracts detailed AI workload information and generates `ai-workload-profile.json`.
> If no source code files are found, exits cleanly with no output.

<!-- TODO: Dead-end case — if app code is the only data source and no AI is detected
     (confidence < 70%), this file produces no artifacts. The user must provide
     additional data (Terraform or billing) to proceed with migration planning. -->

**Execute ALL steps in order. Do not skip or optimize.**

---

## Step 0: Self-Scan for Source Code

Recursively scan the entire target directory tree for source code and dependency manifests:

**Source code:**

- `**/*.py` (Python)
- `**/*.js`, `**/*.ts`, `**/*.jsx`, `**/*.tsx` (JavaScript/TypeScript)
- `**/*.go` (Go)
- `**/*.java` (Java)
- `**/*.scala`, `**/*.kotlin`, `**/*.rust` (Other)

**Dependency manifests:**

- `requirements.txt`, `setup.py`, `pyproject.toml`, `Pipfile` (Python)
- `package.json`, `package-lock.json`, `yarn.lock` (JavaScript)
- `go.mod`, `go.sum` (Go)
- `pom.xml`, `build.gradle` (Java)

**Exit gate:** If NO source code files or dependency manifests are found, **exit cleanly**. Return no output artifacts. Other sub-discovery files may still produce artifacts.

---

## Step 1: Detect GCP SDK Imports

Scan source files for GCP service imports:

- `google.cloud` (Python: `from google.cloud import ...`)
- `@google-cloud/` (JS/TS: `import ... from '@google-cloud/...'`)
- `cloud.google.com/go` (Go: `import "cloud.google.com/go/..."`)
- `com.google.cloud` (Java: `import com.google.cloud.*`)

For each import found, record:

- `file_path` — Source file containing the import
- `import_statement` — The actual import line
- `inferred_gcp_service` — Which GCP service this maps to
- `confidence` — 0.60-0.80 (lower than IaC since we're inferring from code, not reading config)

---

## Step 2: Infer Resources from Code

Map SDK imports to likely GCP resources. These are inferred — they supplement IaC evidence but at lower confidence:

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
| `redis` / `ioredis`          | Redis instance        |

Confidence for inferred resources: 0.60-0.80 (inferring existence, not reading infrastructure config).

---

## Step 3: Flag AI Signals

Scan source code files and dependency manifests for AI-relevant patterns. For each match, record the pattern, file location, and confidence score.

| Pattern                       | What to look for                                                                                                                                          | Confidence |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| 2.1 Google AI Platform SDK    | Imports: `google.cloud.aiplatform` (Python), `@google-cloud/aiplatform` (JS), `cloud.google.com/go/aiplatform` (Go), `com.google.cloud.aiplatform` (Java) | 95%        |
| 2.2 BigQuery ML SDK           | `google.cloud.bigquery` + ML operations; SQL containing `CREATE MODEL` or `ML.*`                                                                          | 85%        |
| 2.3 LLM SDKs                  | Imports: `google.generativeai`, `anthropic`, `openai`, other LLM SDKs                                                                                     | 98%        |
| 2.4 Document/Vision/Speech AI | Imports: `google.cloud.documentai*`, `google.cloud.vision*`, `google.cloud.speech*`, `google.cloud.translate*`, `google.cloud.dialogflow*`                | 90%        |
| 2.5 Embeddings & RAG          | `langchain` + `VertexAIEmbeddings`; `llama_index` + Vertex AI; vector database usage with embeddings                                                      | 85%        |

Also check dependency manifests for AI/ML SDK dependencies:

- `google-cloud-aiplatform`
- `google-cloud-vertexai`
- `google-cloud-bigquery-ml`
- `google-cloud-language`
- `google-cloud-vision`
- `google-cloud-speech`

---

## Step 4: AI Detection Gate

Compute overall AI confidence from all signals found in Step 3:

```text
IF (Multiple strong signals: LLM SDK + AI Platform SDK)
  THEN confidence = 95%+ (very high)

IF (Any one strong signal: LLM SDK, AI Platform SDK, Generative AI imports)
  THEN confidence = 90%+ (high)

IF (Weaker signals only: BigQuery ML, variable patterns)
  THEN confidence = 60-70% (medium)

IF (No signals found)
  THEN confidence = 0% (no AI workload detected)
```

### False Positive Checklist

Before finalizing AI detection, verify signals are genuine:

- **BigQuery alone is not AI** — Require `google_bigquery_ml_model` or `CREATE MODEL` SQL. A `google_bigquery_dataset` by itself is standard analytics.
- **Vector database alone is not AI** — Require embeddings library imports (langchain, llama-index). A Firestore/Datastore by itself is a regular database.
- **Dead/commented-out code excluded** — Only count active code.

**Exit gate:** If overall AI confidence < 70%, **exit cleanly**. Record the signals found and confidence for reference, but do not generate `ai-workload-profile.json`. The inferred resources from Steps 1-2 remain available for other sub-files (e.g., discover-iac.md may use them for evidence merge).

**If confidence >= 70%**, continue to Steps 5-8 below.

---

## Step 5: Extract AI Model Details

For each AI signal found during detection, extract model-level details:

**From application code:**

Scan files that contained AI signals for specific model information:

- **Model identifiers** — Look for model name strings passed to constructors or API calls:
  - `GenerativeModel("gemini-pro")` -> model_id: `"gemini-pro"`
  - `aiplatform.Model.list(filter='display_name="my-model"')` -> model_id: `"my-model"`
  - `TextEmbeddingModel.from_pretrained("text-embedding-004")` -> model_id: `"text-embedding-004"`

- **Capabilities used** — Determine from API calls and method signatures:
  - `text_generation`: `generate_content()`, `predict()`, `messages.create()`
  - `streaming`: `generate_content(stream=True)`, `stream()`, async iterators over responses
  - `function_calling`: `tools=` parameter, `function_declarations=`, tool definitions
  - `vision`: image bytes, image URLs, or multimodal content passed as input
  - `embeddings`: `TextEmbeddingModel`, `VertexAIEmbeddings`, embedding API calls
  - `batch_processing`: batch predict calls, bulk processing patterns

- **Usage context** — Infer from the combination of:
  - File path and module name (e.g., `src/search/indexer.py` -> search/indexing)
  - Class and function names (e.g., `RecommendationEngine.get_recommendations` -> recommendations)
  - Import statements (e.g., `from langchain.embeddings` -> embeddings/RAG)
  - Surrounding code context (what data flows in and out of the AI call)

**From Terraform (if IaC discovery also ran):**

- Vertex AI endpoint configurations (display name, location, machine type)
- Model deployment settings (traffic split, scaling)
- Resource addresses for cross-referencing with code

**From billing data (if billing discovery also ran):**

- Which AI services have billing line items
- Monthly spend per AI service

---

## Step 6: Map Integration Patterns

Determine how the application integrates with AI services:

- **Primary SDK**: Which Google AI SDK is used
  - `google-cloud-aiplatform` (Vertex AI Platform SDK)
  - `google-generativeai` (Gemini API)
  - `vertexai` (Vertex AI SDK for Python)
  - `@google-cloud/aiplatform` (Node.js)
  - `cloud.google.com/go/aiplatform` (Go)

- **SDK version**: Extract from dependency files (`requirements.txt`, `package.json`, `go.mod`, etc.)

- **Frameworks**: Does the code use orchestration frameworks?
  - LangChain (`from langchain...`)
  - LlamaIndex (`from llama_index...`)
  - Semantic Kernel
  - No framework (raw SDK calls)

- **Languages**: Which programming languages contain AI code?

- **Integration pattern**: Classify as one of:
  - `direct_sdk` — Direct Google SDK calls (e.g., `aiplatform.init()`, `model.predict()`)
  - `framework` — Via LangChain, LlamaIndex, or similar
  - `rest_api` — Raw HTTP calls to Vertex AI endpoints
  - `mixed` — Combination of the above

Build the **capabilities summary** — a flat boolean map of which AI capabilities are actively used across all detected models:

```json
{
  "text_generation": true,
  "streaming": true,
  "function_calling": false,
  "vision": false,
  "embeddings": true,
  "batch_processing": false
}
```

A capability is `true` only if there is evidence from code analysis that it is actively used.

---

## Step 7: Capture Supporting Infrastructure

**Only if Terraform files were found (IaC discovery also ran)**, extract AI-related infrastructure resources:

- **AI resources**: `google_vertex_ai_endpoint`, `google_vertex_ai_model`, `google_vertex_ai_featurestore`, `google_vertex_ai_index`, `google_vertex_ai_tensorboard`, etc.
- **Supporting resources** that serve AI primaries:
  - Service accounts used by AI endpoints
  - VPC connectors attached to AI services
  - Secret Manager entries referenced by AI code (API keys, credentials)
  - Cloud Storage buckets used for model artifacts or training data

For each resource, capture: `address`, `type`, `file`, and relevant `config`.

If no Terraform files were provided, set `infrastructure: []`.

---

## Step 8: Generate ai-workload-profile.json

Generate output following schema in `references/shared/output-schema.md` > "ai-workload-profile.json" section.

**CRITICAL field names** — use EXACTLY these keys:

- `model_id` (not model_name, name)
- `service` (not service_type, gcp_service)
- `detected_via` (not detection_method, source)
- `capabilities_used` (not capabilities, features)
- `usage_context` (not description, purpose)
- `pattern` in integration (not integration_type, method)
- `capabilities_summary` (not capabilities, feature_flags)

**Conditional sections:**

- `current_costs` — Include ONLY if billing data was provided (billing discovery ran). Omit entirely if no billing data.
- `infrastructure` — Set to `[]` if no Terraform files were provided (IaC discovery did not run).

After generating the output file, update `$MIGRATION_DIR/.phase-status.json`: set `phases.discover.status` to `"completed"`, record `timestamp` and `outputs`.

---

## Output Validation Checklist — ai-workload-profile.json

- `metadata.sources_analyzed` reflects which data sources were actually provided
- `summary.overall_confidence` matches the detection confidence from Step 4
- `summary.total_models_detected` matches the length of `models` array
- Every entry in `models` has `model_id`, `service`, `detected_via`, `evidence`, `capabilities_used`, and `usage_context`
- `models[].detected_via` only contains sources that were actually analyzed (`"code"`, `"terraform"`, `"billing"`)
- `models[].evidence` array has at least one entry per source listed in `detected_via`
- `models[].capabilities_used` only lists capabilities with evidence from code analysis
- `integration.capabilities_summary` is consistent with the union of all `models[].capabilities_used`
- `infrastructure` is empty array `[]` if no Terraform was provided
- `current_costs` section is present ONLY if billing data was provided; omitted entirely otherwise
- `detection_signals` matches the signals found in Step 3
- All field names use EXACT required keys

---

## Design Phase Integration

The Design phase (`references/phases/design.md`) uses `ai-workload-profile.json`:

1. **`models`** — Determines which Bedrock models to recommend via the model selection decision tree
2. **`integration.capabilities_summary`** — Validates Bedrock feature parity (e.g., if `function_calling` is `true`, selected Bedrock model must support tool use)
3. **`integration.pattern`** and **`integration.primary_sdk`** — Determines code migration guidance (direct SDK swap vs framework provider swap vs REST endpoint change)
4. **`integration.frameworks`** — If LangChain is used, migration may be simpler (swap provider, keep chains)
5. **`infrastructure`** — Identifies supporting AWS resources needed (IAM roles, VPC config)
6. **`current_costs.monthly_ai_spend`** — Baseline for cost comparison in estimate phase

---

## Scope Boundary

**This phase covers Discover & Analysis ONLY.**

FORBIDDEN — Do NOT include ANY of:

- AWS service names, recommendations, or equivalents
- Migration strategies, phases, or timelines
- Terraform generation for AWS
- Cost estimates or comparisons
- Effort estimates

**Your ONLY job: Inventory what exists in GCP. Nothing else.**
