# SOC 2 Type II Control Matrix

## Merchant Protocol — AWS Infrastructure Controls

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

This matrix maps each SOC 2 Trust Service Criteria (TSC) to the specific controls implemented in our CloudFormation infrastructure. Use this document for auditor walkthroughs, evidence collection, and gap analysis.

---

## How to Use This Document

1. **During audit prep**: Walk each row and confirm the control is still active and evidence is available
2. **During the audit**: Provide this matrix to the auditor as a starting point for infrastructure controls
3. **Ongoing**: Update the "Evidence Location" column whenever infrastructure changes are made

---

## CC6 — Logical and Physical Access Controls

| TSC ID | Criteria | Control Implemented | Template Reference | Evidence Location | Status |
|--------|----------|-------------------|-------------------|-------------------|--------|
| CC6.1 | Logical access security | VPC with public/private subnet isolation; all app servers and databases in private subnets only | [`MPVPC`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L299), [`MPPrivateSubnet1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L451), [`MPPrivateSubnet2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L464) | VPC console, subnet route tables | Active |
| CC6.1 | Encryption at rest — Database | Aurora cluster encrypted with dedicated KMS key, automatic key rotation enabled | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`StorageEncrypted: true`), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055) | RDS console > Encryption, KMS console | Active |
| CC6.1 | Encryption at rest — File storage | EFS encrypted with dedicated KMS key, automatic key rotation enabled | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) (`Encrypted: true`), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148) | EFS console > Encryption, KMS console | Active |
| CC6.1 | Encryption in transit — Web traffic | ALB enforces HTTPS with TLS 1.3 policy; HTTP automatically redirects to HTTPS | [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929) (`ELBSecurityPolicy-TLS13-1-2-2021-06`), [`ALBHTTPRedirectListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L942) | ALB console > Listeners | Active |
| CC6.1 | Encryption in transit — Database | Aurora cluster parameter group enforces `require_secure_transport = ON` | [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019) | RDS console > Parameter groups | Active |
| CC6.1 | Encryption in transit — File storage | EFS file system policy denies unencrypted transport (`aws:SecureTransport: false`) | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) > `FileSystemPolicy` | EFS console > File system policy | Active |
| CC6.1 | Instance metadata protection | IMDSv2 required on all EC2 instances (prevents SSRF-based credential theft) | [`BastionLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L871), [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987) (`HttpTokens: required`) | EC2 console > Launch templates | Active |
| CC6.1 | No public database access | RDSPublic security group removed; database only accessible from app server security group | [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740) SG | VPC console > Security Groups | Active |
| CC6.3 | Least privilege IAM | Dedicated IAM roles per instance type with scoped policies; no shared credentials | [`BastionInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L784), [`WebAppInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L807) | IAM console > Roles | Active |
| CC6.3 | KMS key scoping | KMS key policies scoped to specific actions (no `kms:*` wildcard); key rotation enabled | [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) | KMS console > Key policies | Active |
| CC6.6 | Network access restrictions | Bastion SSH restricted to `BastionAllowedCIDR` parameter (office/VPN CIDR); app servers only accept traffic from ALB and bastion security groups | [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`EC2PrivateAPPServers`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L711) | VPC console > Security Groups | Active |
| CC6.6 | Database network isolation | Aurora cluster in private subnets; security group only permits MySQL/Redshift from app server SG | [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740), [`AuroraDBSubnetGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1011) | VPC console > Security Groups | Active |
| CC6.6 | EFS network isolation | EFS mount targets in private subnets; security group only permits NFS from app server SG | [`EFSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L764), [`MountTargetResource1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1187) | VPC console > Security Groups | Active |
| CC6.7 | Data transmission security | All external data transmitted over TLS 1.3; internal traffic routed through private subnets | [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929), [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019), [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) | See CC6.1 entries above | Active |

## CC7 — System Operations / Monitoring

| TSC ID | Criteria | Control Implemented | Template Reference | Evidence Location | Status |
|--------|----------|-------------------|-------------------|-------------------|--------|
| CC7.1 | Infrastructure monitoring | VPC Flow Logs capture ALL traffic (accept + reject) to encrypted CloudWatch log group | [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520), [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) | CloudWatch console > Log groups > `/vpc/flowlogs/` | Active |
| CC7.1 | API audit trail | CloudTrail logs all API calls to encrypted S3 bucket with log file validation | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648), [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) | CloudTrail console, S3 bucket | Active |
| CC7.2 | Anomaly detection — Compute | CloudWatch alarm for WebApp ASG CPU > 80% | [`HighCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | CloudWatch console > Alarms | Active |
| CC7.2 | Anomaly detection — Database | CloudWatch alarm for Aurora CPU > 80% | [`DatabaseCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | CloudWatch console > Alarms | Active |
| CC7.2 | Anomaly detection — Security | CloudWatch alarm for unauthorized API calls exceeding threshold | [`UnauthorizedAPICallsAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | CloudWatch console > Alarms | Active |
| CC7.3 | Log integrity | CloudTrail log file validation enabled; S3 bucket versioning enabled | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) (`EnableLogFileValidation: true`), [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (`VersioningConfiguration`) | CloudTrail console > Trail details | Active |
| CC7.3 | Log retention | VPC Flow Logs retained for configurable period (default 365 days); CloudTrail logs archived to Glacier after 90 days, retained 7 years | [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) (`RetentionInDays`), [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) lifecycle | CloudWatch + S3 console | Active |
| CC7.3 | Log encryption | All logs encrypted with KMS key (CloudTrail S3 bucket + CloudWatch log group) | [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) | KMS console | Active |

## CC8 — Change Management

| TSC ID | Criteria | Control Implemented | Template Reference | Evidence Location | Status |
|--------|----------|-------------------|-------------------|-------------------|--------|
| CC8.1 | Infrastructure as Code | All infrastructure defined in version-controlled CloudFormation templates | [Full template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) | GitHub repository | Active |
| CC8.1 | Automated patching | `AutoMinorVersionUpgrade: true` on Aurora DB instance; AMI sourced from SSM latest parameter | [`AuroraDBInstance`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1088), [`LatestAMI`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L128) parameter | RDS console, SSM console | Active |

## A1 — Availability

| TSC ID | Criteria | Control Implemented | Template Reference | Evidence Location | Status |
|--------|----------|-------------------|-------------------|-------------------|--------|
| A1.1 | High availability — Compute | Web app ASG spans 2 AZs with min 2 instances; auto-scaling policy (CPU target 70%) scales to max 4 | [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956), [`WebAppScalingPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L977) | EC2 console > Auto Scaling Groups | Active |
| A1.1 | High availability — Network | NAT Gateway per AZ; ALB across 2 AZs | [`MPNatGateway1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L398)/[`MPNatGateway2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L408), [`WebAppALB`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L891) | VPC console, EC2 console | Active |
| A1.2 | Data protection — Database | Aurora deletion protection enabled; DeletionPolicy: Snapshot; 35-day backup retention | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | RDS console > Cluster details | Active |
| A1.2 | Data protection — File storage | EFS automatic backups enabled; lifecycle policies for cost optimization | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) (`BackupPolicy: ENABLED`) | EFS console > File system | Active |
| A1.2 | Data protection — Audit logs | CloudTrail S3 bucket has DeletionPolicy: Retain; versioning enabled; Glacier archival | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) | S3 console | Active |

---

## Controls NOT Covered by This Template

These controls must be addressed by other systems or organizational processes:

| Area | What's Needed | Where to Address |
|------|--------------|-----------------|
| WAF / DDoS protection | Web Application Firewall, rate limiting, bot detection | Handled externally (per Merchant Protocol architecture) |
| CI/CD pipeline security | Code review, automated testing, deployment approvals | Handled externally (per Merchant Protocol architecture) |
| Identity provider / SSO | Centralized authentication, MFA enforcement | AWS IAM Identity Center or external IdP |
| Vulnerability scanning | Regular CVE scanning of AMIs and dependencies | AWS Inspector, third-party scanner |
| Penetration testing | Annual or periodic penetration tests | Contracted third party |
| Security awareness training | Employee training on security policies | HR/compliance program |
| Incident response | Documented IR plan with roles and escalation | See [`INCIDENT-RESPONSE-PROCEDURE.md`](INCIDENT-RESPONSE-PROCEDURE.md) |
| Vendor management | Third-party risk assessments | Compliance program |
| Background checks | Employee screening | HR program |
| Physical security | Data center access controls | AWS responsibility (see [`SHARED-RESPONSIBILITY-MODEL.md`](SHARED-RESPONSIBILITY-MODEL.md)) |

---

## Related Documents

- [Architecture Overview](ARCHITECTURE-OVERVIEW.md) — Network diagrams, traffic flows, encryption summary
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — Exact AWS CLI commands to collect audit evidence
- [Access Management Procedure](ACCESS-MANAGEMENT-PROCEDURE.md) — CC6.1, CC6.2, CC6.3, CC6.6
- [Change Management Procedure](CHANGE-MANAGEMENT-PROCEDURE.md) — CC8.1
- [Incident Response Procedure](INCIDENT-RESPONSE-PROCEDURE.md) — CC7.3, CC7.4, CC7.5
- [Backup and Recovery Procedure](BACKUP-AND-RECOVERY-PROCEDURE.md) — A1.2, A1.3
- [Monitoring and Alerting Procedure](MONITORING-AND-ALERTING-PROCEDURE.md) — CC7.1, CC7.2
- [Deployment Guide](DEPLOYMENT-GUIDE.md) — How to deploy and update the template
- [Shared Responsibility Model](SHARED-RESPONSIBILITY-MODEL.md) — AWS vs. Merchant Protocol responsibilities

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial matrix created from SOC 2 template review |
