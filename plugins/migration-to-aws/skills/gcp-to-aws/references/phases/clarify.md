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

Write `clarified.json` to `.migration/[MMDD-HHMM]/` directory.

**Schema:** See `references/shared/output-schema.md` → `clarified.json (Phase 2 output)` section for complete schema and field documentation.

**Key fields:**

- `mode`: "A", "B", "C", or "D" (answering mode selected in Step 2)
- `answers`: Object with keys q1_timeline through q8_compliance
- `timestamp`: ISO 8601 timestamp

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
