# Estimate Phase: AI Workload Cost Analysis

> Loaded by estimate.md when aws-design-ai.json exists.

**Execute ALL steps in order. Do not skip or optimize.**

## Pricing Mode

The parent `estimate.md` selects the pricing mode before loading this file. Use whichever mode was selected:

- **Live (MCP Server)**: Query `get_pricing("AmazonBedrock", "us-east-1")` with model filter, accuracy ±5-10%
- **Fallback (Cached)**: Use the Bedrock pricing table below, accuracy ±15-25%

### Cached Bedrock Pricing Fallback Table

If MCP is unavailable, use these rates (from `pricing-fallback.json`):

| Model               | Model ID                                   | Input (per 1M) | Output (per 1M) | Tier      |
| ------------------- | ------------------------------------------ | -------------- | --------------- | --------- |
| Claude Opus 4.6     | `anthropic.claude-opus-4-6-v1`             | $5.00          | $25.00          | Premium   |
| Claude Sonnet 4.6   | `anthropic.claude-sonnet-4-6`              | $3.00          | $15.00          | Flagship  |
| Claude Haiku 4.5    | `anthropic.claude-haiku-4-5-20251001-v1:0` | $1.00          | $5.00           | Fast      |
| Llama 4 Maverick    | `meta.llama4-maverick-17b-instruct-v1:0`   | $0.24          | $0.97           | Mid       |
| Llama 4 Scout       | `meta.llama4-scout-17b-instruct-v1:0`      | $0.17          | $0.66           | Efficient |
| Llama 3.3 70B       | `meta.llama3-3-70b-instruct-v1:0`          | $0.72          | $0.72           | Mid       |
| Nova Pro            | `amazon.nova-pro-v1:0`                     | $0.80          | $3.20           | Mid       |
| Nova Lite           | `amazon.nova-lite-v1:0`                    | $0.06          | $0.24           | Fast      |
| Nova Micro          | `amazon.nova-micro-v1:0`                   | $0.035         | $0.14           | Budget    |
| Nova Premier        | `amazon.nova-premier-v1:0`                 | $2.40          | $9.60           | Reasoning |
| Mistral Large 3     | `mistral.mistral-large-3-675b-instruct`    | $2.00          | $6.00           | Flagship  |
| Titan Embeddings v2 | `amazon.titan-embed-text-v2:0`             | $0.02          | N/A             | Embedding |

### Source Provider Pricing Reference

For migration cost comparison (from `pricing-fallback.json` → `source_provider_pricing`):

**Gemini (Standard tier):**

| Model                  | Input (per 1M) | Output (per 1M) |
| ---------------------- | -------------- | --------------- |
| Gemini 3.1 Pro Preview | $1.25          | $10.00          |
| Gemini 2.5 Pro         | $1.25          | $10.00          |
| Gemini 2.5 Flash       | $0.15          | $0.60           |
| Gemini 2.0 Flash       | $0.10          | $0.40           |
| Gemini 2.0 Flash Lite  | $0.075         | $0.30           |

**OpenAI (Standard tier, selected models):**

| Model   | Input (per 1M) | Output (per 1M) |
| ------- | -------------- | --------------- |
| GPT-5.2 | $1.75          | $14.00          |
| GPT-5.1 | $1.25          | $10.00          |
| GPT-4.1 | $2.00          | $8.00           |
| GPT-4o  | $2.50          | $10.00          |
| o3      | $2.00          | $8.00           |
| o4-mini | $1.10          | $4.40           |

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

- `metadata.ai_source` — Source provider (gemini/openai/both/other)
- `ai_architecture.honest_assessment` — Overall migration recommendation
- `ai_architecture.tiered_strategy` — Multi-model tiering (if high volume)
- `ai_architecture.bedrock_models[]` — Selected models with pricing comparison
  - `source_provider_price` — Source provider pricing per 1M tokens
  - `bedrock_price` — Bedrock pricing per 1M tokens
  - `honest_assessment` — Per-model migration recommendation
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

Calculate for all models and present as a comparison. Include the source provider pricing from `aws-design-ai.json` → `bedrock_models[].source_provider_price` for direct comparison:

