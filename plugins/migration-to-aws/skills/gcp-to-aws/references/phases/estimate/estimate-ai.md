# Estimate Phase: AI Workload Cost Analysis

> Loaded by estimate.md when aws-design-ai.json exists.

**Execute ALL steps in order. Do not skip or optimize.**

## Pricing Mode

The parent `estimate.md` selects the pricing mode before loading this file. Use whichever mode was selected:

- **Live (MCP Server)**: Query `get_pricing("AmazonBedrock", "us-east-1")` with model filter, accuracy ±5-10%
- **Fallback (Cached)**: Use the Bedrock pricing table below, accuracy ±15-25%

### Cached Bedrock Pricing Fallback Table

If MCP is unavailable, use these rates:

| Model               | Input (per 1M tokens) | Output (per 1M tokens) |
| ------------------- | --------------------- | ---------------------- |
| Claude Opus         | $15.00                | $75.00                 |
| Claude Sonnet       | $3.00                 | $15.00                 |
| Claude Haiku        | $0.80                 | $4.00                  |
| Llama 70B           | $0.60                 | $0.60                  |
| Llama 8B            | $0.10                 | $0.10                  |
| Nova Pro            | $0.80                 | $3.20                  |
| Titan Embeddings v2 | $0.02                 | N/A                    |

The formulas and methodology below are identical for both modes. The only difference is the price source.

## Prerequisites

Before starting, read these files from `$MIGRATION_DIR/`:

**From `ai-workload-profile.json`:**

- `current_costs.monthly_ai_spend` — Current GCP AI spend (if billing data was provided)
- `current_costs.services_detected` — Which AI services are billed
- `models[]` — Which models are in use

**From `preferences.json`:**

- `ai_constraints.ai_token_volume.value` — Expected monthly token volume
- `ai_constraints.ai_capabilities_required.value` — Capabilities the model must support

**From `aws-design-ai.json`:**

- `ai_architecture.bedrock_models[]` — Selected primary and backup models
- `ai_architecture.capability_mapping` — Capability parity assessment

## Part 1: Establish Current GCP AI Costs

Determine current Vertex AI spending from the best available source:

### If billing data was provided (preferred)

Use `current_costs.monthly_ai_spend` from `ai-workload-profile.json`:

```
Current monthly AI spend: $[amount]
Services: [list from services_detected]
```

### If no billing data

Estimate from discovered models and typical Vertex AI pricing:

```
Vertex AI Gemini Pro pricing:
  Input:  $0.125 per 1M tokens
  Output: $0.375 per 1M tokens

Vertex AI Embeddings pricing:
  Input:  $0.025 per 1M tokens

Estimated monthly cost = (estimated input tokens x input rate) + (estimated output tokens x output rate)
```

Use the token volume from `preferences.json` (`ai_constraints.ai_token_volume.value`) as the basis. Apply a 60/40 input/output ratio if the actual ratio is unknown.

### If neither billing nor token volume is known

Note this in the output and present the model comparison table with multiple volume tiers so the user can find their range.

## Part 2: Build Model Comparison Table

This is the core output. Calculate the monthly Bedrock cost for **every viable model** at the user's token volume.

### Token Volume Mapping

Convert the clarified answer to a concrete number for calculation:

| Answer     | Input tokens/month | Output tokens/month | Ratio |
| ---------- | ------------------ | ------------------- | ----- |
| A) < 10M   | 6M                 | 4M                  | 60/40 |
| B) 10-100M | 60M                | 40M                 | 60/40 |
| C) 100M-1B | 600M               | 400M                | 60/40 |
| D) > 1B    | 1.2B               | 800M                | 60/40 |
| E) Unsure  | Show all tiers     | --                  | --    |

If the design phase or discover phase has more specific token estimates, use those instead.

### Cost Formula

For each model:

```
Monthly Cost = (Input tokens / 1M x Input rate per 1M) + (Output tokens / 1M x Output rate per 1M)
```

