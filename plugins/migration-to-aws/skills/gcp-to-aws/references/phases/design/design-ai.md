# Design Phase: AI Workloads (Bedrock)

> Loaded by `design.md` when `ai-workload-profile.json` exists.

**Execute ALL steps in order. Do not skip or optimize.**

---

## Step 0: Load Inputs

Read `$MIGRATION_DIR/ai-workload-profile.json`. This file contains:

- `summary.ai_source` — Source provider: `"gemini"`, `"openai"`, `"both"`, `"other"`
- `models[]` — Each detected AI model with service, capabilities, and evidence
- `integration` — SDK, frameworks, languages, gateway type, and capability summary
- `integration.gateway_type` — Gateway/router type affecting migration effort
- `infrastructure[]` — Terraform resources related to AI (may be empty)
- `current_costs` — Present only if billing data was provided

Read `$MIGRATION_DIR/preferences.json` → `ai_constraints` section (if present). This contains user preferences for AI migration (gateway type, model preference, latency requirements, budget constraints).

If `ai_constraints` is absent: use defaults (prefer managed Bedrock, no latency constraint, no budget cap).

**Load source-specific design reference based on `ai_source`:**

- If `ai_source` = `"gemini"`: load `references/design-refs/ai-gemini-to-bedrock.md`
- If `ai_source` = `"openai"`: load `references/design-refs/ai-openai-to-bedrock.md`
- If `ai_source` = `"both"`: load both files
- If `ai_source` = `"other"` or absent: load `references/design-refs/ai.md` (traditional ML rubric)

---

## Part 1: Bedrock Model Selection

For each model in `ai-workload-profile.json` → `models[]`, select the best-fit Amazon Bedrock model using the source-specific design reference loaded in Step 0.

### Source-Aware Model Mapping

**Do NOT use a hardcoded mapping table.** Instead, use the model mapping tables from the loaded design reference:

- **Gemini source** (`ai-gemini-to-bedrock.md`): Contains Gemini → Bedrock mapping tables organized by tier (Pro, Flash/Lite, Legacy) with per-1M-token pricing and competitive analysis.
- **OpenAI source** (`ai-openai-to-bedrock.md`): Contains OpenAI → Bedrock mapping tables organized by category (Flagship, Pro, GPT-4.1, GPT-4o, o-series, Legacy) with per-1M-token pricing.
- **Both sources**: Apply both mapping tables; each model maps independently.
- **Other/traditional ML** (`ai.md`): Apply SageMaker/Rekognition/Textract rubric.

For each source model, find its row in the design reference mapping table and select the recommended Bedrock model.

### Applying User Preferences

After selecting based on the mapping tables, apply overrides from `ai_constraints`:

- If `ai_constraints.ai_priority` = `"cost"` → prefer the "Winner" column from the design reference; if source provider is cheaper for this model, flag it
- If `ai_constraints.ai_priority` = `"quality"` → prefer Claude Sonnet/Opus regardless of cost
- If `ai_constraints.ai_priority` = `"speed"` → prefer Claude Sonnet (fastest to integrate, best docs)
- If `ai_constraints.ai_model_preference` = `"open-source"` → prefer Llama or Mistral models
- If `ai_constraints.ai_model_preference` = `"proprietary"` → prefer Claude models
- If `ai_constraints.ai_latency` = `"real-time"` → prefer smaller/faster models (Haiku, Nova Lite)
- If `ai_constraints.ai_latency` = `"batch"` → any model works; flag Batch API for 50% savings

### Stay-or-Migrate Assessment

For each model, evaluate whether migration to Bedrock is financially justified:

```
IF Bedrock is cheaper for this model:
  honest_assessment = "strong_migrate"

IF Bedrock is within 25% of source provider AND user priority != "cost":
  honest_assessment = "moderate_migrate"
  rationale: "Bedrock is X% more expensive, but migration justified by [ecosystem integration / model flexibility / vendor diversification]"

IF source provider is >25% cheaper AND user priority = "cost":
  honest_assessment = "weak_migrate" or "recommend_stay"
  rationale: "Source provider is X% cheaper for this model. Migration increases cost. Consider only if [ecosystem consolidation / multi-model strategy] outweighs cost increase."
```

