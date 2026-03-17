# Change Management Procedure

## SOC 2 Criteria: CC8.1

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

This procedure governs how changes to the Merchant Protocol AWS infrastructure are proposed, approved, tested, deployed, and documented. All infrastructure is defined as code in CloudFormation templates and managed through version control.

---

## 1. Change Classification

| Classification | Description | Approval Required | Examples |
|---------------|-------------|-------------------|----------|
| **Standard** | Pre-approved, low-risk, routine | None (pre-approved) | Scaling instance count within existing ASG limits, updating AMI to latest |
| **Normal** | Planned change with known impact | Infrastructure admin | New security group rule, parameter change, new CloudWatch alarm |
| **Significant** | Architectural change, new services, security-impacting | Infrastructure admin + management | New subnet, new service (e.g., adding Redis), changing encryption settings |
| **Emergency** | Unplanned change to restore service | Infrastructure admin (retroactive approval within 24h) | Hotfix for outage, security patch for active vulnerability |

---

## 2. Change Request Process

### 2.1 Proposal

1. Create a pull request in the `aws-cloudformation-templates` repository
2. PR description must include:
   - **What**: Description of the change
   - **Why**: Business justification
   - **Impact**: What resources are affected (reference specific template resources, e.g., [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956))
   - **Risk**: What could go wrong
   - **Rollback plan**: How to revert if the change fails
   - **Classification**: Standard / Normal / Significant / Emergency

### 2.2 Review

1. At least one peer review required for all changes
2. Significant changes require a second reviewer
3. Reviewers verify:
   - [ ] Change matches the stated purpose
   - [ ] No security regressions (security groups, encryption, IAM)
   - [ ] No SOC 2 control violations (check against [`SOC2-CONTROL-MATRIX.md`](SOC2-CONTROL-MATRIX.md))
   - [ ] Rollback plan is viable
   - [ ] Template validates (`aws cloudformation validate-template`)

### 2.3 Testing

1. Deploy change to a staging/test stack first when possible
2. Verify:
   - [ ] Stack creates/updates without errors
   - [ ] Application health checks pass
   - [ ] No unintended resource changes (review CloudFormation changeset)
3. For database changes: test with a snapshot restore, not production

### 2.4 Approval

1. All reviewers approve the PR
2. For Significant changes: management sign-off recorded in PR comments
3. Merge PR to main branch

### 2.5 Deployment

1. Create a CloudFormation changeset in production: `aws cloudformation create-change-set`
2. Review the changeset output — verify only expected resources are modified
3. Execute the changeset: `aws cloudformation execute-change-set`
4. Monitor stack events for completion
5. Verify application health post-deployment
6. Document deployment completion in the change log

See [`DEPLOYMENT-GUIDE.md`](DEPLOYMENT-GUIDE.md) for detailed deployment commands.

### 2.6 Emergency Changes

1. Change may be deployed before PR approval
2. PR must still be created and approved within 24 hours
3. Post-incident review required within 5 business days (see [`INCIDENT-RESPONSE-PROCEDURE.md`](INCIDENT-RESPONSE-PROCEDURE.md))
4. Document in change log with "Emergency" classification

---

## 3. Rollback Procedure

### 3.1 CloudFormation Stack Rollback

If a stack update fails:
1. CloudFormation automatically rolls back to previous state
2. Review stack events to identify failure cause
3. Fix the template and re-attempt

### 3.2 Manual Rollback

If the update succeeds but causes issues:
1. Revert the PR in the repository
2. Deploy the reverted template via changeset
3. For database changes: restore from snapshot if needed — [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) has 35-day backup retention and `DeletionPolicy: Snapshot`

### 3.3 Rollback NOT Possible

If rollback would cause data loss:
1. Escalate to infrastructure admin immediately
2. Assess forward-fix options
3. Document decision and justification

---

## 4. Change Log Template

Maintain for auditor review. This supplements git history with business context:

| Date | Change ID (PR #) | Classification | Description | Deployed By | Rollback Plan | Result |
|------|----------|------|------|------|------|------|
| | | | | | | |

---

## 5. Prohibited Changes

The following changes require explicit management approval and SOC 2 impact assessment before proceeding. Each references the specific template resource that would be affected:

| Prohibited Change | Affected Resource | Why |
|-------------------|------------------|-----|
| Removing encryption from any resource | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027), [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) | CC6.1 — encryption at rest required |
| Opening security groups to 0.0.0.0/0 for non-web ports | [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740), [`EFSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L764) | CC6.6 — network access must be restricted |
| Disabling CloudTrail or VPC Flow Logs | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648), [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520) | CC7.1 — monitoring must be continuous |
| Changing `DeletionPolicy` from `Snapshot` to `Delete` | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | A1.2 — data protection required |
| Disabling `DeletionProtection` on Aurora | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | A1.2 — accidental deletion prevention |
| Removing or reducing backup retention periods | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (35-day), [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) | A1.2, CC7.3 — retention requirements |
| Granting `kms:*` wildcard permissions | [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) | CC6.3 — least privilege |
| Making database publicly accessible | [`AuroraDBInstance`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1088) | CC6.1, CC6.6 — private access only |
| Removing TLS enforcement | [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019), [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929) | CC6.7 — encryption in transit required |

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — Full mapping of controls to TSC criteria
- [Deployment Guide](DEPLOYMENT-GUIDE.md) — Detailed deployment and update commands
- [Incident Response Procedure](INCIDENT-RESPONSE-PROCEDURE.md) — For emergency change follow-up
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — Drift detection and change evidence

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial procedure created |
