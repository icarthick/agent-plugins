# Phase 2: Clarify Requirements

**Phase 2 of 5** — Ask adaptive questions before mapping begins, then interpret answers into ready-to-apply design constraints.

The output — `preferences.json` — is consumed directly by Design and Estimate without any further interpretation.

Questions are organized into **six named categories (A-F)** with documented firing rules. The number of questions depends on which discovery artifacts exist and which GCP services are detected.

---

## Step 0: Prior Run Check

If `$MIGRATION_DIR/preferences.json` already exists:

> "I found existing migration preferences from a previous run. Would you like to:"
>
> A) Re-use these preferences and skip questions
> B) Start fresh and re-answer all questions

- If A: skip to Validation Checklist, proceed with existing file.
- If B: continue to Step 1.

---

## Step 1: Read Inventory and Determine Available Artifacts

Read `$MIGRATION_DIR/` and check which discovery outputs exist:

- `gcp-resource-inventory.json` + `gcp-resource-clusters.json` — infrastructure discovered
- `billing-profile.json` — billing data parsed (v1.2+, not yet implemented)
- `ai-workload-profile.json` — AI workloads detected (v1.1+, not yet implemented)

At least one discovery artifact must exist to proceed.

Present a discovery summary:

**If `gcp-resource-inventory.json` exists:**

> **Infrastructure discovered:** [total resources] GCP resources across [cluster count] clusters
> **Top resource types:** [list top 3-5 types]

---

## Step 2: Extract Known Information

Before generating questions, scan the inventory to extract values that are already known:

1. **GCP regions** — Extract all GCP regions from the inventory. Map to the closest AWS region as a suggested default for Q1.
2. **Resource types present** — Build a set of resource types: compute (Cloud Run, Cloud Functions, GKE, GCE), database (Cloud SQL, Spanner, Memorystore), storage (Cloud Storage), messaging (Pub/Sub).

Record extracted values. Questions whose answers are fully determined by extraction will be skipped and the extracted value used directly with `chosen_by: "extracted"`.

---

## Step 3: Generate Questions by Category

### Category Definitions and Firing Rules

| Category | Name               | Firing Rule                                                                                                         | Questions                                                                                              |
| -------- | ------------------ | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **A**    | Global/Strategic   | **Always fires**                                                                                                    | Q1 (location), Q2 (compliance), Q3 (GCP spend), Q4 (multi-cloud), Q5 (uptime), Q6 (maintenance window) |
| **B**    | Configuration Gaps | Inventory exists with `metadata.source == "billing"` AND at least one resource has `config_confidence == "assumed"` | _(v1.2 stub — fires when billing discovery is implemented)_                                            |
| **C**    | Compute Model      | Compute resources present (Cloud Run, Cloud Functions, GKE, GCE)                                                    | Q7 (K8s sentiment), Q8 (WebSocket), Q9 (Cloud Run traffic), Q10 (Cloud Run spend)                      |
| **D**    | Database Model     | Database resources present (Cloud SQL, Spanner, Memorystore)                                                        | Q11 (DB scale), Q12 (DB I/O)                                                                           |
| **E**    | Migration Posture  | **Disabled by default** — requires explicit user opt-in                                                             | HA upgrades, right-sizing from billing utilization                                                     |
| **F**    | AI/Bedrock         | `ai-workload-profile.json` exists                                                                                   | _(v1.1 stub — fires when AI discovery is implemented)_                                                 |

**Apply firing rules:**

1. Always include Category A.
2. Check inventory `metadata.source` — if `"billing"` with assumed configs, include Category B. _(v1.2: currently never fires)_
3. Check for compute resources — if present, include Category C. Within Category C, skip Q7 if no GKE is present. Skip Q9/Q10 if no Cloud Run is present.
4. Check for database resources — if present, include Category D. Skip Q11/Q12 if no Cloud SQL is present.
5. Category E is disabled by default. Do not present unless user opts in.
6. Check for `ai-workload-profile.json` — if present, include Category F. _(v1.1: currently never fires)_

**If no IaC, billing data, or code is available** (empty discovery): fire only Category A. All service-specific categories are skipped.

---

### Early-Exit Rules

Apply these before presenting questions:

- **Q4 = "Yes, multi-cloud required"** — Immediately record `compute: "eks"`. Skip Q7 (Kubernetes sentiment) — all container workloads resolve to EKS.
- **Q9/Q10 N/A** — Cloud Run not present, auto-skip.
- **Q11/Q12 N/A** — Cloud SQL not present, auto-skip.

---

## Category A — Global/Strategic (Always Fires)

Present questions grouped by section. Use a conversational tone with brief context explaining why each question matters. Show a progress indicator: **"Question N of M"** where M is the total number of questions being asked (after filtering).

---

**Q1 — Where are your users located?**

> I need to understand your user base to recommend the right AWS region and CDN strategy.
>
> A) Single region (e.g., US-only, EU-only)
> B) Multi-region (2-3 regions, e.g., US + EU)
> C) Global (users worldwide, latency critical)
> D) I don't know

Interpret:

```
A -> target_region: "<closest AWS region to GCP region in inventory>"
B -> target_region: "<closest AWS region>", replication: "cross-region"
C -> target_region: "<closest AWS region>", replication: "cross-region", cdn: "required"
D -> same as default (A)
```

_Note: Multi-region infrastructure (Aurora Global Database, multi-region compute) is NOT determined by geography alone — it requires Q5 = Catastrophic to justify the cost and complexity._

Default: A — single region, closest AWS region to GCP region in inventory.

---

**Q2 — Do you have any compliance or regulatory requirements?**

> Compliance requirements determine which AWS services, regions, and configurations are available to you. This gates the entire architecture.
>
> A) None — No specific compliance requirements
> B) SOC 2 / ISO 27001 — Security and availability standards
> C) PCI DSS — Payment card data handling
> D) HIPAA — Healthcare data
> E) FedRAMP / Government — Federal compliance
> F) GDPR / Data residency — EU data sovereignty requirements
> G) I don't know
>
> _(Multiple selections allowed)_

Interpret:

```
A -> (no constraint written — full service catalog available, any region)
B -> compliance: ["soc2"] — CloudTrail, Config, Security Hub enabled; encryption at rest required
C -> compliance: ["pci"] — Dedicated VPC, WAF required, strict segmentation
D -> compliance: ["hipaa"] — BAA-eligible services only, encryption mandatory, us-east-1/us-west-2 preferred
E -> compliance: ["fedramp"] — GovCloud regions required (us-gov-east-1, us-gov-west-1)
F -> compliance: ["gdpr"] — EU regions required (eu-west-1, eu-central-1), data residency constraints
G -> same as default (A) — no constraint assumed; verify with your compliance team before production
```

Default: A — no constraint.

---

**Q3 — Approximately how much are you spending on GCP per month in total?**

> Total GCP spend helps me estimate AWS credits eligibility and provides a cost baseline for the migration plan.
>
> A) < $1,000/month
> B) $1,000-$5,000/month
> C) $5,000-$20,000/month
> D) $20,000-$100,000/month
> E) > $100,000/month
> F) I don't know

Interpret:

```
A -> gcp_monthly_spend: "<$1K" — AWS Activate credits eligibility (~$5K-$25K)
B -> gcp_monthly_spend: "$1K-$5K" — IW Migrate credits at 25% of ARR (~$3K-$15K/yr)
C -> gcp_monthly_spend: "$5K-$20K" — IW Migrate credits at 25% of ARR (~$15K-$60K/yr); Reserved Instance recommendations
D -> gcp_monthly_spend: "$20K-$100K" — IW Migrate credits at 25% of ARR (~$60K-$300K/yr); Savings Plans analysis; AWS Specialist consultation eligible
E -> gcp_monthly_spend: ">$100K" — MAP eligibility (>$500K ARR); Enterprise support tier; dedicated migration team
F -> same as default (B)
```

Default: B — `gcp_monthly_spend: "$1K-$5K"`.

---

**Q4 — Do you need to run workloads across multiple cloud providers?**

> Multi-cloud portability is an immediate decision point — if required, Kubernetes (EKS) is the only portable abstraction, and we can skip several compute questions.
>
> A) Yes, multi-cloud required
> B) No, AWS-only is acceptable
> C) I don't know

Interpret:

```
A -> compute: "eks" — Immediate EKS recommendation. EARLY EXIT: skip Q7.
B -> (no constraint written — full compute decision tree continues)
C -> same as default (B) — assume AWS-only
```

Default: B — no constraint, evaluate full compute options.

---

