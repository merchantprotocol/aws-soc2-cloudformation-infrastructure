# Evidence Collection Guide

## SOC 2 Type II Audit — AWS Infrastructure Evidence

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

This guide lists every piece of evidence an auditor will request for the infrastructure controls in our SOC 2 Type II audit. For each item, it tells you exactly where to find it and which template resource implements the control. Run through this checklist before the audit period begins and again before the auditor arrives.

---

## How SOC 2 Type II Works

- **Type I** = controls are designed correctly (point-in-time)
- **Type II** = controls are designed correctly AND operated effectively over a period (usually 6-12 months)

This means the auditor will ask for evidence that controls were **continuously active** during the audit period, not just that they exist today.

---

## 1. Network Security (CC6.1, CC6.6)

### 1.1 VPC Configuration

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| VPC exists with correct CIDR | [`MPVPC`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L299) | `aws ec2 describe-vpcs --filters "Name=tag:Name,Values=MPWebVPC"` |
| Public/private subnet separation | [`MPPrivateSubnet1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L451), [`MPPrivateSubnet2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L464) | `aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>"` — verify private subnets have `MapPublicIpOnLaunch: false` |
| Route tables correctly configured | [`MPPrivateRouteTable1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L419), [`MPPrivateRouteTable2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L427) | `aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"` — verify private subnets route through NAT, not IGW |

### 1.2 Security Groups

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| No 0.0.0.0/0 on non-web ports | All SGs (L667–L782) | `aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>" --query "SecurityGroups[*].{Name:GroupName,Rules:IpPermissions}"` |
| Bastion SSH restricted to allowed CIDR | [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | Check SG — verify SSH ingress uses [`BastionAllowedCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L154), NOT 0.0.0.0/0 |
| Database only accessible from app servers | [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740) | Check SG — verify ingress references [`EC2PrivateAPPServers`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L711) SG only |
| No RDSPublic security group exists | Removed from template | Verify no SG with description "Public Access To RDS" exists in the VPC |

### 1.3 Screenshot Evidence

Take screenshots of:
- [ ] VPC console showing all subnets and their route table associations
- [ ] Each security group's inbound and outbound rules
- [ ] NAT Gateway configuration in each AZ

---

## 2. Encryption (CC6.1, CC6.7)

### 2.1 Encryption at Rest

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| Aurora storage encrypted | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`StorageEncrypted: true`), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055) | `aws rds describe-db-clusters --query "DBClusters[*].{Cluster:DBClusterIdentifier,Encrypted:StorageEncrypted,KmsKey:KmsKeyId}"` |
| EFS encrypted | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) (`Encrypted: true`), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148) | `aws efs describe-file-systems --query "FileSystems[*].{Id:FileSystemId,Encrypted:Encrypted,KmsKey:KmsKeyId}"` |
| CloudTrail S3 bucket encrypted | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) | `aws s3api get-bucket-encryption --bucket <bucket-name>` |
| KMS key rotation enabled | [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) | `aws kms get-key-rotation-status --key-id <key-id>` — run for each key |

### 2.2 Encryption in Transit

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| ALB uses TLS 1.3 | [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929) (`ELBSecurityPolicy-TLS13-1-2-2021-06`) | `aws elbv2 describe-listeners --load-balancer-arn <arn> --query "Listeners[*].{Port:Port,Protocol:Protocol,SslPolicy:SslPolicy}"` |
| HTTP redirects to HTTPS | [`ALBHTTPRedirectListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L942) | `aws elbv2 describe-listeners` — verify port 80 listener has redirect action |
| Aurora requires TLS | [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019) (`require_secure_transport: ON`) | `aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <name> --query "Parameters[?ParameterName=='require_secure_transport']"` |
| EFS denies unencrypted transport | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) > `FileSystemPolicy` | `aws efs describe-file-system-policy --file-system-id <id>` — verify deny on `SecureTransport: false` |
| CloudTrail bucket denies HTTP | [`CloudTrailBucketPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L610) | `aws s3api get-bucket-policy --bucket <bucket-name>` — verify deny on `SecureTransport: false` |

---

## 3. Logging and Monitoring (CC7.1, CC7.2, CC7.3)

### 3.1 CloudTrail

| Evidence | Template Resource | How to Collect | Audit Period Check |
|----------|------------------|---------------|-------------------|
| Trail is active | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) | `aws cloudtrail get-trail-status --name <trail-name>` — verify `IsLogging: true` | Must be true for entire audit period |
| Log file validation enabled | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) (`EnableLogFileValidation: true`) | `aws cloudtrail describe-trails --query "trailList[*].{Name:Name,Validation:LogFileValidationEnabled}"` | |
| Logs exist for audit period | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) | `aws s3 ls s3://<bucket>/AWSLogs/<account-id>/CloudTrail/ --recursive` | Check for log files spanning the audit period |
| Log file integrity | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) | `aws cloudtrail validate-logs --trail-arn <arn> --start-time <start> --end-time <end>` | Run for full audit period |

