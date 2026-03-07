# Generate Phase: Model Evaluation (Optional)

> Loaded by `generate-ai.md` Part 0 when the user opts into model validation before proceeding with migration.

**Purpose:** Validate recommended Bedrock models with real prompts before committing to migration code.

**Estimated cost:** ~$1-25 depending on prompt count and models tested.

---

## Step 1: Verify AWS Account Prerequisite

Present user with the following prerequisite information:

> Bedrock model evaluation requires an AWS account with model access enabled.
>
> - Anthropic models (Claude): Complete the one-time First Time Use (FTU) form in the Bedrock console Model catalog. Access is granted immediately.
> - All other providers (Nova, Llama, Mistral): Access is enabled by default.
> - First-time third-party model invocation triggers a background subscription (up to 15 min). The IAM role needs `aws-marketplace:Subscribe`, `aws-marketplace:Unsubscribe`, and `aws-marketplace:ViewSubscriptions` permissions.

If the user does not have an AWS account, note this and return to `generate-ai.md` Part 1.

---

## Step 2: Determine Evaluation Path

Present user with three options:

| Path  | When to use                                | Method                                                                                        |
| ----- | ------------------------------------------ | --------------------------------------------------------------------------------------------- |
| **A** | No labeled reference answers (most common) | Automatic evaluation with model-as-judge — a judge model scores outputs on defined criteria   |
| **B** | Reference answers exist for each prompt    | Automatic evaluation with built-in metrics — Bedrock scores outputs against reference answers |
| **C** | Team wants to review outputs directly      | Human evaluation — reviewers rate responses via Likert scale or preference ranking            |

Record the selected path for use in subsequent steps.

---

## Step 3: Build Prompt Dataset

Instruct the user to create `eval-prompts.jsonl` with 10-50 representative prompts. Format — one JSON object per line:

```jsonl
{"prompt": "Actual prompt text", "referenceResponse": "Expected answer (empty string for Path A)", "category": "task-type"}
{"prompt": "Another real prompt", "referenceResponse": "", "category": "task-type"}
```

Present these prompt selection guidelines:

- Use real prompts from the application, not synthetic ones
- Include edge cases that have caused problems historically
- Cover the full range of actual user inputs
- If `ai-workload-profile.json` → `models[].usage_context` indicates function calling, include tool-use prompts
- If RAG patterns were detected, include retrieval-dependent prompts

Path C uses the same format — Bedrock displays outputs to reviewers side-by-side.

---

## Step 4: Upload to S3

Present the following commands for the user to run:

```bash
aws s3 mb s3://my-model-eval-bucket --region us-east-1
aws s3 cp eval-prompts.jsonl s3://my-model-eval-bucket/prompts/eval-prompts.jsonl
aws s3api put-object --bucket my-model-eval-bucket --key results/
```

---

## Step 5: Create the Evaluation Job

Present two options — **Console** or **CLI**.

**Console:** Open Bedrock console → Model evaluation → Create automatic evaluation. Configure: models from the recommendation, matching task type, S3 prompt URI, S3 results location, IAM role.

**CLI:** Use model IDs from `aws-design-ai.json` → `bedrock_models[].aws_model_id`:

```bash
aws bedrock create-evaluation-job \
  --job-name "my-migration-eval" \
  --role-arn "arn:aws:iam::YOUR_ACCOUNT:role/BedrockEvalRole" \
  --inference-config '{
    "models": [
      {"bedrockModel": {"modelIdentifier": "arn:aws:bedrock:us-east-1::foundation-model/MODEL_ID_1"}},
      {"bedrockModel": {"modelIdentifier": "arn:aws:bedrock:us-east-1::foundation-model/MODEL_ID_2"}}
    ]
  }' \
  --output-data-config '{"s3Uri": "s3://my-model-eval-bucket/results/"}' \
  --evaluation-config '{
    "automated": {
      "datasetMetricConfigs": [{
        "taskType": "QuestionAndAnswer",
        "dataset": {"name": "my-prompts", "datasetLocation": {"s3Uri": "s3://my-model-eval-bucket/prompts/eval-prompts.jsonl"}},
        "metricNames": ["Builtin.Accuracy", "Builtin.Robustness"]
      }]
    }
  }'
```

Replace `MODEL_ID_1`/`MODEL_ID_2` with actual model IDs. The `modelIdentifier` field requires full ARN format.

---

## Step 6: Define Custom Metrics (Path A Only)

If Path A was selected, present the user with judge prompt guidance. Use binary pass/fail per criterion (more reliable than numeric scales).

**Customer service example:**

```text
You are evaluating an AI assistant's response to a customer query.
For each criterion, answer PASS or FAIL only.

1. Accuracy: Is the information correct and complete?
2. Tone: Is the response professional and empathetic?
3. Actionability: Does it give the customer a clear next step?

Query: {{prompt}}
Response: {{response}}

For each criterion provide: PASS or FAIL, and one sentence of reasoning.
```

**Code generation example:**

```text
You are evaluating AI-generated code.
For each criterion, answer PASS or FAIL only.

1. Correctness: Does the code solve the stated problem?
2. Quality: Is it readable, well-structured, and idiomatic?
3. Safety: Does it avoid common security pitfalls?

Task: {{prompt}}
Generated code: {{response}}

For each criterion provide: PASS or FAIL, and brief reasoning.
```

Instruct the user to tailor criteria to their application domain.

---

## Step 7: Review Results

Job completion typically takes 15-30 minutes. Present the user with what to check:

- Which model scores highest on the most important criterion
- Whether score differences are meaningful (small deltas may not matter)
- Any model that consistently fails on a specific prompt category (red flag)
- Latency data if real-time requirements exist

---

## Cost Reference

| Scenario                            | Estimated Cost                  |
| ----------------------------------- | ------------------------------- |
| Small eval (20 prompts, 3 models)   | ~$1-5                           |
| Medium eval (100 prompts, 3 models) | ~$5-25                          |
| With model-as-judge                 | Add ~50% for judge model tokens |

---

## Model ID Reference

| Model             | Bedrock Model ID                            |
| ----------------- | ------------------------------------------- |
| Claude Sonnet 4.6 | `anthropic.claude-sonnet-4-6`               |
| Claude Sonnet 4.5 | `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| Claude Haiku 4.5  | `anthropic.claude-haiku-4-5-20251001-v1:0`  |
| Claude Opus 4.6   | `anthropic.claude-opus-4-6-v1`              |
| Llama 4 Maverick  | `meta.llama4-maverick-17b-instruct-v1:0`    |
| Llama 4 Scout     | `meta.llama4-scout-17b-instruct-v1:0`       |
| Nova Pro          | `amazon.nova-pro-v1:0`                      |
| Nova Lite         | `amazon.nova-lite-v1:0`                     |
| Nova Micro        | `amazon.nova-micro-v1:0`                    |
| Llama 3.3 70B     | `meta.llama3-3-70b-instruct-v1:0`           |
| Mistral Large 3   | `mistral.mistral-large-3-675b-instruct`     |

Model IDs may omit the version suffix per the AWS model catalog. Always verify current IDs via the Bedrock console or AWS documentation — IDs change with new versions. For CLI/SDK use, wrap in the full ARN: `arn:aws:bedrock:REGION::foundation-model/MODEL_ID`.
