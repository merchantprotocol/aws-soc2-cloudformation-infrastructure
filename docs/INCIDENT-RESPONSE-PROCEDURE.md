# Incident Response Procedure

## SOC 2 Criteria: CC7.3, CC7.4, CC7.5

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

This procedure defines how the Merchant Protocol team detects, responds to, communicates about, and learns from security incidents affecting the AWS infrastructure provisioned by our CloudFormation templates.

---

## 1. Severity Levels

| Severity | Definition | Response Time | Examples |
|----------|-----------|---------------|----------|
| **SEV-1 Critical** | Active data breach, complete service outage, or active exploitation | Immediate (within 15 minutes) | Unauthorized database access, ransomware, full ALB/app outage |
| **SEV-2 High** | Potential data exposure, partial outage, or suspicious activity | Within 1 hour | Unauthorized API calls detected, single-AZ failure, security group modified unexpectedly |
| **SEV-3 Medium** | Policy violation, degraded performance, or failed security control | Within 4 hours | CloudTrail stopped, expired SSL certificate, elevated error rates |
| **SEV-4 Low** | Minor policy deviation, informational alert | Within 1 business day | Failed login attempts below threshold, non-critical alarm |

---

## 2. Detection Sources

These are the monitoring systems deployed by the [SOC 2 template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) that may trigger incident detection:

| Source | What It Detects | Template Resource | Where to Check |
|--------|----------------|------------------|---------------|
| CloudWatch Alarms | CPU spikes, unauthorized API calls | [`HighCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`DatabaseCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`UnauthorizedAPICallsAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | CloudWatch console > Alarms |
| VPC Flow Logs | Unusual network traffic, rejected connections, data exfiltration patterns | [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520) -> [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) | CloudWatch console > Log groups > `/vpc/flowlogs/` |
| CloudTrail | API calls — who did what, when | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) -> [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) | CloudTrail console or S3 bucket |
| ALB Access Logs | Web request patterns, error rates | [`WebAppALB`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L891) | S3 bucket (if enabled) or CloudWatch |
| AWS GuardDuty | Threat intelligence-based detection | Not in template (enable separately) | GuardDuty console |
| Manual report | User or team member reports an issue | N/A | Incident channel / email |

---

## 3. Response Procedure

### 3.1 Phase 1 — Triage (First 30 Minutes)

1. **Acknowledge** the incident and assign an incident commander
2. **Assess severity** using the table in Section 1
3. **Contain immediately** if active threat:
   - Isolate compromised instance by replacing its security group with a quarantine SG (no ingress, no egress)
   - Do NOT terminate the instance — preserve for forensics
   - If database compromise suspected: rotate master credentials on [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) immediately
   - If SSH key compromise suspected: remove key from bastion authorized_keys and update [`BastionAllowedCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L154) to block attacker CIDR
4. **Open incident channel** for communication (Slack channel, war room, etc.)
5. **Notify** per escalation matrix (Section 4)

### 3.2 Phase 2 — Investigation (Hours 1-4)

1. **Collect evidence** — do NOT modify or delete logs:
   - CloudTrail events from [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580): `aws cloudtrail lookup-events --start-time <time>`
   - VPC Flow Logs from [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490)
   - CloudFormation stack events: `aws cloudformation describe-stack-events`
   - Instance system logs if accessible
2. **Determine scope**:
   - Which resources were affected?
   - Was data accessed or exfiltrated?
   - How did the attacker gain access?
   - Are other resources at risk?
3. **Document findings** in the incident log

### 3.3 Phase 3 — Remediation

1. **Eradicate** the threat:
   - Patch vulnerability or close unauthorized access path
   - Rotate all potentially compromised credentials
   - Update security groups ([`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`EC2PrivateAPPServers`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L711), [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740)), IAM policies, or KMS keys as needed
2. **Recover** services:
   - Restore database from backup if needed — [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) has 35-day PITR and `DeletionPolicy: Snapshot`
   - Restore EFS from backup if needed — [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) has backups enabled
   - Deploy clean instances via CloudFormation stack update using [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987)
   - Verify application health checks pass on [`WebAppTargetGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L913)
3. **Verify** controls are restored:
   - Run through relevant sections of [`SOC2-CONTROL-MATRIX.md`](SOC2-CONTROL-MATRIX.md)
   - Confirm [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648), [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520), and alarms are active
   - Confirm encryption is intact on [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)

### 3.4 Phase 4 — Post-Incident Review (Within 5 Business Days)

1. **Conduct review meeting** with all responders
2. **Document**:
   - Timeline of events
   - Root cause
   - What worked well
   - What could be improved
   - Action items with owners and deadlines
3. **Update procedures** if gaps were identified
4. **Update CloudFormation template** if infrastructure changes are needed (follow [`CHANGE-MANAGEMENT-PROCEDURE.md`](CHANGE-MANAGEMENT-PROCEDURE.md))

---

## 4. Escalation Matrix

| Severity | Notify | Within |
|----------|--------|--------|
| SEV-1 | Infrastructure admin, CTO, legal counsel | 15 minutes |
| SEV-2 | Infrastructure admin, engineering lead | 1 hour |
| SEV-3 | Infrastructure admin | 4 hours |
| SEV-4 | Infrastructure admin (next standup) | 1 business day |

For data breaches involving customer PII: legal counsel and compliance officer must be notified regardless of severity.

---

## 5. Evidence Preservation

**Critical**: Do not destroy evidence during incident response.

- **Do not terminate** compromised EC2 instances — stop them or isolate with security groups
- **Do not delete** CloudTrail logs, Flow Logs, or CloudWatch log groups
- **Do not rotate** KMS keys (this would prevent decrypting historical logs)
- **Do** take EBS snapshots of compromised instances before any changes
- **Do** export relevant CloudWatch log groups to S3 for long-term preservation
- **Do** screenshot any console findings

CloudTrail logs are protected by the template's built-in controls:
- [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) has S3 versioning enabled (deletions create delete markers, not permanent deletes)
- [`CloudTrailBucketPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L610) denies unencrypted transport
- [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) has `EnableLogFileValidation: true` (tampering is detectable)
- [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) lifecycle archives to Glacier after 90 days (immutable for 7 years)

---

## 6. Useful Investigation Commands

```bash
# Recent CloudTrail events for a specific user
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=<username> \
  --start-time <ISO8601> --end-time <ISO8601>

# CloudTrail events for a specific resource
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<resource-arn>

# Query VPC Flow Logs in CloudWatch Insights
# (run against the VPCFlowLogGroup log group)
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| sort @timestamp desc
| limit 100

# Check current security group rules
aws ec2 describe-security-groups --group-ids <sg-id>

# Check CloudFormation stack for unauthorized changes
aws cloudformation detect-stack-drift --stack-name <stack-name>
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>
```

---

## 7. Incident Log Template

| Field | Value |
|-------|-------|
| Incident ID | INC-YYYY-NNN |
| Date/Time Detected | |
| Detected By | |
| Severity | |
| Incident Commander | |
| Description | |
| Affected Resources | |
| Root Cause | |
| Data Impact | |
| Timeline | |
| Remediation Actions | |
| Lessons Learned | |
| Action Items | |
| Closed Date | |

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — Verify controls are restored post-incident
- [Monitoring and Alerting Procedure](MONITORING-AND-ALERTING-PROCEDURE.md) — Detection source details
- [Backup and Recovery Procedure](BACKUP-AND-RECOVERY-PROCEDURE.md) — Data restoration steps
- [Change Management Procedure](CHANGE-MANAGEMENT-PROCEDURE.md) — Emergency change process
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — Where to find audit evidence

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial procedure created |
