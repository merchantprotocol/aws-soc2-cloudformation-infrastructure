# AWS Shared Responsibility Model — SOC 2 Mapping

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

SOC 2 auditors need to understand the boundary between what AWS is responsible for and what Merchant Protocol is responsible for. This document maps that boundary for our specific infrastructure.

---

## The Model

```
┌─────────────────────────────────────────────────────────┐
│              MERCHANT PROTOCOL RESPONSIBILITY            │
│                                                          │
│   Application code, data, IAM configuration,             │
│   security groups, encryption config, OS patching,       │
│   network config, monitoring, access management,         │
│   CloudFormation templates, backup verification          │
│                                                          │
├─────────────────────────────────────────────────────────┤
│                   AWS RESPONSIBILITY                     │
│                                                          │
│   Physical security, hardware, hypervisor,               │
│   managed service internals (RDS engine, EFS storage,    │
│   ALB infrastructure, KMS HSMs, CloudTrail backend,      │
│   S3 durability, AZ infrastructure)                      │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Detailed Mapping by Service

### EC2 (Bastion + Web App Instances)

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| Physical server security | AWS | N/A |
| Hypervisor patching | AWS | N/A |
| AMI selection and OS patching | **Merchant Protocol** | [`LatestAMI`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L128) (Amazon Linux 2023 via SSM) |
| Security group configuration | **Merchant Protocol** | [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`EC2PrivateAPPServers`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L711) |
| IAM instance role and permissions | **Merchant Protocol** | [`BastionInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L784), [`WebAppInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L807) |
| IMDSv2 enforcement | **Merchant Protocol** | [`BastionLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L871), [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987) (`HttpTokens: required`) |
| Application security on instances | **Merchant Protocol** | [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987) UserData |

### Aurora MySQL (Serverless v2)

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| Database engine patching | **AWS** (minor) / **MP** (major) | [`AuroraDBInstance`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1088) (`AutoMinorVersionUpgrade: true`) |
| Storage durability (6-way replication) | AWS | N/A |
| Encryption at rest implementation | AWS (KMS + Aurora integration) | N/A |
| Encryption at rest configuration | **Merchant Protocol** | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`StorageEncrypted: true`), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055) |
| Encryption in transit configuration | **Merchant Protocol** | [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019) (`require_secure_transport: ON`) |
| Backup retention configuration | **Merchant Protocol** | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`BackupRetentionPeriod: 35`) |
| Deletion protection | **Merchant Protocol** | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) (`DeletionProtection: true`, `DeletionPolicy: Snapshot`) |
| Network isolation | **Merchant Protocol** | [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740) SG, [`AuroraDBSubnetGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1011) |
| Database user management | **Merchant Protocol** | Operational (see [`ACCESS-MANAGEMENT-PROCEDURE.md`](ACCESS-MANAGEMENT-PROCEDURE.md)) |
| Query performance and optimization | **Merchant Protocol** | Operational |

### ALB (Application Load Balancer)

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| ALB infrastructure and availability | AWS | N/A |
| TLS certificate management | **Merchant Protocol** | [`SSLCertificateArn`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L169) parameter |
| TLS policy selection | **Merchant Protocol** | [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929) (`ELBSecurityPolicy-TLS13-1-2-2021-06`) |
| Listener configuration (HTTPS, redirect) | **Merchant Protocol** | [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929), [`ALBHTTPRedirectListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L942) |
| Security group (allowed ports) | **Merchant Protocol** | [`ALBPublic`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L687) |