**Q5 — If your application went down unexpectedly right now, what would happen?**

> Availability requirements drive database engine selection, deployment topology, and whether multi-AZ is mandatory.
>
> A) INCONVENIENT — Users can wait, brief outages tolerable (5-30 min)
> B) SIGNIFICANT ISSUE — Customers frustrated, revenue loss
> C) MISSION-CRITICAL — Cannot tolerate outages, SLA violations
> D) CATASTROPHIC — Users in multiple regions lose access simultaneously, data must stay in sync globally with no tolerance for lag
> E) I don't know

Interpret:

```
A -> availability: "single-az" — Single-AZ RDS acceptable, standard deployment
B -> availability: "multi-az" — Multi-AZ RDS required, ALB with health checks, auto-scaling
C -> availability: "multi-az-ha" — Aurora Multi-AZ, multi-AZ mandatory, Route 53 health checks
D -> IF Q1 = C (Global): availability: "multi-region" — Aurora Global Database + active-active multi-region + Route 53 failover
     IF Q1 = A or B: availability: "multi-az-ha" — Aurora Multi-AZ with aggressive RTO/RPO (global infra not warranted without global users)
E -> same as default (B) — assume multi-AZ for safety
```

Default: B — `availability: "multi-az"`.

---

**Q6 — Do you have a scheduled maintenance window where downtime is acceptable?**

> The maintenance window determines your migration cutover strategy and which database migration tooling we recommend. Zero-downtime migrations require significantly more complex infrastructure.
>
> A) Yes — weekly maintenance window (e.g., Sunday 2-4am)
> B) Yes — monthly maintenance window only
> C) No — zero downtime required, must use blue/green or rolling deployment
> D) Flexible — we can schedule one if needed
> E) I don't know

Interpret:

```
A -> cutover_strategy: "maintenance-window-weekly" — pg_dump/pg_restore recommended; standard cutover with DNS switchover
B -> cutover_strategy: "maintenance-window-monthly" — pg_dump/pg_restore recommended; blue/green for app layer
C -> cutover_strategy: "zero-downtime" — AWS DMS required for live DB replication; blue/green deployment; Route 53 weighted routing
D -> cutover_strategy: "flexible" — Recommend scheduling weekly window for pg_dump approach; DMS fallback
E -> same as default (D) — assume flexible
```

Default: D — `cutover_strategy: "flexible"`.

---

## Category B — Configuration Gaps (v1.2 Stub)

_Fire when:_ `gcp-resource-inventory.json` exists with `metadata.source == "billing"` AND at least one resource has `config_confidence == "assumed"`.

**This category is not yet active.** It will fire when billing discovery (`discover-billing.md`) is implemented in v1.2. When active, it fills factual gaps in billing-inferred inventories:

- **Cloud SQL HA**: Single-zone or high-availability?
- **Cloud Run service count**: How many distinct services?
- **Memorystore memory size**: How much memory (GB)?
- **Cloud Functions generation**: Gen 1 or Gen 2?

Category B answers are recorded in `metadata.inventory_clarifications` and update the inventory understanding — they do not produce design constraints directly.

---

## Category C — Compute Model (If Compute Resources Present)

_Fire when:_ Compute resources present (Cloud Run, Cloud Functions, GKE, GCE).

---

**Q7 — How does your team feel about managing Kubernetes?**

_Fire when:_ GKE cluster present AND Q4 != A (multi-cloud). Skip when: Q4 = A (already resolved to EKS) or no GKE in inventory.

> Your team's Kubernetes experience determines whether we recommend EKS (Kubernetes on AWS) or ECS Fargate (simpler managed containers).
>
> A) Love it / Team is K8s expert
> B) Neutral / Competent with K8s
> C) Frustrated / Learning curve steep
> D) N/A — We don't use Kubernetes
> E) I don't know

Interpret:

```
A -> kubernetes: "eks-managed" — EKS recommended, preserves K8s investment
B -> kubernetes: "eks-or-ecs" — EKS with managed node groups to reduce operational burden
C -> kubernetes: "ecs-fargate" — Strong ECS Fargate recommendation, eliminates K8s management
D -> (no constraint written — no K8s workloads)
E -> same as default (B) — assume neutral, evaluate both EKS and ECS
```

Default: B — `kubernetes: "eks-or-ecs"`.

---

**Q8 — Do any of your services need WebSocket support or long-lived connections?**

