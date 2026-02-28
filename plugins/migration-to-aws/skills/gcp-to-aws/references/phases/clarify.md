# Phase 2: Clarify Requirements

## Step 0: Prior Run Check

If `$MIGRATION_DIR/clarified.json` already exists:

> "I found existing clarification answers from a previous run. Would you like to:"
>
> A) Re-use these answers and skip questions
> B) Start fresh and re-answer all questions

- If A: skip to Validation Checklist, proceed with existing file.
- If B: continue to Step 1.

## Step 1: Load Inventory

Read `gcp-resource-inventory.json` and `gcp-resource-clusters.json` from `$MIGRATION_DIR/`.

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

**Fallback handling:** If user selects Mode A or B but then declines to answer questions or provides incomplete answers, offer Mode C (use defaults) or Mode D (free-text description) as alternatives. Phase 2 completes using whichever mode provides answers.

## Step 3: Normalize Answers

For Modes A/B (Q1-Q8 answered):

- Validate each answer is within the option set
- If user gives free-form answer, map to closest option
- Store normalized answers

For Mode C:

- Use all defaults from `shared/clarify-questions.md`

For Mode D (free-text):

1. Parse user text to extract answers for Q1-Q8
   - Look for keywords matching question option descriptions
   - For each question, mark as "extracted" if found or "default" if not

2. **Confirmation step**: Present to user:

   ```
   Based on your requirements, I extracted:
   - Q1 (Timeline): [extracted value]
   - Q2 (Primary concern): [extracted value]
   - Q3 (Team experience): [default value] ← using default
   - ...

   Accept these, or re-run with Mode A/B to override?
   ```

3. If user accepts: store answers with source tracking (extracted vs default)
4. If user declines: fall back to Mode A or B

## Step 4: Write Clarified Output

Write `clarified.json` to `.migration/[MMDD-HHMM]/` directory.

**Schema:** See `references/shared/output-schema.md` → `clarified.json (Phase 2 output)` section for complete schema and field documentation.

**Key fields:**

- `mode`: "A", "B", "C", or "D" (answering mode selected in Step 2)
- `answers`: Object with keys q1_timeline through q8_compliance
- `timestamp`: ISO 8601 timestamp

## Validation Checklist

Before handing off to Design:

- `clarified.json` written to `$MIGRATION_DIR/`
- `mode` field is one of "A", "B", "C", or "D"
- `answers` object contains keys q1 through q8
- Each answer value is within the documented option set (or mapped from free-form)
- `timestamp` is a valid ISO 8601 timestamp
- For Mode D: source tracking present (extracted vs default per answer)
- No null values in answers
- Output is valid JSON

## Step 5: Update Phase Status

Update `$MIGRATION_DIR/.phase-status.json`:

- Set `phases.clarify.status` to `"completed"`
- Set `phases.clarify.timestamp` to current ISO 8601 timestamp
- Set `phases.clarify.outputs` to `["clarified.json"]`
- Update `last_updated` to current timestamp

Output to user: "Clarification complete. Proceeding to Phase 3: Design AWS Architecture."

## Scope Boundary

**This phase covers requirements gathering ONLY.**

FORBIDDEN — Do NOT include ANY of:
- Detailed AWS architecture or service configurations
- Code migration examples or SDK snippets
- Detailed cost calculations
- Migration timelines or execution plans
- Terraform generation

**Your ONLY job: Understand what the user needs. Nothing else.**
