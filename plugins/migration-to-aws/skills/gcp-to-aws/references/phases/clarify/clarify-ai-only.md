# AI-Only Migration — Clarify Requirements

**Standalone flow** — Used when ONLY `ai-workload-profile.json` exists (no infrastructure or billing artifacts). Infrastructure stays on GCP; only AI/LLM calls move to AWS Bedrock.

This file replaces the category-based question system for AI-only migrations. It produces the same `preferences.json` output but with `design_constraints` limited to region and compliance, and `ai_constraints` fully populated.

---

## Step 1: Present AI Detection Summary

> **AI-Only Migration Detected**
> Your project has AI workloads but no infrastructure artifacts (Terraform, billing). I'll focus on migrating your AI/LLM calls to AWS Bedrock while your infrastructure stays on GCP.
>
> **AI source:** [from `summary.ai_source`]
> **Models detected:** [from `models[].model_id`]
> **Capabilities in use:** [from `integration.capabilities_summary` where true]
> **Integration pattern:** [from `integration.pattern`] via [from `integration.primary_sdk`]
> **Gateway/router:** [from `integration.gateway_type`, or "None (direct SDK)"]
> **Frameworks:** [from `integration.frameworks`, or "None"]

---

## Step 2: Ask Questions (Q1–Q10)

Present all questions at once with a progress indicator. Questions use their own numbering (Q1–Q10), independent of the full migration numbering.

```
I have a few questions to tailor your AI migration plan.
You can answer each, skip individual ones (I'll use sensible defaults),
or say "use all defaults" to proceed.
```

---

### Q1 — What AI framework or orchestration layer are you using? (select all that apply)

Same decision logic, auto-detect signals, combination logic, and interpretation as Q14 in `clarify-ai.md`. Refer to that file for full details.

**Auto-detect signals** — scan application code before asking:

- No AI framework imports, raw HTTP calls → A
- LiteLLM, OpenRouter, PortKey, Helicone, Kong, Apigee → B
- LangChain/LangGraph/LlamaIndex → C
- CrewAI, AutoGen, custom multi-agent → D
- OpenAI Agents SDK / Swarm → E
- MCP / A2A protocol → F
- Vapi, Bland.ai, Retell, Whisper → G

> How your AI calls reach the model determines migration effort.
>
> A) No framework — direct API calls
> B) LLM router/gateway (LiteLLM, OpenRouter, PortKey, Kong, Apigee)
> C) LangChain / LangGraph
> D) Multi-agent framework (CrewAI, AutoGen, custom)
> E) OpenAI Agents SDK / custom agent loop
> F) MCP servers or A2A protocol
> G) Voice/conversational agent platform (Vapi, Retell, Bland.ai)

Interpret: same as Q14 in `clarify-ai.md` → `ai_framework` array.

Default: _(auto-detect)_ — fall back to `["direct"]`.

---

### Q2 — What matters most for your AI application?

Same decision logic as Q16 in `clarify-ai.md`.

> This determines our model selection strategy.
>
> A) Best quality/reasoning
> B) Fastest speed
> C) Lowest cost
> D) Specialized capability (covered in Q10)
> E) Balanced
> F) I don't know

Interpret: same as Q16 → `ai_priority`.

Default: E — `ai_priority: "balanced"`.

---

### Q3 — Approximately how much are you spending on OpenAI or Gemini per month?

Same decision logic as Q15 in `clarify-ai.md`.

> AI spend helps me calculate migration credits eligibility and establish a Bedrock cost baseline.
>
> A) < $500/month
> B) $500–$2,000/month
> C) $2,000–$10,000/month
> D) > $10,000/month
> E) I don't know

Interpret: same as Q15 → `ai_monthly_spend`.

Default: B — `ai_monthly_spend: "$500-$2K"`.

---

### Q4 — Cross-cloud API call concerns

**Rationale:** AI-only migrations keep infrastructure on GCP while moving AI calls to AWS Bedrock. Cross-region API calls add latency and potential egress costs that may affect the recommendation. This question is unique to AI-only migrations.

**Context for user:** Explain the tradeoff concretely:

- **Yes — latency critical** — AI calls are in the hot path and every 20–50ms matters (e.g., real-time chat, autocomplete, live transcription)
- **No — latency acceptable** — AI calls are async or users expect a brief wait (e.g., background processing, batch jobs, report generation)
- **Concerned about egress costs** — sending large payloads (images, documents, audio) between GCP and AWS
- **Want to test first** — run both providers in parallel before committing

> Since your infrastructure stays on GCP, AI calls to Bedrock will cross cloud boundaries. This affects latency and data transfer costs.
>
> A) Yes — latency critical, AI calls are in the hot path
> B) No — latency acceptable, async or users can wait
> C) Concerned about egress costs for large payloads
> D) Want to test first — run both providers in parallel

| Answer                 | Recommendation Impact                                                                                                     |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Latency critical       | VPC endpoint for Bedrock strongly recommended; us-east-1 or closest region to GCP deployment; latency benchmarks included |
| Latency acceptable     | Standard Bedrock endpoint; region selected for cost/availability                                                          |
| Concerned about egress | AWS PrivateLink for Bedrock recommended; egress cost analysis included                                                    |
| Want to test first     | Phased migration plan; parallel running guidance included                                                                 |

Interpret:

```
A -> cross_cloud: "latency-critical" — VPC endpoint; closest region to GCP
B -> cross_cloud: "latency-acceptable" — Standard endpoint; region by cost
C -> cross_cloud: "egress-concerned" — PrivateLink; egress analysis
D -> cross_cloud: "test-first" — Phased migration; parallel running
```

Default: B — `cross_cloud: "latency-acceptable"`.

---

### Q5 — Which model are you currently using?

Same decision logic and override hierarchy as Q19 in `clarify-ai.md`.

**Override hierarchy:**

1. Q10 special features — hard overrides
2. Q2 priority — adjusts within Claude family
3. Q7/Q8 volume and latency — may adjust toward provisioned throughput
4. Q5 source model — baseline only

> The model you're currently using helps me recommend the closest Bedrock equivalent.
>
> A) Gemini Flash (1.5/2.0/2.5 Flash)
> B) Gemini Pro (1.5/2.5/3 Pro)
> C) GPT-3.5 Turbo
> D) GPT-4 / GPT-4 Turbo
> E) GPT-4o
> F) GPT-5 / GPT-5.x
> G) o-series (o1, o3)
> H) Other / Multiple models
> I) I don't know

Interpret: same as Q19 in `clarify-ai.md` → `ai_model_baseline`.

Default: _(auto-detect)_ — fall back to Q2 priority-based selection.

---

### Q6 — Do you need vision or just text?

Same decision logic as Q20 in `clarify-ai.md`.

> Vision capability limits which models are available.
>
> A) Text only
> B) Vision required
> C) Audio/Video inputs needed

Interpret: same as Q20 → `ai_vision`.

Default: A — no constraint (text only).

---

### Q7 — Monthly AI usage volume

**Rationale:** Volume determines whether on-demand or provisioned throughput is more cost-effective.

> Volume determines pricing strategy — on-demand vs provisioned throughput.
>
> A) Low (< 1M tokens/month)
> B) Medium (1–10M tokens/month)
> C) High (10–100M tokens/month)
> D) Very high (> 100M tokens/month)
> E) I don't know

| Answer    | Recommendation Impact                                                |
| --------- | -------------------------------------------------------------------- |
| Low       | On-demand pricing; no provisioned throughput needed                  |
| Medium    | On-demand with prompt caching analysis                               |
| High      | Provisioned throughput analysis; prompt caching strongly recommended |
| Very high | Provisioned throughput required; dedicated capacity planning         |

Interpret:

```
A -> ai_token_volume: "<1M" — On-demand; no provisioned throughput
B -> ai_token_volume: "1M-10M" — On-demand; prompt caching analysis
C -> ai_token_volume: "10M-100M" — Provisioned throughput analysis; prompt caching
D -> ai_token_volume: ">100M" — Provisioned throughput required; capacity planning
E -> same as default (B)
```

Default: B — `ai_token_volume: "1M-10M"`.

