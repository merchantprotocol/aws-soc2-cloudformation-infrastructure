# Architecture Overview

## Merchant Protocol — AWS VPC Web Application Infrastructure

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

### Template Variants

| Template | Purpose | Status |
|----------|---------|--------|
| [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) | SOC 2 Type II ready deployment | **Recommended for production** |
| [`cloudformation-launchtemplates.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates.yaml) | Standard deployment (hardened) | Legacy — use SOC 2 template for new stacks |
| [`cloudformation-template.yaml`](../vpc-standard-2privatesubnets/cloudformation-template.yaml) | Launch Configurations (deprecated) | Do not use — AWS has deprecated Launch Configurations |

---

### Network Architecture

```
                         ┌──────────────────────────────────────────────────────────┐
                         │                     VPC (10.0.0.0/16)                    │
                         │                                                          │
     Internet            │   ┌─────────────────────┐  ┌─────────────────────┐       │
        │                │   │  Public Subnet AZ-A  │  │  Public Subnet AZ-B  │      │
        │                │   │    10.0.0.0/20       │  │    10.0.16.0/20      │      │
        ▼                │   │                      │  │                      │      │
  ┌───────────┐          │   │  ┌────────────────┐  │  │  ┌────────────────┐  │      │
  │  Internet │──────────┼───│  │  NAT Gateway   │  │  │  │  NAT Gateway   │  │      │
  │  Gateway  │          │   │  └───────┬────────┘  │  │  └───────┬────────┘  │      │
  └───────────┘          │   │          │           │  │          │           │      │
        │                │   │  ┌────────────────┐  │  │                      │      │
        ├────────────────┼───│  │ Bastion Host   │  │  │                      │      │
        │                │   │  │ (SSH gateway)  │  │  │                      │      │
        │                │   │  └────────────────┘  │  │                      │      │
        │                │   └─────────────────────┘  └─────────────────────┘       │
        │                │                                                          │
        ▼                │   ┌─────────────────────┐  ┌─────────────────────┐       │
  ┌───────────┐          │   │ Private Subnet AZ-A │  │ Private Subnet AZ-B │       │
  │    ALB    │──────────┼──▶│   10.0.128.0/20     │  │   10.0.144.0/20     │       │
  │  (HTTPS)  │          │   │                      │  │                      │      │
  └───────────┘          │   │  ┌────────────────┐  │  │  ┌────────────────┐  │      │
                         │   │  │  Web App (ASG)  │  │  │  │  Web App (ASG)  │ │     │
                         │   │  └───────┬────────┘  │  │  └───────┬────────┘  │     │
                         │   │          │           │  │          │           │      │
                         │   │  ┌────────────────┐  │  │  ┌────────────────┐  │      │
                         │   │  │ EFS Mount Tgt  │  │  │  │ EFS Mount Tgt  │  │      │
                         │   │  └────────────────┘  │  │  └────────────────┘  │      │
                         │   │          │           │  │          │           │      │
                         │   │  ┌────────────────┐  │  │                      │      │
                         │   │  │ Aurora MySQL   │  │  │  (Aurora replica)    │      │
                         │   │  │ (Serverless)   │  │  │                      │      │
                         │   │  └────────────────┘  │  │                      │      │
                         │   └─────────────────────┘  └─────────────────────┘       │
                         └──────────────────────────────────────────────────────────┘
```

### Template Resource Map

| Layer | Resources | Template Lines |
|-------|-----------|---------------|
| VPC & Subnets | [`MPVPC`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L299), [`MPPublicSubnet1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L341)/[`2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L354), [`MPPrivateSubnet1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L451)/[`2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L464) | L299–L486 |
| NAT Gateways | [`MPNatGateway1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L398), [`MPNatGateway2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L408) | L380–L416 |
| VPC Flow Logs | [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520), [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490), [`VPCFlowLogRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L497) | L490–L533 |
| CloudTrail | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648), [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) | L534–L664 |
| Security Groups | [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`ALBPublic`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L687), [`EC2PrivateAPPServers`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L711), [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740), [`EFSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L764) | L667–L782 |
| IAM Roles | [`BastionInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L784), [`WebAppInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L807) | L784–L849 |
| Bastion Host | [`BastionASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L852), [`BastionLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L871) | L852–L889 |
| ALB (HTTPS) | [`WebAppALB`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L891), [`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929), [`ALBHTTPRedirectListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L942) | L891–L954 |
| Web App | [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956), [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987), [`WebAppScalingPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L977) | L956–L1009 |
| Aurora MySQL | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027), [`AuroraDBInstance`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1088), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055) | L1011–L1101 |
| EFS Storage | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`AccessPointResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1203) | L1103–L1214 |
| CloudWatch Alarms | [`HighCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`DatabaseCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`UnauthorizedAPICallsAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | L1217+ |