### 3.2 VPC Flow Logs

| Evidence | Template Resource | How to Collect | Audit Period Check |
|----------|------------------|---------------|-------------------|
| Flow log is active | [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520) | `aws ec2 describe-flow-logs --filter "Name=resource-id,Values=<vpc-id>"` — verify `FlowLogStatus: ACTIVE` | Must be active for entire audit period |
| Log group exists with retention | [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) | `aws logs describe-log-groups --log-group-name-prefix /vpc/flowlogs/` | Verify retention >= 365 days (template default) |
| Logs exist for audit period | [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) | CloudWatch Logs Insights query across date range | Query for first and last entries in audit period |

### 3.3 CloudWatch Alarms

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| Alarms exist and are active | [`HighCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`DatabaseCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`UnauthorizedAPICallsAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | `aws cloudwatch describe-alarms --alarm-name-prefix <stack-name>` — verify `StateValue` and `ActionsEnabled` |
| Alarm history for audit period | Same resources | `aws cloudwatch describe-alarm-history --alarm-name <name> --start-date <start> --end-date <end>` |

---

## 4. Access Controls (CC6.1, CC6.3)

### 4.1 IAM

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| Instance roles exist (no stored credentials) | [`BastionInstanceProfile`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L801), [`WebAppInstanceProfile`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L845) | `aws iam list-instance-profiles` — verify bastion and webapp profiles exist |
| Role policies are scoped | [`BastionInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L784), [`WebAppInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L807) | `aws iam list-attached-role-policies --role-name <role>` and `aws iam list-role-policies --role-name <role>` |
| No wildcard IAM policies | Same roles | Review policy documents for `"Action": "*"` or `"Resource": "*"` |
| IMDSv2 enforced | [`BastionLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L871), [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987) | `aws ec2 describe-launch-template-versions --launch-template-name <name> --query "LaunchTemplateVersions[*].LaunchTemplateData.MetadataOptions"` |

### 4.2 Access Reviews

| Evidence | Where to Find |
|----------|--------------|
| Quarterly access review records | Access register (see [`ACCESS-MANAGEMENT-PROCEDURE.md`](ACCESS-MANAGEMENT-PROCEDURE.md) Section 6) |
| User provisioning/deprovisioning records | Access register |
| MFA enforcement | IAM Identity Center console or `aws iam get-account-summary` |

---

## 5. Data Protection (A1.2)

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| Aurora deletion protection enabled | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`DeletionProtection: true`) | `aws rds describe-db-clusters --query "DBClusters[*].{Cluster:DBClusterIdentifier,DeletionProtection:DeletionProtection}"` |
| Aurora backup retention = 35 days | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`BackupRetentionPeriod: 35`) | `aws rds describe-db-clusters --query "DBClusters[*].{Cluster:DBClusterIdentifier,Retention:BackupRetentionPeriod}"` |
| Aurora automated backups exist | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | `aws rds describe-db-cluster-snapshots --db-cluster-identifier <id> --snapshot-type automated` |
| EFS backups enabled | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) (`BackupPolicy: ENABLED`) | `aws efs describe-backup-policy --file-system-id <id>` |
| CloudTrail bucket versioning | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (`VersioningConfiguration: Enabled`) | `aws s3api get-bucket-versioning --bucket <bucket-name>` |
| CloudTrail bucket lifecycle (Glacier) | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) lifecycle rules | `aws s3api get-bucket-lifecycle-configuration --bucket <bucket-name>` |

