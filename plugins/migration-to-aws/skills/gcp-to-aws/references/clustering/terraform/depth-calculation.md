# Terraform Clustering: Depth Calculation

Assigns topological depth to every resource via Kahn's algorithm (longest path variant).

## Depth Semantics

- **Depth 0**: Resources with no incoming dependencies (can start immediately)
- **Depth N**: Resources where all dependencies are at depth ≤ N-1, and at least one is at depth N-1

Higher depth = later in deployment sequence.

## Algorithm: Kahn's Algorithm (Longest Path Variant)

### Input

All resources with:

- `address`, `type`
- `dependencies[]` array (addresses of resources this one depends on)

### Step 1: Build Dependency Graph

For each resource:

- Outgoing edges: follow its `dependencies[]` array
- Incoming edges: count how many resources depend on this one
- Store: `in_degree[resource] = count_of_incoming_edges`

### Step 2: Initialize Queue

Create queue of all resources with `in_degree = 0`.

These are depth 0 (no dependencies).

Assign: `depth[resource] = 0` for all queued resources.

### Step 3: Process Queue (Longest Path)

While queue not empty:

1. **Dequeue** resource R
2. **For each** resource D that depends on R (traverse reverse edges):
   - Update: `depth[D] = max(depth[D], depth[R] + 1)`
   - Decrement: `in_degree[D] -= 1`
   - **If** `in_degree[D]` becomes 0: **Enqueue** D

**Note:** "Resources that depend on R" means all resources X where X's `dependencies[]` contains R. This correctly assigns higher depths to dependent resources (which must deploy later).

### Step 4: Cycle Detection

If queue empties but unassigned resources remain:

- **Cycle detected**: Some resources have circular dependencies
- **Action**: Find lowest-confidence edge in cycle; remove it
- **Restart** algorithm
- **Log warning**: "Circular dependency detected and broken: {resources and edges}"

### Step 5: Assign Final Depths

All resources have assigned `depth` field.

Verify: Every resource has `depth ∈ [0, max_depth]`.

## Pseudocode

```
function calculateDepth(resources) {
  // Build graph
  in_degree = {}
  depends_on = {}
  dependents_of = {}  // Reverse adjacency: resource → resources that depend on it
  for each resource R:
    in_degree[R] = count incoming edges
    depends_on[R] = R.dependencies[]
    dependents_of[R] = []

  // Populate dependents_of (reverse edges)
  for each resource R:
    for each D in R.dependencies[]:
      dependents_of[D].append(R)

  // Initialize depth 0
  depth = {}
  queue = [R for R in resources if in_degree[R] == 0]
  for each R in queue:
    depth[R] = 0

  // Process queue (longest path variant)
  while queue not empty:
    R = queue.dequeue()
    for each D in dependents_of[R]:  // Iterate resources that depend on R
      depth[D] = max(depth[D], depth[R] + 1)
      in_degree[D] -= 1
      if in_degree[D] == 0:
        queue.enqueue(D)

  // Cycle check
  if any resource not assigned depth:
    find_and_break_cycle()
    return calculateDepth(resources)  // Retry

  return depth
}
```

## Example

**Resources and dependencies:**

```
A: depends on [] → depth 0
B: depends on [A] → depth 1
C: depends on [A] → depth 1
D: depends on [B, C] → depth 2
```

**Queue trace:**

1. Initial queue: [A] (in_degree 0)
2. Dequeue A, depth[A]=0; enqueue B, C (both now in_degree 0)
3. Dequeue B, depth[B]=1; update depth[D]=max(0,1+1)=2; enqueue D (in_degree 0)
4. Dequeue C, depth[C]=1; update depth[D]=max(2,1+1)=2; D already enqueued
5. Dequeue D, depth[D]=2
6. Queue empty; all depths assigned

**Final**: A:0, B:1, C:1, D:2 ✓

## Deployment Order Guarantee

Resources sorted by ascending depth can deploy in order:

```
Deploy depth 0: A
Deploy depth 1: B, C (parallel OK)
Deploy depth 2: D
```

No dependency violations; parallelism at same depth.