_Fire when:_ Compute resources present AND WebSocket usage cannot be determined from inventory.

> WebSocket support affects load balancer configuration. This confirms whether ALB WebSocket configuration is needed in the migration templates.
>
> A) Yes — Real-time features, WebSockets, persistent connections
> B) No — Standard HTTP/HTTPS only
> C) I don't know

Interpret:

```
A -> websocket: "required" — ALB with WebSocket support, ECS Fargate or EKS required
B -> (no constraint written)
C -> same as default (B) — assume no WebSocket; can be reconfigured later
```

Default: B — no constraint.

---

**Q9 — What's your typical traffic pattern for your Cloud Run services?**

_Fire when:_ Cloud Run present in inventory. Skip when: no Cloud Run.

> Cloud Run's scale-to-zero is its primary cost advantage. Understanding your traffic pattern helps me determine whether migrating Cloud Run to AWS makes financial sense.
>
> A) Business hours only (9am-5pm weekdays, ~40 hrs/week)
> B) Active most of the day (16-20 hours, ~120 hrs/week)
> C) Constant 24/7 traffic (~168 hrs/week)
> D) N/A — We don't use Cloud Run
> E) I don't know

Interpret:

```
A -> cloud_run_traffic_pattern: "business-hours" — AWS likely 40-50% MORE expensive; flag cost increase
B -> cloud_run_traffic_pattern: "most-of-day" — Moderate cost difference; present both options
C -> cloud_run_traffic_pattern: "constant-24-7" — AWS costs similar or cheaper; ECS Fargate recommended
D -> (no constraint written — Cloud Run not used)
E -> same as default (C) — assume constant traffic for conservative estimate
```

Default: C — `cloud_run_traffic_pattern: "constant-24-7"`.

---

**Q10 — Approximately how much are you spending on Cloud Run per month?**

_Fire when:_ Cloud Run present in inventory. Skip when: no Cloud Run.

> Absolute Cloud Run spend determines whether the migration math makes financial sense regardless of traffic pattern.
>
> A) < $100/month
> B) $100-$500/month
> C) $500-$1,500/month
> D) > $1,500/month
> E) N/A — We don't use Cloud Run
> F) I don't know

Interpret:

```
A -> cloud_run_monthly_spend: "<$100" — Recommend staying on Cloud Run; migration cost exceeds savings
B -> cloud_run_monthly_spend: "$100-$500" — Present cost comparison; migration may make sense if consolidating
C -> cloud_run_monthly_spend: "$500-$1500" — Fixed-cost AWS options attractive (ECS Fargate reserved)
D -> cloud_run_monthly_spend: ">$1500" — Strong case for ECS Fargate with Savings Plans
E -> (no constraint written)
F -> same as default (B)
```

Default: B — `cloud_run_monthly_spend: "$100-$500"`.

---

## Category D — Database Model (If Database Resources Present)

_Fire when:_ Database resources present (Cloud SQL, Spanner, Memorystore).

---

**Q11 — Are you experiencing scale limitations with Cloud SQL?**

_Fire when:_ Cloud SQL present in inventory. Skip when: no Cloud SQL.

> Scale limitations indicate whether standard Aurora is sufficient or whether more specialized options are needed.
>
> A) No — Current scale is acceptable (< 10K writes/sec, < 50K reads/sec)
> B) Yes — Hitting hard scale limits (> 100K writes/sec, global <10ms latency)
> C) N/A — We don't use Cloud SQL
> D) I don't know

Interpret:

```
A -> database_tier: "standard" — Standard Aurora Multi-AZ
B -> database_tier: "aurora-scale" — Aurora DSQL considered for global active-active; architecture review flagged
C -> (no constraint written)
D -> same as default (A) — assume standard scale
```

Default: A — `database_tier: "standard"`.

---

**Q12 — What's your typical database I/O workload?**

_Fire when:_ Cloud SQL present in inventory. Skip when: no Cloud SQL.

> Aurora has two pricing modes — standard and I/O-Optimized. Choosing wrong can mean paying 40% more than necessary.
>
> A) Low (< 1,000 IOPS) — Mostly reads, infrequent writes
> B) Medium (1,000-10,000 IOPS) — Balanced workload
> C) High (> 10,000 IOPS) — Write-heavy, high transactions
> D) N/A — We don't use Cloud SQL
> E) I don't know

