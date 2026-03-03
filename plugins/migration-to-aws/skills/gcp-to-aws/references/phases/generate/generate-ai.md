# Generate Phase: AI Migration Plan

> Loaded by generate.md when estimation-ai.json exists.

**Execute ALL steps in order. Do not skip or optimize.**

## Prerequisites

Read the following artifacts from `$MIGRATION_DIR/`:

- `aws-design-ai.json` (REQUIRED) — AI architecture design from Phase 3
- `estimation-ai.json` (REQUIRED) — AI cost estimates from Phase 4
- `ai-workload-profile.json` (REQUIRED) — AI workload profile from Phase 1
- `preferences.json` (REQUIRED) — User migration preferences from Phase 2

If any required file is missing: **STOP**. Output: "Missing required artifact: [filename]. Complete the prior phase that produces it."

## Part 0: Model Validation (Optional)

Before proceeding with migration code, offer the user an opportunity to validate the recommended Bedrock models with real prompts.

> "Would you like to validate the recommended Bedrock model(s) with your own prompts before proceeding? This requires an AWS account, costs ~$2-5, and helps ensure the recommended model meets your quality bar."
>
> A) Yes — validate with real prompts first
> B) No — proceed with migration (I'll validate during integration testing)

**If Yes:** Load `references/phases/generate/generate-evaluation.md` and follow the evaluation guide. Resume at Part 1 after evaluation is complete.

**If No:** Continue to Part 1.

---

## Part 1: Fast-Track Timeline

AI migrations are typically faster than full infrastructure migrations because:

- No data migration (models are API-based)
- No infrastructure provisioning (Bedrock is serverless)
- Changes are primarily code-level (SDK swap or config change)
- Feature flags enable gradual rollout

### Gateway-Aware Timeline

Check `preferences.json` → `ai_constraints.ai_gateway` to determine the appropriate timeline:

#### Gateway Users (1-3 DAYS) — `ai_gateway` = `llm_router`, `api_gateway`, `voice_platform`, or `framework`

Gateway users can migrate by changing configuration, not code:

- **Day 1:** Enable Bedrock model access + configure gateway to route to Bedrock
- **Day 2:** Test in staging, validate quality and latency
- **Day 3:** Gradual rollout (10% → 50% → 100%)

| Gateway Type                      | Migration Action                                                            | Effort            |
| --------------------------------- | --------------------------------------------------------------------------- | ----------------- |
| LLM Router (LiteLLM, OpenRouter)  | Change model string to `bedrock/<model_id>`                                 | 1 config line     |
| API Gateway (Kong, Apigee)        | Add Bedrock upstream + SigV4 signing                                        | 1-2 config files  |
| Voice Platform (Vapi, Bland.ai)   | Check native Bedrock support, update dashboard                              | Dashboard config  |
| Framework (LangChain, LlamaIndex) | Swap provider import (`ChatBedrock` instead of `ChatVertexAI`/`ChatOpenAI`) | 1-5 lines of code |

#### Direct SDK Users (1-3 WEEKS) — `ai_gateway` = `direct`

Direct SDK users need full code migration:

##### Week 1: Setup and Adapter Development

- Enable Bedrock model access in AWS account (from `aws-design-ai.json` model IDs)
- Create IAM role for Bedrock access
- Develop provider adapter (abstraction layer over source SDK and Bedrock)
- Implement feature flag for provider switching
- Unit test adapter with both providers

##### Week 2: Integration Testing and Comparison

- Deploy adapter to staging environment
- Run A/B test harness comparing source provider vs Bedrock responses
- Measure latency, quality, and cost per request
- Tune prompt templates for Bedrock models (if quality differs)
- Validate all capabilities from `ai-workload-profile.json` work on Bedrock

##### Week 3: Gradual Rollout and Cutover

- Enable feature flag for 10% of traffic to Bedrock
- Monitor error rates, latency, and quality metrics
- Increase to 50%, then 100% over 3-5 days
- Disable source provider after 48 hours stable at 100%
- Remove feature flag and source SDK code (optional, can keep as fallback)

### Timeline Adjustments

- **Gateway user, single model**: 1-2 days (config change)
- **Gateway user, multiple models**: 2-3 days (config changes per model)
- **Single model, direct SDK**: 1 week (simple SDK swap)
- **Multiple models, direct SDK**: 2 weeks (adapter per model)
- **Framework integration (LangChain, etc.)**: 1-2 weeks (framework handles most changes)
- **Custom inference pipeline**: 3 weeks (more testing needed)
- **If running alongside infrastructure migration**: Align with infrastructure Weeks 3-8

## Part 2: Step-by-Step Migration Guide

### SDK Migration Patterns

Based on `ai-workload-profile.json` `integration.pattern`:

#### Python: Vertex AI to Bedrock

```python
# BEFORE: Vertex AI
from google.cloud import aiplatform
from vertexai.generative_models import GenerativeModel

model = GenerativeModel("gemini-pro")
response = model.generate_content("Hello, world!")
print(response.text)

# AFTER: Amazon Bedrock
import boto3
import json

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6",  # From aws-design-ai.json
    messages=[{"role": "user", "content": [{"text": "Hello, world!"}]}]
)
print(response["output"]["message"]["content"][0]["text"])
```

#### JavaScript/TypeScript: Vertex AI to Bedrock

```javascript
// BEFORE: Vertex AI
const { VertexAI } = require("@google-cloud/vertexai");
const vertexAI = new VertexAI({ project: "my-project", location: "us-central1" });
const model = vertexAI.getGenerativeModel({ model: "gemini-pro" });
const result = await model.generateContent("Hello, world!");

// AFTER: Amazon Bedrock
const { BedrockRuntimeClient, ConverseCommand } = require("@aws-sdk/client-bedrock-runtime");
const client = new BedrockRuntimeClient({ region: "us-east-1" });
const result = await client.send(
  new ConverseCommand({
    modelId: "anthropic.claude-sonnet-4-6",
    messages: [{ role: "user", content: [{ text: "Hello, world!" }] }],
  })
);
```

#### Go: Vertex AI to Bedrock

```go
// BEFORE: Vertex AI
import "cloud.google.com/go/aiplatform/apiv1"

// AFTER: Amazon Bedrock
import (
    "github.com/aws/aws-sdk-go-v2/service/bedrockruntime"
)

client := bedrockruntime.NewFromConfig(cfg)
output, err := client.Converse(ctx, &bedrockruntime.ConverseInput{
    ModelId: aws.String("anthropic.claude-sonnet-4-6"),
    Messages: []types.Message{
        {Role: types.ConversationRoleUser, Content: []types.ContentBlock{
            &types.ContentBlockMemberText{Value: "Hello, world!"},
        }},
    },
})
```

#### Java: Vertex AI to Bedrock

```java
// BEFORE: Vertex AI
import com.google.cloud.vertexai.VertexAI;
import com.google.cloud.vertexai.generativeai.GenerativeModel;

// AFTER: Amazon Bedrock
import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;
import software.amazon.awssdk.services.bedrockruntime.model.*;

BedrockRuntimeClient client = BedrockRuntimeClient.builder()
    .region(Region.US_EAST_1)
    .build();
ConverseResponse response = client.converse(ConverseRequest.builder()
    .modelId("anthropic.claude-sonnet-4-6")
    .messages(Message.builder()
        .role(ConversationRole.USER)
        .content(ContentBlock.fromText("Hello, world!"))
        .build())
    .build());
```

### Streaming Migration

```python
# BEFORE: Vertex AI streaming
for chunk in model.generate_content("Hello", stream=True):
    print(chunk.text, end="")

# AFTER: Bedrock streaming
response = bedrock.converse_stream(
    modelId="anthropic.claude-sonnet-4-6",
    messages=[{"role": "user", "content": [{"text": "Hello"}]}]
)
for event in response["stream"]:
    if "contentBlockDelta" in event:
        print(event["contentBlockDelta"]["delta"]["text"], end="")
```

### Embeddings Migration

```python
# BEFORE: Vertex AI embeddings
from vertexai.language_models import TextEmbeddingModel
model = TextEmbeddingModel.from_pretrained("text-embedding-004")
embeddings = model.get_embeddings(["Hello, world!"])

# AFTER: Bedrock Titan Embeddings
response = bedrock.invoke_model(
    modelId="amazon.titan-embed-text-v2:0",
    body=json.dumps({"inputText": "Hello, world!"})
)
embedding = json.loads(response["body"].read())["embedding"]
```

### OpenAI SDK Migration (if `ai_source` = `"openai"` or `"both"`)

```python
# BEFORE: OpenAI SDK
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello, world!"}]
)
print(response.choices[0].message.content)

# AFTER: Amazon Bedrock
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
response = bedrock.converse(
    modelId="anthropic.claude-sonnet-4-6",
    messages=[{"role": "user", "content": [{"text": "Hello, world!"}]}]
)
print(response["output"]["message"]["content"][0]["text"])
```

### Gateway Migration Patterns (if `ai_gateway` != `"direct"`)

#### LLM Router (LiteLLM)

```python
# BEFORE: LiteLLM with OpenAI/Gemini
from litellm import completion
response = completion(model="gpt-4o", messages=[{"role": "user", "content": "Hello"}])

# AFTER: LiteLLM with Bedrock (config change only)
response = completion(model="bedrock/anthropic.claude-sonnet-4-6", messages=[{"role": "user", "content": "Hello"}])
```

#### LangChain (OpenAI → Bedrock)

```python
# BEFORE: LangChain with OpenAI
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o")

# AFTER: LangChain with Bedrock
from langchain_aws import ChatBedrock
llm = ChatBedrock(model_id="anthropic.claude-sonnet-4-6")
```

#### LlamaIndex (Vertex AI → Bedrock)

```python
# BEFORE: LlamaIndex with Vertex AI
from llama_index.llms.vertex import Vertex
llm = Vertex(model="gemini-pro")

# AFTER: LlamaIndex with Bedrock
from llama_index.llms.bedrock_converse import BedrockConverse
llm = BedrockConverse(model="anthropic.claude-sonnet-4-6")
```

### Model ID Mapping

Populate from `aws-design-ai.json` `ai_architecture.bedrock_models[]`:

| GCP Model            | Bedrock Model                  | Capability                 |
| -------------------- | ------------------------------ | -------------------------- |
| `gemini-pro`         | From `aws_model_id` in design  | Text generation, streaming |
| `text-embedding-004` | `amazon.titan-embed-text-v2:0` | Embeddings                 |

For each model in the design, map the GCP model ID to the Bedrock model ID and verify capability coverage from `capabilities_matched[]`.

## Part 3: Rollback Plan

### Feature Flag Strategy

The provider adapter includes a feature flag that controls routing:

```
AI_PROVIDER=vertex_ai    # Default: use existing Vertex AI
AI_PROVIDER=bedrock      # Switch to Bedrock
AI_PROVIDER=shadow       # Send to both, return Vertex AI response (for comparison)
```

### Rollback Steps

1. Set `AI_PROVIDER=vertex_ai` (instant, no deployment needed if using env var)
2. Verify Vertex AI is receiving all traffic
3. Monitor for 1 hour to confirm stability
4. Investigate Bedrock issues
5. Re-attempt migration after fixes

### Rollback Triggers

- Bedrock response quality below acceptable threshold (defined in success criteria)
- Bedrock latency P95 > 2x Vertex AI baseline
- Bedrock error rate > 1% for 5 minutes
- Bedrock cost per request > 3x Vertex AI

## Part 4: Monitoring and Observability

### Key Metrics to Track

| Metric                          | Source                        | Alert Threshold          |
| ------------------------------- | ----------------------------- | ------------------------ |
| Request latency (P50, P95, P99) | CloudWatch / application logs | P95 > 2x baseline        |
| Error rate                      | Application error logs        | > 1% for 5 min           |
| Token usage (input/output)      | Bedrock CloudWatch metrics    | > 150% of estimated      |
| Cost per request                | Bedrock cost metrics          | > 3x Vertex AI           |
| Response quality score          | A/B test harness output       | < 90% of Vertex AI score |
| Feature flag state              | Application config            | Unexpected changes       |

### Dashboard Layout

1. **Request volume**: Requests/minute by provider (Vertex AI vs Bedrock)
2. **Latency comparison**: Side-by-side P50/P95/P99
3. **Error rates**: By provider and error type
4. **Token usage**: Input/output tokens per hour
5. **Cost tracking**: Hourly/daily cost by model
6. **Quality scores**: From A/B test results (if shadow mode active)

### Alerting Rules

- **Critical**: Error rate > 5% for 2 min → Page on-call, auto-rollback
- **High**: Latency P95 > 3x baseline for 5 min → Page on-call
- **Medium**: Cost per day > 2x projected → Notify team lead
- **Low**: Token usage trending > 120% of estimate → Daily report

## Part 5: Production Readiness Checklist

Before switching production traffic to Bedrock:

- [ ] Bedrock model access enabled in AWS account
- [ ] IAM role with `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` permissions
- [ ] Provider adapter deployed and tested in staging
- [ ] A/B test harness run with >= 100 representative prompts
- [ ] Response quality >= 90% of Vertex AI quality score
- [ ] Latency P95 within 2x of Vertex AI baseline
- [ ] Error rate < 0.1% in staging
- [ ] Monitoring dashboards configured
- [ ] Alerting rules active
- [ ] Rollback procedure documented and tested
- [ ] Feature flag mechanism verified (can switch instantly)
- [ ] Cost estimates validated against actual staging usage

## Part 6: Success Criteria

### Quality Targets

| Criteria             | Target                                               | Measurement            |
| -------------------- | ---------------------------------------------------- | ---------------------- |
| Response quality     | >= 90% of Vertex AI baseline                         | A/B comparison scoring |
| Capability coverage  | 100% of capabilities from `ai-workload-profile.json` | Feature test suite     |
| Prompt compatibility | All existing prompts work without modification       | Regression test suite  |

### Latency Targets

| Criteria                        | Target                   | Measurement         |
| ------------------------------- | ------------------------ | ------------------- |
| P50 latency                     | Within 1.5x of Vertex AI | Application metrics |
| P95 latency                     | Within 2x of Vertex AI   | Application metrics |
| Time to first token (streaming) | Within 2x of Vertex AI   | Streaming metrics   |

### Cost Targets

| Criteria         | Target                                        | Measurement          |
| ---------------- | --------------------------------------------- | -------------------- |
| Monthly cost     | Within 20% of `estimation-ai.json` projection | AWS billing          |
| Cost per request | Within 30% of Vertex AI per-request cost      | Per-request tracking |

## Part 7: Output Format

Generate `generation-ai.json` in `$MIGRATION_DIR/` with the following schema:

```json
{
  "phase": "generate",
  "generation_source": "ai",
  "timestamp": "2026-02-26T14:30:00Z",
  "migration_plan": {
    "total_weeks": 2,
    "approach": "fast_track",
    "phases": [
      {
        "name": "Setup and Adapter Development",
        "week": 1,
        "key_activities": [
          "Enable Bedrock model access",
          "Create IAM role",
          "Develop provider adapter",
          "Implement feature flag"
        ]
      },
      {
        "name": "Integration Testing",
        "week": 2,
        "key_activities": [
          "Deploy to staging",
          "Run A/B comparison",
          "Validate quality and latency"
        ]
      },
      {
        "name": "Gradual Rollout",
        "week": 3,
        "key_activities": [
          "Enable 10% traffic",
          "Scale to 100%",
          "Disable Vertex AI"
        ]
      }
    ],
    "models_to_migrate": [
      {
        "gcp_model_id": "gemini-pro",
        "bedrock_model_id": "anthropic.claude-sonnet-4-6",
        "capabilities": ["text_generation", "streaming"],
        "migration_complexity": "medium"
      }
    ]
  },
  "step_by_step_guide": {
    "languages": ["python"],
    "primary_pattern": "direct_sdk",
    "files_to_modify": [
      {
        "file": "src/ai/client.py",
        "change_type": "sdk_swap",
        "complexity": "medium"
      }
    ],
    "dependency_changes": {
      "remove": ["google-cloud-aiplatform"],
      "add": ["boto3"]
    }
  },
  "rollback_plan": {
    "mechanism": "feature_flag",
    "flag_name": "AI_PROVIDER",
    "default_value": "vertex_ai",
    "rollback_time": "instant (env var change)",
    "triggers": [
      "Quality below 90% threshold",
      "P95 latency > 2x baseline",
      "Error rate > 1% for 5 min"
    ]
  },
  "monitoring": {
    "dashboards": [
      "Request volume",
      "Latency comparison",
      "Error rates",
      "Token usage",
      "Cost tracking"
    ],
    "alerting_rules": [
      {
        "severity": "critical",
        "condition": "Error rate > 5% for 2 min",
        "action": "Page on-call, auto-rollback"
      }
    ]
  },
  "production_readiness_checklist": [
    "Bedrock model access enabled",
    "IAM role configured",
    "Provider adapter tested in staging",
    "A/B comparison completed (>= 100 prompts)",
    "Quality >= 90% of Vertex AI",
    "Monitoring and alerting active",
    "Rollback procedure tested"
  ],
  "success_criteria": {
    "quality": {
      "response_quality": ">= 90% of Vertex AI baseline",
      "capability_coverage": "100%"
    },
    "latency": {
      "p50": "within 1.5x of Vertex AI",
      "p95": "within 2x of Vertex AI"
    },
    "cost": {
      "monthly": "within 20% of estimation-ai.json projection",
      "per_request": "within 30% of Vertex AI"
    }
  },
  "recommendation": {
    "approach": "Fast-track migration with feature flag rollback",
    "confidence": "high",
    "key_risks": ["Model quality differences", "Prompt tuning needed"],
    "estimated_total_effort_hours": 80
  }
}
```

## Output Validation Checklist

- `phase` is `"generate"`
- `generation_source` is `"ai"`
- `migration_plan.total_weeks` is 1-4 (fast track)
- `migration_plan.models_to_migrate` covers all models from `aws-design-ai.json`
- `step_by_step_guide.languages` matches `ai-workload-profile.json` languages
- `step_by_step_guide.files_to_modify` matches `aws-design-ai.json` code_migration
- `rollback_plan.mechanism` is `"feature_flag"`
- `monitoring` has both dashboards and alerting rules
- `production_readiness_checklist` has at least 5 items
- `success_criteria` covers quality, latency, and cost
- Output is valid JSON

## Generate Phase Integration

The parent orchestrator (`generate.md`) uses `generation-ai.json` to:

1. Gate Stage 2 artifact generation — `generate-artifacts-ai.md` requires this file
2. Provide AI migration context to `generate-artifacts-docs.md` for MIGRATION_GUIDE.md
3. Set phase completion outputs in `.phase-status.json`