The overall `honest_assessment` at the architecture level is the weakest assessment across all models:

- All `strong_migrate` → overall `strong_migrate`
- Any `moderate_migrate` → overall `moderate_migrate`
- Any `weak_migrate` → overall `weak_migrate`
- Any `recommend_stay` → overall `recommend_stay` (flag prominently)

### Model Comparison Table

When writing the design report, include a comparison for each mapped model:

| Dimension        | Source Provider                 | AWS Bedrock (Target)        |
| ---------------- | ------------------------------- | --------------------------- |
| Model            | (detected model_id)             | (selected Bedrock model)    |
| Provider         | (Gemini / OpenAI / etc.)        | (Anthropic / Amazon / Meta) |
| Max Context      | (source model limit)            | (Bedrock model limit)       |
| Input Price/1M   | (from design-ref mapping table) | (from design-ref mapping)   |
| Output Price/1M  | (from design-ref mapping table) | (from design-ref mapping)   |
| Price Comparison | (Winner column from design-ref) | (percentage difference)     |
| Streaming        | (yes/no from capabilities)      | (yes/no)                    |
| Function Calling | (yes/no from capabilities)      | (yes/no — tool use)         |
| Assessment       | —                               | (honest_assessment)         |

---

## Part 1B: Volume-Based Strategy

If `ai_constraints.ai_token_volume` is `"high"` (10-100M/day) or `"very_high"` (>100M/day), recommend a multi-model tiered approach instead of a single model:

### Tiering Strategy

| Volume Tier           | Strategy                                | Tiered Approach                        |
| --------------------- | --------------------------------------- | -------------------------------------- |
| Low (<1M tokens/day)  | Single best model for quality           | No tiering needed                      |
| Medium (1-10M/day)    | Present cost comparison at volume       | Optional tiering                       |
| High (10-100M/day)    | Multi-model tiered approach recommended | 3-tier routing                         |
| Very high (>100M/day) | Mandatory tiering                       | 3-tier routing with strict percentages |

### Recommended Tier Split (High/Very High Volume)

```
Tier 1 — Simple tasks (60%): Nova Micro or Llama 4 Scout
  Examples: classification, extraction, short answers, routing decisions
  Model: amazon.nova-micro-v1:0 ($0.035/$0.14 per 1M)

Tier 2 — Moderate tasks (30%): Llama 4 Maverick or Nova Pro
  Examples: summarization, moderate generation, Q&A with context
  Model: meta.llama4-maverick-17b-instruct-v1:0 ($0.24/$0.97 per 1M)

Tier 3 — Complex tasks (10%): Claude Sonnet 4.6
  Examples: reasoning, long-form generation, agentic tasks, tool use
  Model: anthropic.claude-sonnet-4-6 ($3.00/$15.00 per 1M)
```

Generate a `tiered_strategy` object in the output if volume is high or very high. Set `tiered_strategy: null` otherwise.

---

## Part 2: Feature Parity Validation

Compare GCP Vertex AI capabilities against Amazon Bedrock capabilities for each detected workload.

### Capability Matrix

| Capability        | Vertex AI                     | Amazon Bedrock                 | Parity  | Migration Notes                        |
| ----------------- | ----------------------------- | ------------------------------ | ------- | -------------------------------------- |
| Text Generation   | GenerativeModel API           | InvokeModel / Converse API     | Full    | Use Converse API for unified interface |
| Streaming         | stream_generate_content       | InvokeModelWithResponseStream  | Full    | Same SSE pattern                       |
| Function Calling  | Tool declarations in generate | Tool use in Converse API       | Full    | Different schema format                |
| Embeddings        | TextEmbeddingModel            | InvokeModel (Titan Embeddings) | Full    | Different dimension defaults           |
| Vision/Multimodal | Gemini multimodal input       | Claude multimodal messages     | Full    | Base64 or S3 URI input                 |
| Batch Processing  | BatchPredictionJob            | Batch Inference (async)        | Partial | Bedrock batch has different API        |
| Fine-tuning       | Vertex AI tuning              | Bedrock Custom Model           | Partial | Limited model selection for tuning     |
| Model Garden      | 150+ models                   | Growing model catalog          | Partial | Check specific model availability      |
| Grounding / RAG   | Vertex AI Search & RAG        | Bedrock Knowledge Bases        | Full    | Different chunking/indexing config     |
| Agents            | Vertex AI Agent Builder       | Bedrock Agents                 | Full    | Different orchestration format         |

