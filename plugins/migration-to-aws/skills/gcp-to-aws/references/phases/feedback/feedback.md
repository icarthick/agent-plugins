# Phase 6: Feedback (Optional)

Collects optional user feedback and anonymized usage telemetry to improve the migration tool. Submits to the Pulse survey API.

**Execute ALL steps in order. Do not skip or deviate.**

## Prerequisites

Read `$MIGRATION_DIR/.phase-status.json`. Verify `phases.discover.status == "completed"`. If not: **STOP**. Output: "Feedback requires at least the Discover phase to be completed."

## Step 1: Build Trace

Load `references/phases/feedback/feedback-trace.md` and execute it. This produces `$MIGRATION_DIR/trace.json`.

If trace building fails: log the error, set `trace_included` to `false`, and continue to Step 2.

## Step 2: Ask Feedback Questions

Present 5 optional questions. **Auto-skip** any question about a phase that did not complete (check `phases.*.status` in `.phase-status.json`).

Preface with: "These 5 questions are optional. Press Enter or type 'skip' to skip any question."

**Q1 — Time saved** (always ask):
"How much time did this tool save you compared to manual migration planning?"

- `[A]` Saved days of work
- `[B]` Saved hours of work
- `[C]` About the same as manual
- `[D]` Slower than manual
- `[S]` Skip

**Q2 — Mapping accuracy** (skip if `phases.design.status != "completed"`):
"How accurate were the GCP-to-AWS service mappings?"

- `[A]` Very accurate — minimal changes needed
- `[B]` Mostly accurate — a few adjustments
- `[C]` Partially accurate — significant changes needed
- `[D]` Not accurate — had to redo most mappings
- `[S]` Skip

**Q3 — Terraform usability** (skip if `phases.generate.status != "completed"`):
"How usable were the generated Terraform configurations?"

- `[A]` Ready to use with minor tweaks
- `[B]` Good starting point, needed moderate changes
- `[C]` Useful as reference only
- `[D]` Not useful
- `[S]` Skip

**Q4 — Cost accuracy** (skip if `phases.estimate.status != "completed"`):
"How accurate were the cost estimates compared to your expectations?"

- `[A]` Very close to expected
- `[B]` Roughly in the right range
- `[C]` Significantly off
- `[D]` Cannot evaluate yet
- `[S]` Skip

**Q5 — Weakest phase** (always ask, multi-select):
"Which phase(s) need the most improvement? Select all that apply."

- `[A]` Discover (resource detection)
- `[B]` Clarify (requirements gathering)
- `[C]` Design (service mapping)
- `[D]` Estimate (cost analysis)
- `[E]` Generate (artifact creation)
- `[S]` Skip

Record each answer. Skipped and auto-skipped questions get `null`.

## Step 3: Show Preview

Display the trace JSON and collected answers to the user:

```
--- Feedback Preview ---

Answers:
  Q1 (Time saved): <answer or "skipped">
  Q2 (Mapping accuracy): <answer or "skipped" or "auto-skipped (phase not completed)">
  Q3 (Terraform usability): <answer or "skipped" or "auto-skipped (phase not completed)">
  Q4 (Cost accuracy): <answer or "skipped" or "auto-skipped (phase not completed)">
  Q5 (Weakest phase): <answer or "skipped">

Trace data (anonymized):
  <pretty-printed trace.json — no resource names, file paths, or account IDs>

--- End Preview ---
```

Ask: "Submit this feedback? [Y] Yes / [N] No"

If user declines: skip to Step 5 with `submitted: false`.

## Step 4: Submit to Pulse API

Generate a random visitor ID:

```bash
VISITOR_ID=$(uuidgen)
```

Compress trace for the payload:

```bash
COMPRESSED_TRACE=$(python3 -c "import gzip,base64,sys; print(base64.b64encode(gzip.compress(sys.stdin.buffer.read())).decode())" < "$MIGRATION_DIR/trace.json")
```

Build the GraphQL payload using the **full field object format**. Each field must include `prompt`, `choices` (where applicable), `type`, and `fieldId`. Map answers to choice indices (0-based). Use `null` for skipped questions.

**Field IDs:**

| Question               | fieldId                                | type                         |
| ---------------------- | -------------------------------------- | ---------------------------- |
| Q1 Time saved          | `095b18e8-8543-43f9-bd20-a0e1d67d0cc4` | `MultipleChoiceSingleSelect` |
| Q2 Mapping accuracy    | `7e2b699d-3b28-4675-abb7-08cd9e9c4b4e` | `MultipleChoiceSingleSelect` |
| Q3 Terraform usability | `13ace99a-42d6-4a7e-9436-f8d871388e38` | `MultipleChoiceSingleSelect` |
| Q4 Cost accuracy       | `7e60d3c0-e266-4887-ad98-4e37e6bf12c2` | `MultipleChoiceSingleSelect` |
| Q5 Weakest phase       | `67d0d3a5-c94e-4c6b-aedb-d33794740477` | `MultipleChoiceMultiSelect`  |
| Q6 Trace (auto)        | `6729dcba-fd33-40c0-900c-8d75a8858b61` | `FreeFormText`               |

**Survey ID:** `Survey-39lhkKkoENCwOL3piwU9wkOkSHn`

Submit via curl:

```bash
curl -s -X POST https://api.pulse.amazon/ \
  -H "Content-Type: application/json" \
  -H "Cookie: visitor_id=$VISITOR_ID" \
  -d '<GraphQL mutation submitSurveyResponse payload>'
```

Record the HTTP status and response. If curl fails or returns non-200: set `submitted: false` and continue.

## Step 5: Write feedback.json

Write `$MIGRATION_DIR/feedback.json`:

```json
{
  "timestamp": "<ISO 8601>",
  "submitted": true,
  "submission_id": "<from API response, or null>",
  "visitor_id": "<generated UUID>",
  "phases_completed_at_feedback": ["<list of completed phases>"],
  "answers": {
    "q1_time_saved": "<A|B|C|D|null>",
    "q2_mapping_accuracy": "<A|B|C|D|null>",
    "q3_terraform_usability": "<A|B|C|D|null>",
    "q4_cost_accuracy": "<A|B|C|D|null>",
    "q5_weakest_phase": ["<selected letters>"]
  },
  "trace_included": true,
  "auto_skipped": ["<question IDs skipped because phase not completed>"]
}
```

If submission failed: set `"submitted": false` and `"submission_id": null`. Still record answers.

## Step 6: Update Phase Status

Update `$MIGRATION_DIR/.phase-status.json`:

- Set `phases.feedback.status` to `"completed"`
- Set `phases.feedback.timestamp` to current ISO 8601 timestamp
- Set `phases.feedback.outputs` to `["feedback.json", "trace.json"]`
- Update `last_updated` to current timestamp

Output to user: "Feedback recorded. Thank you for helping improve this tool."

After feedback completes, return control to the workflow execution in SKILL.md. The calling checkpoint determines whether to advance to the next phase or end the migration.
