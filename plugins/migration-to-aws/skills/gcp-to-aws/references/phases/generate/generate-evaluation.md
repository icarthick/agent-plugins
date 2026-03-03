# Generate Phase: Model Evaluation (Optional)

> Loaded by `generate-ai.md` Part 0 when the user opts into model validation before proceeding with migration.

**Purpose:** Validate recommended Bedrock models with real prompts before committing to migration code.

**Cost:** ~$1-25 depending on prompt count and models tested.

---

## Why This Matters

The design phase recommended 2-3 Bedrock models based on requirements, cost, and features. But benchmarks and vendor claims only go so far. The only way to know which model actually works best for **your** prompts and **your** use case is to test it yourself.

---

## Prerequisite: AWS Account Required

Bedrock model evaluation requires an AWS account. If you don't have one yet, this is the best reason to create one — you get to validate your model choice with real data before migrating anything.

Once you have an account, enable model access for the models you want to evaluate:

1. Open the Bedrock console
2. In the left nav, choose **Model catalog** and find the models from your recommendation
3. For **Anthropic models (Claude)**, you must complete a one-time First Time Use (FTU) form before invoking — select any Claude model in the catalog to trigger this. Access is granted immediately after submission.
4. For all other providers (Nova, Llama, Mistral, etc.), access is enabled by default — no request needed.

> **Note:** When you invoke a third-party model for the first time, Bedrock automatically initiates a subscription in the background (up to 15 minutes). Ensure your IAM role has `aws-marketplace:Subscribe`, `aws-marketplace:Unsubscribe`, and `aws-marketplace:ViewSubscriptions` permissions to avoid 403 errors after this setup period.

---

## Step 1: Determine Your Evaluation Approach

**Ask yourself: Do you have labeled reference answers for your prompts?**

### Path A: No labeled data (most common)

You have prompts but no "correct answer" to compare against.

Use **Automatic evaluation with model-as-judge**:

- A judge model (Claude) scores the outputs on criteria you define
- No ground truth labels needed
- Best for: open-ended generation, summarization, chat, agentic tasks

### Path B: You have reference answers

You have prompts AND the expected correct output for each.

Use **Automatic evaluation with built-in metrics**:

- Bedrock scores outputs against your reference answers automatically
- Best for: classification, Q&A, structured extraction tasks

### Path C: You want your team to review outputs directly

You want humans on your team to compare model responses side-by-side.

Use **Human evaluation**:

- Your team rates responses using a Likert scale or preference ranking
- Best for: voice/conversational AI, subjective quality judgments

---

## Step 2: Build Your Prompt Dataset

Create a file with 10-50 prompts representative of your actual workload. More prompts = more reliable results, but 10-20 is enough to get a signal.

**Format:** One JSON object per line (JSONL), saved as a `.jsonl` file.

### For Path A or B (automatic evaluation):

```jsonl
{"prompt": "Your actual prompt here", "referenceResponse": "Expected answer (leave blank for Path A)", "category": "task-type"}
{"prompt": "Another real prompt from your app", "referenceResponse": "", "category": "task-type"}
```

**Tips for good prompts:**

- Use real prompts from your application, not made-up ones
- Include edge cases that have caused problems in the past
- Cover the range of inputs your users actually send
- If you detected function calling in your code, include prompts that trigger tool use
- If you detected RAG patterns, include prompts that require retrieval
- Reference `ai-workload-profile.json` → `models[].usage_context` for domain-specific prompt crafting hints

### For Path C (human evaluation):

Same format — Bedrock will show each prompt's outputs to your reviewers side-by-side.

**Save your file as:** `eval-prompts.jsonl`

---

## Step 3: Upload to S3

```bash
# Create a bucket (replace with your preferred name and region)
aws s3 mb s3://my-model-eval-bucket --region us-east-1

# Upload your prompt dataset
aws s3 cp eval-prompts.jsonl s3://my-model-eval-bucket/prompts/eval-prompts.jsonl

# Create an output folder for results
aws s3api put-object --bucket my-model-eval-bucket --key results/
```

---

