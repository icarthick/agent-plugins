# .phase-status.json

Hierarchical phase tracking with per-phase metadata. This is the SINGLE source of truth for the `.phase-status.json` schema. All steering files reference this definition.

```json
{
  "migration_id": "0226-1430",
  "started_at": "2026-02-26T14:30:00Z",
  "last_updated": "2026-02-26T15:35:22Z",
  "project_directory": "/path/to/project",
  "input_files_detected": {
    "terraform_files": 12,
    "terraform_lines": 850,
    "app_code_languages": ["python"]
  },
  "discovery_outputs": [
    "gcp-resource-inventory.json",
    "gcp-resource-clusters.json"
  ],
  "phases": {
    "discover": {
      "status": "completed",
      "timestamp": "2026-02-26T14:31:00Z",
      "outputs": ["gcp-resource-inventory.json", "gcp-resource-clusters.json"]
    },
    "clarify": {
      "status": "completed",
      "timestamp": "2026-02-26T14:32:00Z",
      "outputs": ["preferences.json"]
    },
    "design": { "status": "in_progress", "timestamp": null, "outputs": [] },
    "estimate": { "status": "pending", "timestamp": null, "outputs": [] },
    "generate": { "status": "pending", "timestamp": null, "outputs": [] },
    "feedback": { "status": "pending", "timestamp": null, "outputs": [] }
  }
}
```

**Field Definitions:**

| Field                  | Type             | Set When                                                                                |
| ---------------------- | ---------------- | --------------------------------------------------------------------------------------- |
| `migration_id`         | string           | Created (matches folder name, never changes)                                            |
| `started_at`           | ISO 8601         | Created (never changes)                                                                 |
| `last_updated`         | ISO 8601         | After each phase update                                                                 |
| `project_directory`    | string           | Start of Discover                                                                       |
| `input_files_detected` | object           | End of Discover (terraform_files count, terraform_lines count, app_code_languages list) |
| `discovery_outputs`    | string[]         | End of Discover (list of artifact filenames produced)                                   |
| `phases.*.status`      | string           | Phase transitions: `"pending"` -> `"in_progress"` -> `"completed"`                      |
| `phases.*.timestamp`   | ISO 8601 or null | Phase completion                                                                        |
| `phases.*.outputs`     | string[]         | Phase completion (files created by that phase)                                          |

**Rules:**

- `phases.*.status` progresses: `"pending"` -> `"in_progress"` -> `"completed"`. Never goes backward.
- `discovery_outputs` is populated at the end of Discover. Subsequent phases read this to determine available artifacts.
- `input_files_detected` is populated during Discover and never changed.
