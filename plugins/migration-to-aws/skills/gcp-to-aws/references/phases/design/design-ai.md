# Design Phase: AI Workloads (Bedrock)

> Loaded by `design.md` when `ai-workload-profile.json` exists.

**Execute ALL steps in order. Do not skip or optimize.**

---

## Step 0: Load Inputs

Read `$MIGRATION_DIR/ai-workload-profile.json`. This file contains:

- `models[]` — Each detected AI model with service, capabilities, and evidence
- `integration` — SDK, frameworks, languages, and capability summary
- `infrastructure[]` — Terraform resources related to AI (may be empty)
- `current_costs` — Present only if billing data was provided

Read `$MIGRATION_DIR/preferences.json` → `ai_constraints` section (if present). This contains user preferences for AI migration (model preference, latency requirements, budget constraints).

If `ai_constraints` is absent: use defaults (prefer managed Bedrock, no latency constraint, no budget cap).

---

## Part 1: Bedrock Model Selection

For each model in `ai-workload-profile.json` → `models[]`, select the best-fit Amazon Bedrock model.

### Decision Tree (by token volume)

Evaluate based on `capabilities_used` and `usage_context`:

| GCP Model Pattern                          | Capabilities                                 | Bedrock Model                 | Rationale                                      |
| ------------------------------------------ | -------------------------------------------- | ----------------------------- | ---------------------------------------------- |
| `gemini-pro` / `gemini-1.5-pro`            | text_generation, streaming, function_calling | Claude 3.5 Sonnet (Anthropic) | Best general-purpose; supports tools/streaming |
| `gemini-1.5-flash` / `gemini-flash`        | text_generation (high throughput)            | Claude 3.5 Haiku (Anthropic)  | Lower cost, faster latency for high-volume     |
| `gemini-ultra` / `gemini-1.0-ultra`        | text_generation (complex reasoning)          | Claude 3.5 Sonnet (Anthropic) | Complex reasoning parity                       |
| `text-bison` / `chat-bison`                | text_generation (legacy)                     | Amazon Titan Text             | Cost-effective for simple text tasks           |
| `text-embedding-*` / `textembedding-gecko` | embeddings                                   | Amazon Titan Embeddings V2    | Native Bedrock embedding model                 |
| `gemini-pro-vision` / `gemini-*` + vision  | vision, image_understanding                  | Claude 3.5 Sonnet (Anthropic) | Multimodal support                             |
| `code-bison` / `codechat-bison`            | code_generation                              | Claude 3.5 Sonnet (Anthropic) | Superior code generation                       |
| `imagen-*`                                 | image_generation                             | Amazon Titan Image Generator  | Native Bedrock image generation                |

### Model Comparison Table

When writing the design report, include a comparison for each mapped model:

| Dimension        | GCP (Source)                      | AWS Bedrock (Target)        |
| ---------------- | --------------------------------- | --------------------------- |
| Model            | (detected model_id)               | (selected Bedrock model)    |
| Provider         | Google / Vertex AI                | (Anthropic / Amazon / etc.) |
| Max Context      | (GCP model limit)                 | (Bedrock model limit)       |
| Streaming        | (yes/no from capabilities)        | (yes/no)                    |
| Function Calling | (yes/no from capabilities)        | (yes/no — tool use)         |
| Embeddings       | (yes/no)                          | (yes/no)                    |
| Estimated Cost   | (from current_costs if available) | (defer to Estimate phase)   |

### Override Rules

- If `ai_constraints.model_preference` exists → use the user's preferred model family
- If `ai_constraints.latency_requirement` = "low" → prefer Haiku over Sonnet
- If `ai_constraints.budget_cap` exists → prefer lower-cost models (Haiku, Titan)

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

For each detected integration pattern, generate a migration guide.

### Pattern 1: Direct SDK (google-cloud-aiplatform → boto3)

**GCP (before):**

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-pro")
response = model.generate_content("Summarize this document")
```

**AWS (after):**

```python
import boto3
import json

client = boto3.client("bedrock-runtime")
response = client.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[{"role": "user", "content": [{"text": "Summarize this document"}]}]
)
```

### Pattern 2: LangChain Framework

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
    model_id="anthropic.claude-3-5-sonnet-20241022-v2:0",
    model_kwargs={"temperature": 0.7}
)
response = llm.invoke("Summarize this document")
```

### Pattern 3: Embeddings

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

### Pattern 4: Streaming

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
import json

client = boto3.client("bedrock-runtime")
response = client.converse_stream(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
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
    "bedrock_models_selected": 2,
    "timestamp": "2026-02-26T14:30:00Z"
  },
  "ai_architecture": {
    "bedrock_models": [
      {
        "gcp_model_id": "gemini-pro",
        "gcp_service": "vertex_ai_generative",
        "aws_model_id": "anthropic.claude-3-5-sonnet-20241022-v2:0",
        "aws_service": "Amazon Bedrock",
        "capabilities_matched": ["text_generation", "streaming"],
        "capability_gaps": [],
        "rationale": "General-purpose LLM with streaming and tool use parity",
        "migration_complexity": "medium"
      },
      {
        "gcp_model_id": "text-embedding-004",
        "gcp_service": "vertex_ai_embeddings",
        "aws_model_id": "amazon.titan-embed-text-v2:0",
        "aws_service": "Amazon Bedrock",
        "capabilities_matched": ["embeddings"],
        "capability_gaps": [],
        "rationale": "Native Bedrock embedding model; dimension compatibility verified",
        "migration_complexity": "low"
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

- `metadata.bedrock_models_selected` matches length of `ai_architecture.bedrock_models`
- Every model in `ai-workload-profile.json` → `models[]` has a corresponding entry in `bedrock_models`
- Every `bedrock_models[]` entry has `gcp_model_id`, `aws_model_id`, `capabilities_matched`, `capability_gaps`
- `capability_mapping` covers every capability from `integration.capabilities_summary` that was `true`
- `code_migration.primary_pattern` matches `integration.pattern` from the input
- All `migration_complexity` values are `"low"`, `"medium"`, or `"high"`
- Output is valid JSON

### aws-design-ai-report.md

- Overview section lists model count and integration pattern
- Every model has a mapping section
- Feature parity table is present
- Code migration guide includes at least one pattern example