## Step 4: Create the Evaluation Job

### Option A: Console (easiest)

1. Open the Bedrock console
2. In the navigation pane, choose **Model evaluation**
3. In the **Build an evaluation** card, under **Automatic** choose **Create automatic evaluation**
4. Fill in:
   - **Models to evaluate:** Add each of your 2-3 recommended models
   - **Task type:** Choose the type that matches your use case (text generation, summarization, Q&A, classification)
   - **Prompt dataset:** Choose "Use your own" → paste your S3 URI
   - **Results location:** Your S3 output folder
   - **IAM role:** Choose "Create a new role" if you don't have one
5. Choose **Create**

### Option B: CLI

Use model IDs from `aws-design-ai.json` → `bedrock_models[].aws_model_id`:

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

Replace MODEL_ID_1 and MODEL_ID_2 with actual model IDs from your recommendation. Note that `modelIdentifier` requires the full ARN format.

---

## Step 5: Define Custom Metrics (Path A only)

If you're using model-as-judge, define what "good" means for your use case. The judge model uses your criteria to score each response.

**Use binary pass/fail per criterion rather than numeric scales.** Research shows binary scoring produces more reliable and consistent results across different judge models than 1-5 scales.

**Example judge prompt for a customer service use case:**

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

**Example judge prompt for a code generation use case:**

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

Tailor the criteria to what actually matters for your application.

---

## Step 6: Read the Results

Once the job status shows **Completed** (typically 15-30 minutes):

1. Open the job in the Bedrock console to see the summary report card
2. Or download the full results from your S3 output folder

**What to look for:**

- Which model scores highest on your most important criterion
- Whether the differences are meaningful (a 0.1 difference on a 5-point scale may not matter)
- Any model that consistently fails on a specific category of prompt — that's a red flag
- Latency data if you have real-time requirements

**Re-evaluate when:**

- A major new model version is released
- Your application's prompt patterns change significantly
- You're seeing quality degradation in production
- You're considering a cost optimization (switching to a cheaper model)

---

## Cost Estimate

Bedrock model evaluation costs are based on the tokens processed:

- **Small eval (20 prompts, 3 models):** ~$1-5
- **Medium eval (100 prompts, 3 models):** ~$5-25
- **With model-as-judge:** Add ~50% for judge model tokens

This is a one-time cost to validate a migration decision.

---

## What This Doesn't Replace

This evaluation tells you which model performs best on your **current** prompts. It doesn't:

- Guarantee production performance at scale
- Replace A/B testing in production with real user traffic
- Account for prompt changes you'll make during migration
- Test edge cases you haven't thought of yet

The right sequence is: **eval on sample data → migrate with feature flag → A/B test in production → monitor continuously.**

---

## Quick Reference: Model IDs for Evaluation Jobs

| Model | Bedrock Model ID |
|-------|-----------------|
| Claude Sonnet 4.6 | `anthropic.claude-sonnet-4-6` |
| Claude Sonnet 4.5 | `anthropic.claude-sonnet-4-5-20250929-v1:0` |
| Claude Haiku 4.5 | `anthropic.claude-haiku-4-5-20251001-v1:0` |
| Claude Opus 4.6 | `anthropic.claude-opus-4-6-v1` |
| Llama 4 Maverick | `meta.llama4-maverick-17b-instruct-v1:0` |
| Llama 4 Scout | `meta.llama4-scout-17b-instruct-v1:0` |
| Nova Pro | `amazon.nova-pro-v1:0` |
| Nova Lite | `amazon.nova-lite-v1:0` |
| Nova Micro | `amazon.nova-micro-v1:0` |
| Llama 3.3 70B | `meta.llama3-3-70b-instruct-v1:0` |
| Mistral Large 3 | `mistral.mistral-large-3-675b-instruct` |

**Note:** Some model IDs omit the version suffix — this matches the official AWS model catalog. **Always verify current model IDs** via the Bedrock console or AWS documentation — IDs can change with new versions. For CLI/SDK use, wrap the model ID in the full ARN: `arn:aws:bedrock:REGION::foundation-model/MODEL_ID`.