Interpret:

```
A -> db_io_workload: "low" — Aurora standard pricing
B -> db_io_workload: "medium" — Aurora standard; flag I/O-Optimized as option if workload grows
C -> db_io_workload: "high" — Aurora I/O-Optimized recommended (up to 40% savings at high I/O)
D -> (no constraint written)
E -> same as default (B) — assume medium I/O
```

Default: B — `db_io_workload: "medium"`.

---

## Category E — Migration Posture (Disabled by Default)

_Fire when:_ User explicitly opts in.
_Default behavior when disabled:_ Apply conservative defaults — no HA upgrades, no right-sizing.

If the user opts in, present after all other categories:

- **HA upgrade preference**: Should we recommend upgrading Single-AZ to Multi-AZ where possible?
  > Default: No — keep current topology.
- **Right-sizing from billing**: Should we use billing utilization data to right-size instance types?
  > Default: No — match current capacity.

---

## Category F — AI/Bedrock (v1.1 Stub)

_Fire when:_ `ai-workload-profile.json` exists in `$MIGRATION_DIR/`.

**This category is not yet active.** It will fire when AI discovery (`discover-app-code.md`) is implemented in v1.1. When active, it asks up to 5 questions (Q13-Q17) covering:

- AI migration priority (speed vs cost vs quality)
- Token volume estimates
- Latency tolerance
- Model type/provider preference
- Additional capability requirements beyond detected

Category F answers are written to `ai_constraints` in `preferences.json`.

---

## Step 4: Present Questions

Show all generated questions at once, grouped by section. Present Category A first, then C, then D, then B (if applicable), then F (if applicable), then E (if opted in).

```
Before mapping your infrastructure to AWS, I have some questions to tailor the migration plan.
You can answer each, skip individual ones (I'll use sensible defaults),
or say "use all defaults" to proceed with all recommendations.

--- Section 1: About Your Users & Requirements ---

Question 1 of [M]: [Q1 text with context]
Question 2 of [M]: [Q2 text with context]
...

--- Section 2: Your Infrastructure ---
[Only if Categories C/D fire]

Question [N] of [M]: [Q7 text with context]
...
```

Wait for the user's response. Do NOT proceed to Design without a response or an explicit "use all defaults".

---

## Answer Combination Triggers

| Scenario                | Key Answers                      | Recommendation                        |
| ----------------------- | -------------------------------- | ------------------------------------- |
| Must stay portable      | Q4 = Yes multi-cloud             | EKS only, no ECS Fargate              |
| Kubernetes-averse       | Q4 = No + Q7 = Frustrated        | ECS Fargate strongly recommended      |
| WebSocket app           | Q8 = Yes                         | ALB WebSocket config required         |
| Low-traffic Cloud Run   | Q9 = Business hours + Q10 < $100 | Recommend staying on Cloud Run        |
| High I/O database       | Q12 = High IOPS                  | Aurora I/O-Optimized                  |
| Global active-active DB | Q5 = Catastrophic + Q1 = Global  | Aurora Global Database + multi-region |
| Zero downtime required  | Q6 = No downtime                 | Blue/green + AWS DMS required         |
| HIPAA compliance        | Q2 = HIPAA                       | BAA services only, specific regions   |
| FedRAMP required        | Q2 = FedRAMP                     | GovCloud regions only                 |

---

## Step 5: Interpret and Write preferences.json

Apply the interpret rule for every answered question. For skipped questions, apply the documented default. Write `$MIGRATION_DIR/preferences.json`:

```json
{
  "metadata": {
    "timestamp": "<ISO timestamp>",
    "discovery_artifacts": ["gcp-resource-inventory.json"],
    "questions_asked": ["Q1", "Q2", "Q3", "Q4", "Q5", "Q6", "Q7", "Q11", "Q12"],
    "questions_defaulted": ["Q8"],
    "questions_skipped_extracted": [],
    "questions_skipped_early_exit": [],
    "questions_skipped_not_applicable": ["Q9", "Q10"],
    "category_e_enabled": false,
    "inventory_clarifications": {}
  },
  "design_constraints": {
    "target_region": { "value": "us-east-1", "chosen_by": "user" },
    "compliance": { "value": ["hipaa"], "chosen_by": "user" },
    "gcp_monthly_spend": { "value": "$5K-$20K", "chosen_by": "user" },
    "availability": { "value": "multi-az", "chosen_by": "default" },
    "cutover_strategy": { "value": "maintenance-window-weekly", "chosen_by": "user" },
    "kubernetes": { "value": "eks-or-ecs", "chosen_by": "user" },
    "database_tier": { "value": "standard", "chosen_by": "user" },
    "db_io_workload": { "value": "medium", "chosen_by": "user" }
  }
}
```

