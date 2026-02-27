# Phase 2: Clarify Requirements

## Step 1: Load Inventory

Read `gcp-resource-inventory.json` and `gcp-resource-clusters.json` from `.migration/*/`.

## Step 2: Select Answering Mode

Present 4 modes to user:

| Mode  | Style       | When to use                                  |
| ----- | ----------- | -------------------------------------------- |
| **A** | All at once | "I'll answer all 8 questions together"       |
| **B** | One-by-one  | "Ask me each question separately"            |
| **C** | Defaults    | "Use default answers (no questions)"         |
| **D** | Free text   | "I'll describe requirements in my own words" |

If user selects **Mode C** or **Mode D**: use default answers from `shared/clarify-questions.md` and continue to Step 3.

If user selects **Mode A** or **Mode B**: Present all 8 questions (from `shared/clarify-questions.md`), collect answers, continue to Step 3.

## Step 3: Normalize Answers

For Modes A/B (Q1-Q8 answered):

- Validate each answer is within the option set
- If user gives free-form answer, map to closest option
- Store normalized answers

For Modes C/D:

- Use defaults from `shared/clarify-questions.md`

## Step 4: Write Clarified Output

Write `clarified.json`:

```json
{
  "mode": "A|B|C|D",
  "answers": {
    "q1_timeline": "3-6 months",
    "q2_primary_concern": "cost",
    "q3_team_experience": "moderate",
    "q4_traffic_profile": "predictable",
    "q5_database_requirements": "structured",
    "q6_cost_sensitivity": "high",
    "q7_multi_cloud": "no",
    "q8_compliance": "none"
  },
  "timestamp": "2026-02-26T14:30:00Z"
}
```

## Step 5: Update Phase Status

Update `.phase-status.json`:

```json
{
  "phase": "clarify",
  "status": "completed",
  "timestamp": "2026-02-26T14:30:00Z",
  "version": "1.0.0"
}
```

Output to user: "Clarification complete. Proceeding to Phase 3: Design AWS Architecture."