### Generate Comparison Table

Calculate for all models and present as a comparison:

```
Your token volume: [X]M input + [Y]M output per month

Model               Monthly Cost    vs Current GCP    Quality    Capabilities Match
------------------------------------------------------------------------------------
Claude Opus         $[calc]         [+/-]$[diff]      Excellent  [Yes/No: list gaps]
Claude Sonnet       $[calc]         [+/-]$[diff]      Very Good  [Yes/No: list gaps]
Claude Haiku        $[calc]         [+/-]$[diff]      Good       [Yes/No: list gaps]
Llama 70B           $[calc]         [+/-]$[diff]      Good       [Yes/No: list gaps]
Llama 8B            $[calc]         [+/-]$[diff]      Fair       [Yes/No: list gaps]
Nova Pro            $[calc]         [+/-]$[diff]      Good       [Yes/No: list gaps]
------------------------------------------------------------------------------------
Current (Vertex AI) $[current]      --                 --         --
```

**Capabilities Match** — Check each model against `ai_constraints.ai_capabilities_required.value` from `preferences.json`. Mark "No" and list gaps if a required capability is missing (e.g., "No: missing function_calling").

### If embeddings are needed

Add a separate line for the embeddings model:

```
Embeddings (Titan v2): $[calc] per month
  [X]M tokens x $0.02/1M = $[calc]
```

Embeddings cost is additive — it's separate from the primary model cost.

## Part 3: Recommended Model Cost Breakdown

Based on the model selected in the design phase, provide a detailed breakdown:

```
Recommended: [Model name] ([rationale from design phase])

Monthly Cost Breakdown:
  Input tokens:    [X]M x $[rate]/1M = $[calc]
  Output tokens:   [Y]M x $[rate]/1M = $[calc]
  Embeddings:      [Z]M x $[rate]/1M = $[calc]  (if applicable)
  ------------------------------------------------
  Total monthly:   $[total]

Comparison to current:
  Current GCP AI spend:  $[current]/month
  Projected Bedrock:     $[projected]/month
  Difference:            [+/-]$[diff]/month ([+/-]X%)
  Annual difference:     [+/-]$[annual diff]
```

Also show the backup model cost for comparison:

```
Backup: [Model name]
  Monthly cost: $[calc]
  vs Recommended: [+/-]$[diff] ([+/-]X%)
```

## Part 4: One-Time Migration Costs

AI-only migration has significantly lower one-time costs than full infrastructure migration.

```
Component                    Hours    Rate     Cost
------------------------------------------------------
Code migration (SDK swap)    8-16     $150     $1,200-2,400
Testing & validation         8-16     $150     $1,200-2,400
Staging deployment           4-8      $150     $600-1,200
Production rollout           4-8      $150     $600-1,200
------------------------------------------------------
TOTAL Development                              $3,600-7,200

Infrastructure:
  AWS account/Bedrock setup              $0 (free)
  Initial Bedrock testing (tokens)       $50-200
  ------------------------------------------------
  TOTAL Infrastructure                   $50-200

TOTAL One-Time Cost:                     $3,650-7,400
```

Adjust based on complexity signals from `ai-workload-profile.json`:

- `integration.pattern = "framework"` (LangChain/LlamaIndex) -> Lower end (just swap provider)
- `integration.pattern = "direct_sdk"` -> Mid range (swap SDK + API calls)
- `integration.pattern = "rest_api"` -> Higher end (change endpoints + auth + response parsing)
- `summary.total_models_detected` > 3 -> Add $1,000-2,000 for additional model migrations

## Part 5: ROI Analysis

### Monthly Savings/Cost

```
Monthly difference: $[current GCP] - $[projected Bedrock] = $[savings or cost increase]
Annual difference:  $[monthly diff] x 12 = $[annual]
```

### Payback Period

```
If Bedrock is cheaper:
  Payback = One-time cost / Monthly savings
  Example: $5,000 / $50 savings per month = 100 months

If Bedrock is more expensive:
  No payback from cost alone -- justify with capability, quality, or strategic value
```

