# Backup and Recovery Procedure

## SOC 2 Criteria: A1.2, A1.3

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

This procedure documents how data in the Merchant Protocol AWS infrastructure is backed up, how recovery is performed, and how backup integrity is verified.

---

## 1. Backup Inventory

| Data Store | Template Resource | Backup Method | Retention | Frequency | Encryption |
|-----------|------------------|--------------|-----------|-----------|-----------|
| Aurora MySQL | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | Automated snapshots | 35 days | Continuous (point-in-time) | KMS ([`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055)) |
| Aurora MySQL | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | Manual snapshots | Until deleted | On-demand (pre-change) | KMS ([`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055)) |
| EFS | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) | AWS Backup | Per AWS Backup plan | Daily (automatic) | KMS ([`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148)) |
| CloudTrail logs | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) | S3 with lifecycle | 7 years (Glacier after 90 days) | Continuous | KMS ([`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)) |
| VPC Flow Logs | [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) | CloudWatch Logs | Configurable (default 365 days) | Continuous | KMS ([`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)) |
| EC2 instances | [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987) | No persistent data — stateless, rebuilt from launch template | N/A | N/A | N/A |
| Infrastructure config | [Full template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) | Git repository | Indefinite | Every commit | GitHub encryption |

---

## 2. Aurora Database Backup

### 2.1 Automated Backups (Always Active)

- Aurora continuously backs up data to S3
- Point-in-time recovery (PITR) available for any second within the 35-day retention window (configured at [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027), `BackupRetentionPeriod: 35`)
- Backups are encrypted with [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055)
- No manual intervention needed — managed by AWS
- `DeletionProtection: true` prevents accidental cluster deletion
- `DeletionPolicy: Snapshot` ensures a final snapshot is taken if the CloudFormation stack is deleted

### 2.2 Manual Snapshots (Before Changes)

Create a manual snapshot before any significant database change:

```bash
aws rds create-db-cluster-snapshot \
  --db-cluster-identifier <cluster-id> \
  --db-cluster-snapshot-identifier <stack-name>-pre-change-$(date +%Y%m%d-%H%M%S)
```

### 2.3 Database Recovery

**Point-in-time recovery** (recover to a specific second):

```bash
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier <cluster-id> \
  --db-cluster-identifier <new-cluster-name> \
  --restore-to-time <ISO8601-timestamp> \
  --vpc-security-group-ids <RDSPrivate-sg-id> \
  --db-subnet-group-name <subnet-group-name>
```

**Snapshot recovery** (recover to snapshot point):

```bash
aws rds restore-db-cluster-from-snapshot \
  --snapshot-identifier <snapshot-id> \
  --db-cluster-identifier <new-cluster-name> \
  --engine aurora-mysql \
  --vpc-security-group-ids <RDSPrivate-sg-id> \
  --db-subnet-group-name <subnet-group-name>
```

After restoring, create a new DB instance in the restored cluster:

```bash
aws rds create-db-instance \
  --db-instance-identifier <new-instance-name> \
  --db-cluster-identifier <new-cluster-name> \
  --db-instance-class db.serverless \
  --engine aurora-mysql
```

Update the application to point to the new cluster endpoint.

---

## 3. EFS Backup

### 3.1 Automated Backup

- [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) has `BackupPolicy: ENABLED` which triggers AWS Backup integration
- Daily backups retained per the default AWS Backup plan
- Backups are encrypted with [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148)

### 3.2 EFS Recovery

```bash
# List available recovery points
aws backup list-recovery-points-by-resource \
  --resource-arn arn:aws:elasticfilesystem:<region>:<account>:file-system/<fs-id>

# Start restore job
aws backup start-restore-job \
  --recovery-point-arn <recovery-point-arn> \
  --iam-role-arn <backup-role-arn> \
  --metadata '{"file-system-id":"<fs-id>","Encrypted":"true","KmsKeyId":"<key-arn>","PerformanceMode":"maxIO","newFileSystem":"true"}'
```

---

## 4. EC2 Instance Recovery

EC2 instances are **stateless** — they are rebuilt from launch templates, not backed up individually.

**Recovery process**:
1. The [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956) automatically replaces unhealthy instances
2. If the launch template needs updating, deploy via CloudFormation stack update
3. Persistent data lives in [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) and [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103), not on instance storage

**If an instance needs forensic preservation** (see [`INCIDENT-RESPONSE-PROCEDURE.md`](INCIDENT-RESPONSE-PROCEDURE.md)):
```bash
# Create AMI of the instance before any changes
aws ec2 create-image \
  --instance-id <instance-id> \
  --name "forensic-$(date +%Y%m%d-%H%M%S)" \
  --no-reboot
```

---

## 5. CloudFormation Stack Recovery

If the CloudFormation stack itself is corrupted or deleted:

1. Stack template is in version control — redeploy from git
2. [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) has `DeletionPolicy: Snapshot` — a snapshot is automatically created on stack deletion
3. [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) has `DeletionPolicy: Retain` — bucket survives stack deletion
4. EFS: recovery depends on AWS Backup recovery points

**Full stack redeploy from git** (see [`DEPLOYMENT-GUIDE.md`](DEPLOYMENT-GUIDE.md) for full parameter list):
```bash
aws cloudformation create-stack \
  --stack-name <stack-name> \
  --template-body file://cloudformation-launchtemplates-soc2.yaml \
  --parameters ParameterKey=SSLCertificateArn,ParameterValue=<cert-arn> \
               ParameterKey=KeyPair,ParameterValue=<key-name> \
               ParameterKey=DBMasterUserPassword,ParameterValue=<password> \
               ParameterKey=BastionAllowedCIDR,ParameterValue=<cidr> \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## 6. Recovery Testing

### 6.1 Schedule

| Test | Frequency | Last Tested | Next Due |
|------|-----------|-------------|----------|
| Aurora PITR to test cluster | Quarterly | | |
| Aurora snapshot restore | Quarterly | | |
| EFS restore to new file system | Semi-annually | | |
| Full stack redeploy from git | Annually | | |

### 6.2 Test Procedure

1. Perform the recovery in a non-production environment
2. Verify:
   - [ ] Data integrity — spot-check records/files match production
   - [ ] Application connects and serves traffic
   - [ ] Encryption is intact on restored resources
3. Document results: date, tester, outcome, any issues
4. Clean up test resources after verification

### 6.3 Recovery Time Objectives

| Data Store | Template Resource | RTO (Recovery Time) | RPO (Data Loss) |
|-----------|------------------|-------------------|-----------------|
| Aurora MySQL | [`AuroraServerlessV2Cluster`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1027) | ~30 minutes (PITR) | Seconds (continuous backup) |
| EFS | [`FileSystemResource`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) | ~1-4 hours (depends on size) | Up to 24 hours (daily backup) |
| EC2 instances | [`WebAppASG`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L956) | ~5 minutes (ASG replacement) | None (stateless) |
| Full stack | [Full template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) | ~1 hour | Database: seconds, EFS: up to 24h |

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — A1.2 control mapping
- [Incident Response Procedure](INCIDENT-RESPONSE-PROCEDURE.md) — Forensic preservation and recovery during incidents
- [Deployment Guide](DEPLOYMENT-GUIDE.md) — Full stack deployment commands
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — How to verify backup controls for audit

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial procedure created |