```
Your token volume: [X]M input + [Y]M output per month
Source provider: [from ai_source: Gemini/OpenAI/Both]

Model                 Bedrock Monthly    vs Source Provider    vs Current GCP    Quality    Capabilities
---------------------------------------------------------------------------------------------------------
Claude Sonnet 4.6     $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Very Good  [Yes/No]
Claude Opus 4.6       $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Excellent  [Yes/No]
Claude Haiku 4.5      $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Good       [Yes/No]
Llama 4 Maverick      $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Good       [Yes/No]
Llama 4 Scout         $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Fair       [Yes/No]
Nova Pro              $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Good       [Yes/No]
Nova Lite             $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Fair       [Yes/No]
Nova Micro            $[calc]            [+/-]$[diff] ([%])    [+/-]$[diff]     Basic      [Yes/No]
---------------------------------------------------------------------------------------------------------
Source Provider       $[source_calc]     --                     --               --         --
Current GCP Billing   $[current]         --                     --               --         --
```

**"vs Source Provider"** — This is the key column for migration decisions. Calculate as:
`(Bedrock monthly - Source provider monthly) / Source provider monthly × 100`

If Bedrock is more expensive for the recommended model, flag this prominently:

> **Note:** Bedrock is [X]% more expensive than [source provider] for [model]. See ROI analysis for justification.

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
  No payback from cost alone — justify with capability, quality, or strategic value
```

### Honest Cost Assessment (When Migration Increases Cost)

If the recommended Bedrock model is more expensive than the source provider, present this transparently:

```
COST IMPACT: Bedrock is [X]% more expensive for [model] compared to [source provider].

At your volume ([X]M input + [Y]M output/month):
  Source provider ([model]):  $[source_cost]/month
  Bedrock ([model]):          $[bedrock_cost]/month
  Additional cost:            $[diff]/month ($[annual_diff]/year)

Migration is justified by:
  ✓ [List applicable non-cost benefits from below]
  ✓ [List applicable optimization opportunities that could close the gap]

Migration is NOT justified if:
  ✗ Cost is the only priority (ai_priority = "cost")
  ✗ No broader AWS adoption planned
  ✗ Current provider meets all quality and feature needs
```

Reference `aws-design-ai.json` → `ai_architecture.honest_assessment` for the design phase's recommendation. If `honest_assessment` is `"recommend_stay"`, present this prominently.

### Non-Cost Benefits

Present these alongside the financial analysis:

- **Model quality** — Claude/Llama may offer better reasoning, coding, or analysis than source models for specific use cases
- **Model flexibility** — Bedrock offers 30+ models from multiple providers; can switch without infrastructure changes
- **Prompt caching** — Claude models support prompt caching (90% cost reduction on cached portions) — not available on Vertex AI or OpenAI Standard tier
- **AWS ecosystem** — If planning broader AWS adoption, AI-only is a low-risk first step
- **Vendor diversification** — Reduces single-vendor risk (Google or OpenAI)
- **Feature access** — Bedrock Guardrails, Knowledge Bases, Agents — managed features not available elsewhere
- **Multi-model strategy** — Route tasks to optimal models (impossible on single-provider platforms)

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

### 6. Multi-Model Tiered Routing (High Volume)

If `ai_constraints.ai_token_volume` is `"high"` or `"very_high"`, or if `aws-design-ai.json` has a `tiered_strategy`, present tiered cost analysis:

```
Current: Single model ([recommended]) at [volume]
  Monthly cost: $[single_model_cost]

Tiered approach (from design phase tiered_strategy):
  Tier 1 — Simple (60%): Nova Micro      $[calc]
  Tier 2 — Moderate (30%): Llama 4 Maverick  $[calc]
  Tier 3 — Complex (10%): Claude Sonnet   $[calc]
  -----------------------------------------------
  Total tiered monthly:                    $[total]
  vs Single model:                         -$[savings] ([X]% savings)
  vs Source provider:                      [+/-]$[diff] ([X]%)
```

Example at 150M input + 75M output tokens/month:

```
Single model (Claude Sonnet):
  Input:  150M × $3.00/1M = $450
  Output: 75M × $15.00/1M = $1,125
  Total: $1,575/month

Tiered approach:
  Tier 1 (60%): 90M in + 45M out × Nova Micro = $3.15 + $6.30 = $9.45
  Tier 2 (30%): 45M in + 22.5M out × Llama 4 Maverick = $10.80 + $21.83 = $32.63
  Tier 3 (10%): 15M in + 7.5M out × Claude Sonnet = $45.00 + $112.50 = $157.50
  Total: $199.58/month (87% savings vs single model)
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