### Non-Cost Benefits

Present these alongside the financial analysis:

- **Model quality** — Claude/Llama may offer better reasoning, coding, or analysis than Gemini for specific use cases
- **Model flexibility** — Bedrock offers multiple models; can switch without infrastructure changes
- **AWS ecosystem** — If planning broader AWS adoption, AI-only is a low-risk first step
- **Vendor diversification** — Reduces single-vendor risk
- **Feature access** — Bedrock features not available on Vertex AI (e.g., Guardrails, Knowledge Bases)

## Part 6: Cost Optimization Opportunities

Present optimization strategies relevant to AI workloads:

### 1. Model Downsizing

If the user chose a premium model but has high volume:

```
Current recommendation: Claude Sonnet
  Monthly cost at [X]M tokens: $[cost]

Alternative: Llama 70B (for cost-critical tasks)
  Monthly cost at [X]M tokens: $[cost]
  Savings: $[diff]/month ([X]%)

Approach: Route simple tasks to Llama, complex tasks to Claude
```

### 2. Prompt Caching (If Supported)

```
If your prompts have repeated system prompts or context:
  Without caching: $[cost]/month
  With caching (90% cache hit): $[cost x 0.1 + small overhead]/month
  Savings: ~$[diff]/month
```

### 3. Batch API (For Non-Real-Time Workloads)

```
If latency = "Batch" (from preferences.json ai_constraints):
  On-demand API: $[cost]/month
  Batch API (50% discount): $[cost x 0.5]/month
  Savings: $[diff]/month
```

### 4. Provisioned Throughput (For High Volume)

```
If token volume > 100M/month and traffic is predictable:
  On-demand pricing: $[cost]/month
  Provisioned throughput: May offer lower per-token rate at commitment
  Evaluate: Compare monthly commitment vs on-demand at your volume
```

### 5. Input Token Reduction

```
Techniques to reduce token costs:
  - Shorter system prompts (fewer input tokens per request)
  - Summarize context before sending (reduce RAG context size)
  - Use structured output formats (reduce output verbosity)
  - Estimated savings: 10-30% token reduction achievable
```

## Output

Write `estimation-ai.json` and `estimation-ai-report.md`.

### estimation-ai.json schema