---

### Q8 — How important is response speed?

Same decision logic as Q21 in `clarify-ai.md`.

> Latency requirements can override cost and quality preferences.
>
> A) Critical (< 500ms)
> B) Important (< 2s)
> C) Flexible (2–10s)

Interpret: same as Q21 → `ai_latency`.

Default: B — `ai_latency: "important"`.

---

### Q9 — How complex are your AI tasks?

Same decision logic as Q22 in `clarify-ai.md`.

> Task complexity determines whether cheaper models can handle your workload.
>
> A) Simple (classification, short summaries, extraction)
> B) Moderate (analysis, structured content, few-shot)
> C) Complex (multi-step reasoning, tool use, agentic workflows)

Interpret: same as Q22 → `ai_complexity`.

Default: B — `ai_complexity: "moderate"`.

---

### Q10 — Specialized features needed

Same decision logic as Q17 in `clarify-ai.md`.

> Some features are only available in specific models. What's your most critical specialized requirement?
>
> A) Function calling / Tool use
> B) Ultra-long context (> 300K tokens)
> C) Extended thinking / Chain-of-thought
> D) Prompt caching
> E) RAG optimization
> F) Agentic workflows
> G) Real-time speed (< 500ms)
> H) Multimodal with image generation
> I) Real-time conversational speech
> J) None — standard features sufficient

Interpret: same as Q17 → `ai_critical_feature`.

Default: J — no additional override.

---

## Step 3: Write preferences.json

Write `$MIGRATION_DIR/preferences.json` with AI-only structure:

```json
{
  "metadata": {
    "timestamp": "<ISO timestamp>",
    "migration_type": "ai-only",
    "discovery_artifacts": ["ai-workload-profile.json"],
    "questions_asked": ["Q1", "Q2", "Q3", "Q4", "Q5", "Q6", "Q7", "Q8", "Q9", "Q10"],
    "questions_defaulted": [],
    "questions_skipped_extracted": [],
    "questions_skipped_not_applicable": []
  },
  "design_constraints": {
    "target_region": { "value": "us-east-1", "chosen_by": "derived" }
  },
  "ai_constraints": {
    "ai_framework": { "value": ["langchain"], "chosen_by": "extracted" },
    "ai_priority": { "value": "balanced", "chosen_by": "user" },
    "ai_monthly_spend": { "value": "$500-$2K", "chosen_by": "user" },
    "cross_cloud": { "value": "latency-acceptable", "chosen_by": "user" },
    "ai_model_baseline": { "value": "claude-sonnet-4-6", "chosen_by": "derived" },
    "ai_vision": { "value": "text-only", "chosen_by": "user" },
    "ai_token_volume": { "value": "1M-10M", "chosen_by": "user" },
    "ai_latency": { "value": "important", "chosen_by": "user" },
    "ai_complexity": { "value": "moderate", "chosen_by": "user" },
    "ai_critical_feature": { "value": "none", "chosen_by": "user" },
    "ai_capabilities_required": {
      "value": ["text_generation", "streaming"],
      "chosen_by": "derived"
    }
  }
}
```

### Schema Notes (AI-Only)

1. `metadata.migration_type` is `"ai-only"` — downstream phases use this to skip infrastructure design/estimation.
2. `design_constraints` is minimal — only `target_region` (derived from GCP deployment region or cross-cloud latency preference).
3. `ai_constraints.cross_cloud` is unique to AI-only migrations — not present in full migration preferences.
4. `ai_constraints.ai_token_volume` uses different tiers than full migration Q18 — more granular for AI-only cost analysis.
5. All other schema rules from `clarify.md` apply (value/chosen_by fields, no nulls, derived capabilities).

---

## Step 4: Update Phase Status

Update `$MIGRATION_DIR/.phase-status.json`:

- Set `phases.clarify.status` to `"completed"`
- Set `phases.clarify.timestamp` to current ISO 8601 timestamp
- Set `phases.clarify.outputs` to `["preferences.json"]`
- Update `last_updated` to current timestamp

Output to user: "Clarification complete. Proceeding to Phase 3: Design AI Migration Architecture."