For each capability used (from `integration.capabilities_summary`):

1. Check if Bedrock supports it → record parity status
2. If **Partial** or **None**: add to `capability_gaps[]` in output with migration notes
3. If **Full**: record as confirmed

---

## Part 3: Analyze Detected Workloads

For each model in `models[]`:

1. **Identify the workload type**:
   - Text generation (chatbot, summarization, content creation)
   - Embeddings (search, RAG, similarity)
   - Vision (image analysis, OCR)
   - Code generation (code completion, review)
   - Custom model serving (fine-tuned models)

2. **Map the integration pattern** (from `integration.pattern`):

   | GCP Pattern                            | AWS Pattern                                           | Migration Effort              |
   | -------------------------------------- | ----------------------------------------------------- | ----------------------------- |
   | `direct_sdk` (google-cloud-aiplatform) | Bedrock SDK (boto3 / @aws-sdk/client-bedrock-runtime) | Medium — SDK swap             |
   | `framework` (LangChain + Vertex)       | LangChain + Bedrock                                   | Low — change provider config  |
   | `rest_api` (Vertex REST endpoints)     | Bedrock REST API (runtime.bedrock)                    | Medium — endpoint + auth swap |
   | `mixed`                                | Mixed (match per-model)                               | Varies                        |

3. **Record the migration complexity** per model: Low / Medium / High

---

## Part 4: Infrastructure Mapping

Map GCP AI infrastructure resources to AWS equivalents.

| GCP Resource                                                 | AWS Equivalent                                                  | Notes                                            |
| ------------------------------------------------------------ | --------------------------------------------------------------- | ------------------------------------------------ |
| `google_vertex_ai_endpoint`                                  | Bedrock Model Access (no infra needed)                          | Bedrock is serverless — no endpoint provisioning |
| `google_vertex_ai_index` / `google_vertex_ai_index_endpoint` | Amazon OpenSearch Serverless (vector) or Bedrock Knowledge Base | For vector search / RAG workloads                |
| `google_vertex_ai_featurestore`                              | Amazon SageMaker Feature Store                                  | 1:1 managed feature store mapping                |
| `google_vertex_ai_dataset`                                   | S3 bucket + Bedrock training job config                         | Bedrock uses S3 for training data                |
| `google_vertex_ai_pipeline_job`                              | AWS Step Functions + Bedrock                                    | Orchestration for ML pipelines                   |

For each resource in `infrastructure[]`:

1. Find the AWS equivalent from the table above
2. If resource is a service account (`role: "supports_ai"`): map to IAM role with Bedrock permissions
3. Record the mapping with confidence = `inferred`

---

## Part 5: Code Migration Plan

For each detected integration pattern, generate a migration guide. Include patterns relevant to the detected `ai_source` and `gateway_type`.

### Pattern 1: Direct SDK — Vertex AI → boto3

**GCP (before):**

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-pro")
response = model.generate_content("Summarize this document")
```

**AWS (after):**

```python
import boto3

client = boto3.client("bedrock-runtime")
response = client.converse(
    modelId="anthropic.claude-sonnet-4-6",
    messages=[{"role": "user", "content": [{"text": "Summarize this document"}]}]
)
```

### Pattern 2: Direct SDK — OpenAI → boto3

**OpenAI (before):**

```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Summarize this document"}]
)
```

**AWS (after):**

```python
import boto3

client = boto3.client("bedrock-runtime")
response = client.converse(
    modelId="anthropic.claude-sonnet-4-6",
    messages=[{"role": "user", "content": [{"text": "Summarize this document"}]}]
)
```

### Pattern 3: LangChain Framework (Vertex AI)

**GCP (before):**

```python
from langchain_google_vertexai import ChatVertexAI

llm = ChatVertexAI(model="gemini-pro", temperature=0.7)
response = llm.invoke("Summarize this document")
```

**AWS (after):**

```python
from langchain_aws import ChatBedrock