### EFS (Elastic File System)

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| Storage durability and availability | AWS | N/A |
| Encryption at rest (KMS integration) | AWS | N/A |
| Encryption configuration | **Merchant Protocol** | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) (`Encrypted: true`), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148) |
| Encryption in transit enforcement | **Merchant Protocol** | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) > `FileSystemPolicy` (denies `SecureTransport: false`) |
| Backup enablement | **Merchant Protocol** | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) (`BackupPolicy: ENABLED`) |
| Access point permissions | **Merchant Protocol** | [`AccessPointResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1203) (`Permissions: 0750`) |
| Network isolation | **Merchant Protocol** | [`EFSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L764) SG, [`MountTargetResource1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1187)/`2` |

### KMS (Key Management Service)

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| HSM security and key storage | AWS | FIPS 140-2 Level 3 |
| Key rotation implementation | AWS | N/A |
| Key rotation enablement | **Merchant Protocol** | [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) (`EnableKeyRotation: true`) |
| Key policy (who can use keys) | **Merchant Protocol** | Same resources — scoped policies, no `kms:*` wildcard |

### CloudTrail

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| API event capture infrastructure | AWS | N/A |
| Trail configuration | **Merchant Protocol** | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) (log validation, multi-region) |
| Log storage and lifecycle | **Merchant Protocol** | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (Glacier archival, 7-year retention) |
| Log integrity verification | **Merchant Protocol** | Must run `validate-logs` periodically |
| Log review and response | **Merchant Protocol** | Per [`MONITORING-AND-ALERTING-PROCEDURE.md`](MONITORING-AND-ALERTING-PROCEDURE.md) |

### VPC Networking

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| Physical network infrastructure | AWS | N/A |
| AZ isolation | AWS | N/A |
| VPC design and CIDR allocation | **Merchant Protocol** | [`MPVPC`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L299), [`paramVpcCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L98) |
| Route table configuration | **Merchant Protocol** | [`MPPrivateRouteTable1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L419)/[`2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L427) (private subnets via NAT only) |
| NAT Gateway provisioning | **Merchant Protocol** | [`MPNatGateway1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L398)/[`2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L408) (one per AZ) |
| Flow Log configuration | **Merchant Protocol** | [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520) (ALL traffic logged) |
| Flow Log review | **Merchant Protocol** | Per [`MONITORING-AND-ALERTING-PROCEDURE.md`](MONITORING-AND-ALERTING-PROCEDURE.md) |

### S3 (CloudTrail Bucket)

| Responsibility | Owner | Template Reference |
|---------------|-------|-------------------|
| Storage durability (11 9's) | AWS | N/A |
| Bucket encryption configuration | **Merchant Protocol** | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (SSE-KMS via [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)) |
| Bucket policy (deny HTTP) | **Merchant Protocol** | [`CloudTrailBucketPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L610) (denies `SecureTransport: false`) |
| Public access block | **Merchant Protocol** | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (all four block settings enabled) |
| Versioning | **Merchant Protocol** | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (`VersioningConfiguration: Enabled`) |
| Lifecycle (Glacier archival) | **Merchant Protocol** | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (90 days to Glacier, 7 year retention) |

---

## What This Means for SOC 2

### AWS SOC 2 Reports

AWS maintains its own SOC 2 Type II reports for the services we use. These can be requested through AWS Artifact:

1. Go to AWS Console > AWS Artifact
2. Download the latest "SOC 2 Type II" report
3. Provide to your auditor as evidence of subservice organization controls

Your auditor may issue a "carve-out" or "inclusive" opinion regarding AWS:
- **Carve-out**: AWS controls are excluded from your audit scope (you reference AWS's own SOC 2 report)
- **Inclusive**: AWS controls are included (rare, more complex)

Most organizations use the carve-out method.

### Our Responsibility Summary

For the SOC 2 audit, Merchant Protocol must demonstrate:

1. **We configured AWS services securely** — the [CloudFormation template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) is the evidence
2. **We monitor and respond to events** — [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648), [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520), CloudWatch alarms, and our [procedures](MONITORING-AND-ALERTING-PROCEDURE.md)
3. **We control access** — IAM roles, security groups, bastion pattern, KMS key policies (see [`ACCESS-MANAGEMENT-PROCEDURE.md`](ACCESS-MANAGEMENT-PROCEDURE.md))
4. **We protect data** — encryption at rest and in transit, backup retention, deletion protection (see [`BACKUP-AND-RECOVERY-PROCEDURE.md`](BACKUP-AND-RECOVERY-PROCEDURE.md))
5. **We manage changes** — infrastructure as code, git history, change management procedure (see [`CHANGE-MANAGEMENT-PROCEDURE.md`](CHANGE-MANAGEMENT-PROCEDURE.md))
6. **AWS handles the rest** — reference AWS SOC 2 report via carve-out

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — Full TSC mapping with template references
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — How to collect evidence for each responsibility
- [Architecture Overview](ARCHITECTURE-OVERVIEW.md) — Visual representation of the infrastructure

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial document created |
