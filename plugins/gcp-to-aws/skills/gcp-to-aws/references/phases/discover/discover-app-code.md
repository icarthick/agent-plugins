# Discover Phase: App Code Discovery (v1.1+)

**Status**: Not yet implemented (v1.1 feature).

## Overview

This discoverer will scan application code for `google.cloud.*` imports and detect compute workload types, AI services, and data dependencies.

## Expected Behavior (v1.1+)

- Scan Python, Node.js, Go codebases for `google.cloud.*` imports
- Detect compute workload types (batch, streaming, async jobs)
- Identify AI/ML workload characteristics
- Output: `app_code_resources.json` with detected services and confidence

## Expected Output Schema (v1.1+)

```json
{
  "app_code_resources": [
    {
      "service": "Cloud Run",
      "detected_imports": ["google.cloud.storage", "google.cloud.firestore"],
      "workload_type": "web",
      "evidence": "code analysis",
      "confidence": 0.92
    }
  ]
}
```

## Integration with Unify Phase

`unify-resources.md` will check for `app_code_resources.json` and merge service evidence into final inventory.

**Current Action**: Skip if v1.1 not available. `discover.md` will not call this discoverer in v1.0.