```json
{
  "phase": "estimate",
  "timestamp": "2026-02-24T14:00:00Z",
  "pricing_mode": "live|fallback",
  "accuracy_confidence": "±5-10%|±15-25%",

  "current_costs": {
    "source": "billing_data|estimated",
    "gcp_monthly_ai_spend": 450,
    "services": ["Vertex AI Predictions", "Generative AI API"]
  },

  "token_volume": {
    "source": "preferences_ai_constraints|design_phase|estimated",
    "monthly_input_tokens": 600000000,
    "monthly_output_tokens": 400000000,
    "input_output_ratio": "60/40"
  },

  "model_comparison": [
    {
      "model": "claude-opus",
      "monthly_cost": 39000,
      "vs_current": "+$38,550",
      "quality": "excellent",
      "capabilities_match": true,
      "missing_capabilities": []
    }
  ],

  "recommended_model": {
    "model": "llama-70b",
    "monthly_cost": 600,
    "breakdown": {
      "input_tokens": 360,
      "output_tokens": 240,
      "embeddings": 12
    },
    "rationale": "Cost-optimized: supports all required capabilities at lowest cost"
  },

  "backup_model": {
    "model": "claude-sonnet",
    "monthly_cost": 7800,
    "rationale": "Quality fallback for tasks requiring higher accuracy"
  },

  "embeddings": {
    "model": "titan-embeddings-v2",
    "monthly_cost": 12,
    "monthly_tokens": 600000000,
    "note": "Separate from primary model cost"
  },

  "cost_comparison": {
    "current_gcp_monthly": 450,
    "projected_bedrock_monthly": 612,
    "monthly_difference": 162,
    "annual_difference": 1944,
    "percent_change": "+36%"
  },

  "one_time_costs": {
    "development": 4800,
    "infrastructure": 100,
    "total_one_time": 4900,
    "complexity_factors": [
      "direct_sdk integration pattern (moderate effort)",
      "2 models to migrate",
      "1 framework dependency (langchain)"
    ]
  },

  "roi_analysis": {
    "monthly_cost_delta": 162,
    "payback_period_months": null,
    "justification": "Migration increases monthly cost by $162 (+36%). Justified by model flexibility, vendor diversification, and AWS ecosystem access.",
    "non_cost_benefits": [
      "Access to multiple model providers via single API",
      "Bedrock Guardrails for content safety",
      "Foundation for broader AWS adoption",
      "Vendor diversification (reduce GCP single-vendor risk)"
    ]
  },

  "optimization_opportunities": [
    {
      "opportunity": "Prompt caching",
      "potential_savings_monthly": 180,
      "implementation_effort": "low",
      "description": "Cache repeated system prompts to reduce input token costs by ~30%"
    },
    {
      "opportunity": "Batch API for non-real-time tasks",
      "potential_savings_monthly": 120,
      "implementation_effort": "medium",
      "description": "Route batch processing tasks through Batch API for 50% token discount"
    },
    {
      "opportunity": "Input token reduction",
      "potential_savings_monthly": 60,
      "implementation_effort": "medium",
      "description": "Optimize prompt templates and RAG context to reduce input tokens by 10-20%"
    }
  ],

  "optimized_projection": {
    "monthly_with_optimizations": 252,
    "vs_current": "-$198",
    "note": "With all optimizations applied, Bedrock could be cheaper than current Vertex AI spend"
  }
}
```

### estimation-ai-report.md

Present the AI cost analysis as a readable report covering:

- Current GCP AI costs
- Model comparison table with all viable Bedrock models
- Recommended model with detailed cost breakdown
- One-time migration costs
- ROI analysis
- Cost optimization opportunities
- Optimized projection

## Output Validation Checklist

- `pricing_mode` is either `"live"` or `"fallback"`
- `current_costs` populated from `ai-workload-profile.json` or estimated
- `token_volume` populated from `preferences.json` `ai_constraints.ai_token_volume.value`
- `model_comparison` includes ALL viable Bedrock models (not just the recommended one)
- Every model in `model_comparison` has `capabilities_match` checked against `ai_constraints.ai_capabilities_required.value`
- `recommended_model.rationale` references specific user answers (priority, preference, volume)
- `cost_comparison` shows clear delta between current and projected
- `one_time_costs` adjusted based on integration complexity from `ai-workload-profile.json`
- `roi_analysis` is honest — if migration increases cost, say so and justify with non-cost benefits
- `optimization_opportunities` only includes strategies relevant to the user's workload pattern
- `optimized_projection` shows the best-case monthly cost with all optimizations applied
- No compute, database, storage, or networking cost calculations (those belong in `estimate-infra.md`)
- All cost values are numbers, not strings
- Output is valid JSON

## Execute Phase Integration

The Execute phase uses `estimation-ai.json` as follows:

1. **`recommended_model`** — Which Bedrock model to provision and test
2. **`one_time_costs`** — Budget for the migration effort
3. **`optimization_opportunities`** — Which optimizations to implement and when (some in initial migration, some post-migration)
4. **`cost_comparison`** — Set cost monitoring targets and alerts in production
5. **`model_comparison`** — Fallback options if the recommended model doesn't meet quality bar during testing

## Scope Boundary

**This phase covers financial analysis ONLY for AI workloads.**

FORBIDDEN — Do NOT include ANY of:

- Compute, database, storage, or networking cost calculations
- Infrastructure provisioning steps
- Code migration examples
- Detailed migration timeline

**Your ONLY job: Show the financial picture of moving AI to Bedrock. Nothing else.**