### Schema Rules

1. Every entry in `design_constraints` is an object with `value` and `chosen_by` fields.
2. `chosen_by` values: `"user"` (explicitly answered), `"default"` (system default applied — includes "I don't know" answers), `"extracted"` (inferred from inventory), `"derived"` (computed from combination of answers + detected capabilities).
3. Only write a key to `design_constraints` if the answer produces a constraint. Absent keys mean "no constraint — Design decides."
4. Do not write null values.
5. For billing-source inventories (v1.2), `metadata.inventory_clarifications` records Category B answers.
6. `metadata.questions_skipped_early_exit` records questions skipped due to early-exit logic (e.g., Q7 skipped because Q4=multi-cloud).
7. `metadata.questions_skipped_extracted` records questions skipped because inventory already provided the answer.
8. `metadata.questions_skipped_not_applicable` records questions skipped because the relevant service wasn't in the inventory.
9. `ai_constraints` section is present ONLY if Category F fired. Omit entirely if no AI artifacts exist. _(v1.1: currently always omitted)_

---

## Defaults Table

| Question               | Default           | Constraint                                        |
| ---------------------- | ----------------- | ------------------------------------------------- |
| Q1 — Location          | A (single region) | `target_region`: closest AWS region to GCP region |
| Q2 — Compliance        | A (none)          | no constraint                                     |
| Q3 — GCP spend         | B ($1K-$5K)       | `gcp_monthly_spend: "$1K-$5K"`                    |
| Q4 — Multi-cloud       | B (AWS-only)      | no constraint                                     |
| Q5 — Uptime            | B (significant)   | `availability: "multi-az"`                        |
| Q6 — Maintenance       | D (flexible)      | `cutover_strategy: "flexible"`                    |
| Q7 — K8s sentiment     | B (neutral)       | `kubernetes: "eks-or-ecs"`                        |
| Q8 — WebSocket         | B (no)            | no constraint                                     |
| Q9 — Cloud Run traffic | C (24/7)          | `cloud_run_traffic_pattern: "constant-24-7"`      |
| Q10 — Cloud Run spend  | B ($100-$500)     | `cloud_run_monthly_spend: "$100-$500"`            |
| Q11 — DB scale         | A (no limits)     | `database_tier: "standard"`                       |
| Q12 — DB I/O           | B (medium)        | `db_io_workload: "medium"`                        |

---

## Validation Checklist

Before handing off to Design:

- [ ] `preferences.json` written to `$MIGRATION_DIR/`
- [ ] `design_constraints.target_region` is populated with `value` and `chosen_by`
- [ ] `design_constraints.availability` is populated (if Q5 was asked or defaulted)
- [ ] Only keys with non-null values are present in `design_constraints`
- [ ] Every entry in `design_constraints` has `value` and `chosen_by` fields
- [ ] Config gap answers recorded in `metadata.inventory_clarifications` (billing mode only)
- [ ] Early-exit skips recorded in `metadata.questions_skipped_early_exit`
- [ ] `ai_constraints` section present ONLY if Category F fired _(v1.1: currently always omitted)_
- [ ] Output is valid JSON

---

## Step 6: Update Phase Status

Update `$MIGRATION_DIR/.phase-status.json`:

- Set `phases.clarify.status` to `"completed"`
- Set `phases.clarify.timestamp` to current ISO 8601 timestamp
- Set `phases.clarify.outputs` to `["preferences.json"]`
- Update `last_updated` to current timestamp

Output to user: "Clarification complete. Proceeding to Phase 3: Design AWS Architecture."

---

## Scope Boundary

**This phase covers requirements gathering ONLY.**

FORBIDDEN — Do NOT include ANY of:

- Detailed AWS architecture or service configurations
- Code migration examples or SDK snippets
- Detailed cost calculations
- Migration timelines or execution plans
- Terraform generation

**Your ONLY job: Understand what the user needs. Nothing else.**