llm = ChatBedrock(
    model_id="anthropic.claude-sonnet-4-6",
    model_kwargs={"temperature": 0.7}
)
response = llm.invoke("Summarize this document")
```

### Pattern 4: LangChain Framework (OpenAI)

**OpenAI (before):**

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0.7)
response = llm.invoke("Summarize this document")
```

**AWS (after):**

```python
from langchain_aws import ChatBedrock

llm = ChatBedrock(
    model_id="anthropic.claude-sonnet-4-6",
    model_kwargs={"temperature": 0.7}
)
response = llm.invoke("Summarize this document")
```

### Pattern 5: LLM Router (LiteLLM)

If `gateway_type` = `"llm_router"` and LiteLLM detected:

**Before:**

```python
from litellm import completion

response = completion(model="gpt-4o", messages=[{"role": "user", "content": "Hello"}])
```

**After (config change only):**

```python
from litellm import completion

response = completion(model="bedrock/anthropic.claude-sonnet-4-6", messages=[{"role": "user", "content": "Hello"}])
```

Migration effort: **1 line changed.** LiteLLM handles Bedrock auth via AWS credentials.

### Pattern 6: Embeddings

**GCP (before):**

```python
from vertexai.language_models import TextEmbeddingModel

model = TextEmbeddingModel.from_pretrained("text-embedding-004")
embeddings = model.get_embeddings(["search query"])
```

**AWS (after):**

```python
import boto3
import json

client = boto3.client("bedrock-runtime")
response = client.invoke_model(
    modelId="amazon.titan-embed-text-v2:0",
    body=json.dumps({"inputText": "search query"})
)
embedding = json.loads(response["body"].read())["embedding"]
```

### Pattern 7: Streaming

**GCP (before):**

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-pro")
for chunk in model.generate_content("Tell me a story", stream=True):
    print(chunk.text, end="")
```

**AWS (after):**

```python
import boto3

client = boto3.client("bedrock-runtime")
response = client.converse_stream(
    modelId="anthropic.claude-sonnet-4-6",
    messages=[{"role": "user", "content": [{"text": "Tell me a story"}]}]
)
for event in response["stream"]:
    if "contentBlockDelta" in event:
        print(event["contentBlockDelta"]["delta"]["text"], end="")
```

---

## Part 6: Generate Output

**File 1: `aws-design-ai.json`**

Write to `$MIGRATION_DIR/aws-design-ai.json`:

```json
{
  "metadata": {
    "phase": "design",
    "focus": "ai_workloads",
    "ai_source": "gemini",
    "bedrock_models_selected": 2,
    "timestamp": "2026-02-26T14:30:00Z"
  },
  "ai_architecture": {
    "honest_assessment": "strong_migrate",
    "tiered_strategy": null,
    "bedrock_models": [
      {
        "gcp_model_id": "gemini-pro",
        "gcp_service": "vertex_ai_generative",
        "aws_model_id": "anthropic.claude-sonnet-4-6",
        "aws_service": "Amazon Bedrock",
        "capabilities_matched": ["text_generation", "streaming"],
        "capability_gaps": [],
        "rationale": "General-purpose LLM with streaming and tool use parity",
        "migration_complexity": "medium",
        "honest_assessment": "strong_migrate",
        "source_provider_price": { "input_per_1m": 1.25, "output_per_1m": 5.00 },
        "bedrock_price": { "input_per_1m": 3.00, "output_per_1m": 15.00 },
        "price_comparison": "Source 58% cheaper on input — migration justified by ecosystem consolidation"
      },
      {
        "gcp_model_id": "text-embedding-004",
        "gcp_service": "vertex_ai_embeddings",
        "aws_model_id": "amazon.titan-embed-text-v2:0",
        "aws_service": "Amazon Bedrock",
        "capabilities_matched": ["embeddings"],
        "capability_gaps": [],
        "rationale": "Native Bedrock embedding model; dimension compatibility verified",
        "migration_complexity": "low",
        "honest_assessment": "strong_migrate",
        "source_provider_price": { "input_per_1m": 0.025, "output_per_1m": 0.0 },
        "bedrock_price": { "input_per_1m": 0.02, "output_per_1m": 0.0 },
        "price_comparison": "Bedrock 20% cheaper"
      }
    ],
    "capability_mapping": {
      "text_generation": { "parity": "full", "notes": "Use Converse API" },
      "streaming": { "parity": "full", "notes": "InvokeModelWithResponseStream" },
      "embeddings": { "parity": "full", "notes": "Titan Embeddings V2" },
      "function_calling": { "parity": "full", "notes": "Tool use in Converse API" },
      "vision": { "parity": "full", "notes": "Claude multimodal messages" },
      "batch_processing": { "parity": "partial", "notes": "Different async API pattern" }
    },
    "code_migration": {
      "primary_pattern": "direct_sdk",
      "framework": null,
      "files_to_modify": [
        {
          "file": "src/ai/client.py",
          "change_type": "sdk_swap",
          "complexity": "medium",
          "description": "Replace google-cloud-aiplatform with boto3 bedrock-runtime"
        }
      ],
      "dependency_changes": {
        "remove": ["google-cloud-aiplatform", "vertexai"],
        "add": ["boto3"]
      }
    },
    "infrastructure": [
      {
        "gcp_resource": "google_vertex_ai_endpoint.main",
        "aws_equivalent": "Bedrock Model Access (serverless)",
        "notes": "No endpoint provisioning needed — Bedrock is fully managed",
        "confidence": "inferred"
      }
    ],
    "services_to_migrate": [
      {
        "gcp_service": "Vertex AI (Generative)",
        "aws_service": "Amazon Bedrock",
        "migration_effort": "medium",
        "notes": "SDK swap + model ID mapping"
      },
      {
        "gcp_service": "Vertex AI (Embeddings)",
        "aws_service": "Amazon Bedrock",
        "migration_effort": "low",
        "notes": "Provider swap in embedding client"
      }
    ]
  }
}
```

**File 2: `aws-design-ai-report.md`**

```markdown
# AI Workload Design Report