---

## 6. Availability (A1.1)

| Evidence | Template Resource | How to Collect |
|----------|------------------|---------------|
| Multi-AZ deployment | [`MPPrivateSubnet1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L451) (AZ-A), [`MPPrivateSubnet2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L464) (AZ-B) | `aws ec2 describe-subnets` — verify resources in at least 2 AZs |
| ASG spans multiple AZs | [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956) | `aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[*].{Name:AutoScalingGroupName,AZs:AvailabilityZones,Min:MinSize,Max:MaxSize}"` |
| ASG min >= 2 for webapp | [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956) (`MinSize: 2`, `MaxSize: 4`) | Same command — verify MinSize >= 2 |
| Scaling policy active | [`WebAppScalingPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L977) (CPU target 70%) | `aws autoscaling describe-policies --auto-scaling-group-name <name>` |

---

## 7. Change Management (CC8.1)

| Evidence | Template Resource | Where to Find |
|----------|------------------|--------------|
| Infrastructure as Code | [Full template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) | GitHub repository — all resources defined in CloudFormation |
| Git history showing reviews | N/A | GitHub PR history with approvals |
| CloudFormation stack events | All resources | `aws cloudformation describe-stack-events --stack-name <name>` — shows all changes with timestamps |
| CloudFormation drift detection | All resources | `aws cloudformation detect-stack-drift --stack-name <name>` — proves no out-of-band changes |
| Change log | N/A | Change log (see [`CHANGE-MANAGEMENT-PROCEDURE.md`](CHANGE-MANAGEMENT-PROCEDURE.md) Section 4) |

---

## 8. Pre-Audit Checklist

Run this checklist 2 weeks before the audit period begins and again 1 week before the auditor arrives:

- [ ] [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) is logging (`IsLogging: true`)
- [ ] [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520) is active
- [ ] All CloudWatch alarms are in OK or ALARM state (not INSUFFICIENT_DATA)
- [ ] No security group has 0.0.0.0/0 on non-web ports
- [ ] [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027): `StorageEncrypted: true` and `DeletionProtection: true`
- [ ] [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929) has TLS 1.3 policy
- [ ] [`ALBHTTPRedirectListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L942) redirects HTTP to HTTPS
- [ ] KMS key rotation enabled on [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)
- [ ] [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) encryption and backup policy enabled
- [ ] Access review completed within last 90 days (see [`ACCESS-MANAGEMENT-PROCEDURE.md`](ACCESS-MANAGEMENT-PROCEDURE.md))
- [ ] Change log is up to date (see [`CHANGE-MANAGEMENT-PROCEDURE.md`](CHANGE-MANAGEMENT-PROCEDURE.md))
- [ ] CloudFormation drift detection shows no drift
- [ ] No emergency changes are undocumented

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — Full TSC mapping with template references
- [Architecture Overview](ARCHITECTURE-OVERVIEW.md) — Network diagrams and resource map
- [Access Management Procedure](ACCESS-MANAGEMENT-PROCEDURE.md) — Access review evidence
- [Change Management Procedure](CHANGE-MANAGEMENT-PROCEDURE.md) — Change log evidence
- [Monitoring and Alerting Procedure](MONITORING-AND-ALERTING-PROCEDURE.md) — Alert review evidence
- [Shared Responsibility Model](SHARED-RESPONSIBILITY-MODEL.md) — AWS vs. MP boundary for auditor

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial guide created |
