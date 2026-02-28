# Phase 5: Execution Plan

## Step 1: Validate Design and Estimation

Read `aws-design.json` and `estimation.json`. If either missing or incomplete: **STOP**. Output: "Prior phases incomplete. Complete Phase 3 (Design) and Phase 4 (Estimation) first."

## Step 2: Build Execution Timeline

Create 8-12 week timeline with critical path based on cluster dependencies and data transfer complexity.

### Week 1-2: Planning & Setup

- Finalize AWS account structure
- Set up network (VPC, subnets, routing)
- Provision core IAM roles
- Validate connectivity (GCP to AWS for data migration)

### Week 3-5: Proof of Concept

- Deploy smallest cluster to AWS
- Test application performance
- Validate data pipeline (GCP → AWS)
- Measure baseline latency

### Week 6-8: Full Infrastructure

- Deploy remaining clusters
- Set up cross-cluster networking
- Implement monitoring and logging
- Establish backup/restore procedures

### Week 9-10: Data Migration

- Migrate primary data (databases, storage)
- Validate data integrity
- Establish replication / dual-write for production cutover

### Week 11: Cutover

- Test failover procedures
- DNS switch (GCP → AWS)
- Monitor for 24-48 hours
- Rollback procedures on standby

### Week 12: Cleanup

- Decommission GCP resources
- Archive GCP data
- Final cost reconciliation

## Step 3: Risk Assessment

**Critical risks:**

- **Data loss during migration**: Mitigation: dual-write for 2 weeks before cutover; full backup before migration
- **Performance regression**: Mitigation: PoC testing in Week 3-5; load testing in Week 6
- **Team capacity**: Mitigation: assume 2 FTE engineers dedicated for 12 weeks; external support if needed
- **Rollback complexity**: Mitigation: practice rollback procedures in Week 9; maintain read-only GCP copy for 2 weeks post-cutover

## Step 4: Rollback Procedures

**Trigger conditions for rollback:**

- Data integrity issues detected during validation
- Performance regression >20% vs GCP baseline
- Cost overruns >50% vs estimation
- Critical unforeseen AWS service limitations

**Rollback steps (reversible up to DNS cutover):**

1. Pause dual-write replication
2. Reverse DNS records (AWS → GCP)
3. Shut down AWS workloads (keep for 1 week as standby)
4. Resume GCP read traffic
5. Monitor for 24 hours

Post-DNS, rollback is manual: requires restore from backup (2-4 hour RTO).

## Step 5: GCP Teardown Checklist

Only after 2 weeks stable AWS operation:

- [ ] Archive all GCP data to Cloud Storage (long-term retention)
- [ ] Delete GCP compute instances, databases, storage buckets
- [ ] Delete GCP VPC and networking
- [ ] Disable GCP billing
- [ ] Archive project for audit trail

## Step 6: Write Execution Output

Write `execution.json`:

```json
{
  "timeline_weeks": 12,
  "critical_path": [
    "VPC setup (Week 1)",
    "PoC deployment (Week 3-5)",
    "Data migration (Week 9-10)",
    "DNS cutover (Week 11)"
  ],
  "risks": [
    {
      "category": "data_loss",
      "probability": "low",
      "impact": "critical",
      "mitigation": "dual-write + backup"
    },
    {
      "category": "performance_regression",
      "probability": "medium",
      "impact": "high",
      "mitigation": "PoC testing (Week 3-5); load testing (Week 6)"
    }
  ],
  "rollback_window": "Reversible until DNS cutover (Week 11)",
  "gcp_teardown": "Week 14 (2 weeks after cutover)"
}
```

Write `execution-timeline.md`:

```
# GCP→AWS Migration Timeline

## Week 1-2: Planning & Setup
- [ ] AWS account setup
- [ ] VPC / Subnets / Routing
- [ ] IAM roles & policies
- [ ] GCP→AWS connectivity test

## Week 3-5: Proof of Concept
- [ ] Deploy pilot cluster
- [ ] Latency & performance baseline
- [ ] Data pipeline validation
- [ ] Sign-off on architecture

## Week 6-8: Full Infrastructure
- [ ] Deploy all clusters
- [ ] Cross-cluster networking
- [ ] Monitoring / logging setup
- [ ] Backup procedures

## Week 9-10: Data Migration
- [ ] Primary data migration
- [ ] Data integrity validation
- [ ] Establish dual-write

## Week 11: Cutover
- [ ] Failover test
- [ ] DNS switch
- [ ] 24-48hr monitoring

## Week 12: Cleanup
- [ ] Cost reconciliation
- [ ] Final validation

## Week 14: GCP Teardown
- [ ] Archive data
- [ ] Delete resources
- [ ] Close project
```

## Step 7: Update Phase Status

Update `$MIGRATION_DIR/.phase-status.json`:
- Set `phases.execute.status` to `"completed"`
- Set `phases.execute.timestamp` to current ISO 8601 timestamp
- Set `phases.execute.outputs` to `["execution.json", "execution-timeline.md"]`
- Update `last_updated` to current timestamp

Output to user:

"✓ Migration plan complete. Summary:

- Timeline: 12 weeks
- AWS monthly cost: $3,500 (Balanced)
- Payback period: 32.5 months
- Rollback window: Through DNS cutover

Files saved:

- aws-design.json
- estimation.json
- execution.json

Use this plan to guide your migration. All phases of the GCP-to-AWS migration analysis are complete."