### Traffic Flow

1. **Inbound web traffic**: Internet -> [`WebAppALB`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L891) (public subnets, port 443 only) -> Web App instances (private subnets, port 80)
2. **HTTP requests**: Automatically redirected to HTTPS (301) at the [`ALBHTTPRedirectListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L942)
3. **SSH administration**: Restricted CIDR -> Bastion Host ([`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) SG) -> Web App instances (private subnets)
4. **Database access**: Web App instances -> Aurora MySQL ([`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740) SG, TLS enforced via [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019))
5. **Shared storage**: Web App instances -> EFS mount targets ([`EFSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L764) SG, encryption in transit enforced)
6. **Outbound from private subnets**: Web App -> [`MPNatGateway1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L398)/[`2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L408) (per-AZ) -> Internet

### Security Group Relationships

```
Internet ──▶ ALBPublic (80, 443) ──▶ EC2PrivateAPPServers (80, 443 from ALB only)
                                          │
                                          ├──▶ RDSPrivate (3306, 5439 from AppServers only)
                                          │
                                          └──▶ EFSPrivate (2049 from AppServers only)

Allowed CIDR ──▶ EC2PublicBastion (SSH) ──▶ EC2PrivateAPPServers (SSH from Bastion only)
```

### Encryption Summary

| Layer | At Rest | In Transit | Template Reference |
|-------|---------|------------|-------------------|
| Web traffic | N/A | TLS 1.3 ([`ALBHTTPSListener`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L929)) | L929–L940 |
| Database | KMS ([`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), auto-rotation) | MySQL `require_secure_transport = ON` ([`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019)) | L1019–L1025, L1055–L1080 |
| File storage (EFS) | KMS ([`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), auto-rotation) | EFS policy denies unencrypted transport ([`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103)) | L1103–L1145, L1148–L1178 |
| Audit logs (CloudTrail) | KMS ([`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)) + S3 SSE | HTTPS ([`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) policy denies `SecureTransport: false`) | L534–L577, L580–L645 |
| VPC Flow Logs | KMS ([`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)) via CloudWatch | CloudWatch API (HTTPS) | L490–L496, L534–L577 |

### Monitoring & Logging

| System | Destination | Retention | Template Reference |
|--------|------------|-----------|-------------------|
| VPC Flow Logs | CloudWatch Log Group ([`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490)) | Configurable (default 365 days) | L490–L533 |
| CloudTrail API logs | S3 bucket ([`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580)) with Glacier archival | 7 years (2555 days), Glacier after 90 days | L580–L664 |
| CloudWatch Alarms | [`HighCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`DatabaseCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`UnauthorizedAPICallsAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | Indefinite | L1217+ |

### High Availability

| Component | Redundancy | Template Reference |
|-----------|-----------|-------------------|
| Web App | ASG min 2, max 4 across 2 AZs with CPU-based auto-scaling | [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956), [`WebAppScalingPolicy`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L977) |
| NAT Gateway | One per AZ (no cross-AZ single point of failure) | [`MPNatGateway1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L398), [`MPNatGateway2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L408) |
| ALB | Managed by AWS across 2 AZs | [`WebAppALB`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L891) |
| Aurora | Serverless v2 with multi-AZ capability | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) |
| EFS | Mount targets in both private subnets | [`MountTargetResource1`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1187)/[`2`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1187) |
| Bastion | ASG (min/max 1) for auto-recovery | [`BastionASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L852) |

### Parameters Required at Deploy Time

| Parameter | Template Line | Description | SOC 2 Relevance |
|-----------|--------------|-------------|-----------------|
| [`SSLCertificateArn`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L169) | L169 | ACM certificate for HTTPS | Required — no HTTP-only deployments allowed |
| [`BastionAllowedCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L154) | L154 | CIDR block for SSH access | Must be set to office/VPN range, not 0.0.0.0/0 |
| [`KeyPair`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L143) | L143 | EC2 key pair for SSH | Rotate regularly per access control policy |
| [`DBMasterUserPassword`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L13) | L13 | Aurora admin password | Store in Secrets Manager post-deploy |
| [`FlowLogRetentionDays`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L180) | L180 | VPC Flow Log retention | Default 365 days; adjust per retention policy |

### Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — Maps each TSC to specific template resources
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — AWS CLI commands to verify controls
- [Deployment Guide](DEPLOYMENT-GUIDE.md) — How to deploy and update the stack
- [Shared Responsibility Model](SHARED-RESPONSIBILITY-MODEL.md) — AWS vs. Merchant Protocol boundaries