## Overview

Mapped X GCP AI models to Y Amazon Bedrock models.
Integration pattern: [direct_sdk|framework|rest_api|mixed]

## Model Mappings

### [gcp_model_id] → [aws_model_id]

- GCP Service: [service]
- AWS Service: Amazon Bedrock
- Capabilities matched: [list]
- Capability gaps: [list or "None"]
- Migration complexity: [low|medium|high]
- Rationale: [rationale]

[repeat per model]

## Feature Parity Summary

| Capability                | Parity | Notes |
| ------------------------- | ------ | ----- |
| [from capability_mapping] |        |       |

## Code Migration Guide

### Files to modify

[from code_migration.files_to_modify]

### Dependency changes

- Remove: [list]
- Add: [list]

### Code examples

[Include relevant patterns from Part 5 based on detected integration pattern]

## Infrastructure Changes

[from infrastructure mappings]
```

## Output Validation Checklist

### aws-design-ai.json

- `metadata.ai_source` matches `summary.ai_source` from `ai-workload-profile.json`
- `metadata.bedrock_models_selected` matches length of `ai_architecture.bedrock_models`
- `ai_architecture.honest_assessment` is one of: `"strong_migrate"`, `"moderate_migrate"`, `"weak_migrate"`, `"recommend_stay"`
- `ai_architecture.tiered_strategy` is `null` for low/medium volume, or an object with `tiers[]` for high/very-high
- Every model in `ai-workload-profile.json` → `models[]` has a corresponding entry in `bedrock_models`
- Every `bedrock_models[]` entry has `gcp_model_id`, `aws_model_id`, `capabilities_matched`, `capability_gaps`, `honest_assessment`
- Every `bedrock_models[]` entry has `source_provider_price`, `bedrock_price`, and `price_comparison`
- `capability_mapping` covers every capability from `integration.capabilities_summary` that was `true`
- `code_migration.primary_pattern` matches `integration.pattern` from the input
- All `migration_complexity` values are `"low"`, `"medium"`, or `"high"`
- All model IDs use current Bedrock identifiers (e.g., `anthropic.claude-sonnet-4-6`, not legacy IDs)
- Output is valid JSON

### aws-design-ai-report.md

- Overview section lists model count and integration pattern
- Every model has a mapping section
- Feature parity table is present
- Code migration guide includes at least one pattern example
