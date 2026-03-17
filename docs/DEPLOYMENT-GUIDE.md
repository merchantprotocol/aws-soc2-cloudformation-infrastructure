# Deployment Guide

## CloudFormation Template — SOC 2 Ready

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Prerequisites

Before deploying, you need to create **2 things** ahead of time. Everything else is created automatically by the template.

### Prerequisite 1: ACM SSL Certificate

The template requires an SSL certificate ARN for the [`SSLCertificateArn`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L169) parameter. CloudFormation cannot create this for you because it requires you to prove you own the domain.

**Option A: AWS Console (Easiest)**

1. Go to **AWS Console > Certificate Manager (ACM)**
2. Click **"Request a certificate"**
3. Choose **"Request a public certificate"**
4. Enter your domain name (e.g., `merchantprotocol.com`)
5. Add a Subject Alternative Name: `*.merchantprotocol.com` (covers all subdomains)
6. Choose **DNS validation**
7. Click **"Request"**
8. On the certificate details page, click **"Create records in Route 53"** (if you use Route 53) or manually add the CNAME records shown to your DNS provider
9. Wait for status to change from "Pending validation" to **"Issued"** (usually 5-30 minutes)
10. Copy the **Certificate ARN** (looks like `arn:aws:acm:us-east-1:123456789012:certificate/abc-def-123`)

**Option B: AWS CLI**

```bash
# Step 1: Request the certificate
aws acm request-certificate \
  --domain-name merchantprotocol.com \
  --subject-alternative-names "*.merchantprotocol.com" \
  --validation-method DNS

# Step 2: Get the DNS validation records
aws acm describe-certificate \
  --certificate-arn <arn-from-step-1> \
  --query "Certificate.DomainValidationOptions[*].ResourceRecord"

# Step 3: Add those CNAME records to your DNS provider
# Step 4: Wait for validation, then check status
aws acm describe-certificate \
  --certificate-arn <arn-from-step-1> \
  --query "Certificate.Status"
# Should return "ISSUED"
```

**Important**: The certificate must be in the **same AWS region** where you deploy the stack.

### Prerequisite 2: EC2 Key Pair

The template needs an existing key pair for SSH access (used by [`KeyPair`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L143) parameter). The CloudFormation Console will show a dropdown of your existing key pairs.

**Option A: AWS Console**

1. Go to **AWS Console > EC2 > Key Pairs** (under Network & Security)
2. Click **"Create key pair"**
3. Name it (e.g., `mp-production`)
4. Choose **RSA** and **.pem** format
5. Click **"Create key pair"** — it will auto-download the `.pem` file
6. Move it somewhere safe and set permissions: `chmod 400 mp-production.pem`

**Option B: AWS CLI**

```bash
aws ec2 create-key-pair \
  --key-name mp-production \
  --query 'KeyMaterial' \
  --output text > mp-production.pem
chmod 400 mp-production.pem
```

**Important**: Store this `.pem` file securely. If you lose it, you cannot recover it — you'd need to create a new key pair.

### What You'll Need at Deploy Time

| What | Where to Get It | Required? |
|------|----------------|-----------|
| SSL Certificate ARN | Prerequisite 1 above | **Yes — no default** |
| Key Pair name | Prerequisite 2 above | **Yes — no default** |
| Database password | You choose it (8+ chars, mixed case, number, symbol) | **Yes — no default** |
| Office/VPN CIDR | Your public IP range (Google "what is my IP" + add `/32` for single IP) | Has default but **should change** |
| Everything else | Has sensible defaults | Optional to change |

### IAM Permissions

The AWS user or role deploying the stack needs permissions to create: VPC, EC2, ELB, RDS, EFS, IAM roles, KMS keys, CloudTrail, CloudWatch, S3. An `AdministratorAccess` policy works, or create a scoped deployment role.

---

## Deploy the SOC 2 Template

### Option 1: AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name mp-production \
  --template-body file://vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml \
  --parameters \
    ParameterKey=SSLCertificateArn,ParameterValue=arn:aws:acm:us-east-1:123456789012:certificate/abc-123 \
    ParameterKey=KeyPair,ParameterValue=mp-production \
    ParameterKey=BastionAllowedCIDR,ParameterValue=203.0.113.0/24 \
    ParameterKey=DBMasterUserPassword,ParameterValue='YourP@ssw0rd!' \
    ParameterKey=DBMasterUsername,ParameterValue=admin \
    ParameterKey=DBName,ParameterValue=mp_saas \
  --capabilities CAPABILITY_NAMED_IAM \
  --tags Key=Environment,Value=production Key=ManagedBy,Value=CloudFormation
```

### Option 2: AWS Console

1. Go to CloudFormation > Create Stack > Upload a template file
2. Upload [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)
3. Fill in parameters — pay attention to the "SOC 2 Compliance Parameters" section at the top
4. On the review page, check "I acknowledge that AWS CloudFormation might create IAM resources with custom names"
5. Create stack

### Monitor Deployment

```bash
# Watch stack events
aws cloudformation describe-stack-events \
  --stack-name mp-production \
  --query "StackEvents[*].{Time:Timestamp,Status:ResourceStatus,Resource:LogicalResourceId,Reason:ResourceStatusReason}" \
  --output table

# Check stack status
aws cloudformation describe-stacks \
  --stack-name mp-production \
  --query "Stacks[0].StackStatus"
```

Expected deployment time: ~15-20 minutes (NAT Gateways and Aurora take the longest).

---

## What Gets Created

The template creates the following resources (see [`ARCHITECTURE-OVERVIEW.md`](ARCHITECTURE-OVERVIEW.md) for diagrams):

| Category | Resources | Template Lines |
|----------|-----------|---------------|
| Networking | VPC, 2 public subnets, 2 private subnets, IGW, 2 NAT Gateways, route tables | [L299–L486](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L299) |
| Monitoring | VPC Flow Logs, CloudTrail, CloudWatch Alarms | [L490–L664](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) |
| Security | 5 security groups, 2 IAM roles, 3 KMS keys | [L534–L849](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534) |
| Compute | Bastion host ASG, Web App ASG (min 2), ALB with HTTPS | [L852–L1009](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L852) |
| Database | Aurora MySQL Serverless v2 (encrypted, TLS enforced) | [L1011–L1101](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1011) |
| Storage | EFS (encrypted, backups enabled) | [L1103–L1214](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1103) |

---

## Post-Deployment Steps

These steps must be completed after the stack is created:

### 1. Store Database Password in Secrets Manager

Do not leave the database password only in CloudFormation parameters:

```bash
aws secretsmanager create-secret \
  --name mp-production/aurora-master \
  --description "Aurora master credentials for mp-production stack" \
  --secret-string '{"username":"admin","password":"YourP@ssw0rd!","host":"<cluster-endpoint>","port":"3306","dbname":"mp_saas"}'
```

### 2. Configure DNS

Point your domain to the [`WebAppALB`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L891):

```bash
# Get ALB DNS name from stack outputs
aws cloudformation describe-stacks \
  --stack-name mp-production \
  --query "Stacks[0].Outputs[?OutputKey=='ALBDNSName'].OutputValue" \
  --output text
```

Create a CNAME or Route 53 alias record pointing your domain to this ALB DNS name.

### 3. Configure SNS Alert Routing

```bash
# Create alerts topic
aws sns create-topic --name mp-production-alerts

# Subscribe your team
aws sns subscribe \
  --topic-arn arn:aws:sns:<region>:<account>:mp-production-alerts \
  --protocol email \
  --notification-endpoint ops@merchantprotocol.com
```

Then add the SNS topic ARN as an action to each CloudWatch alarm.

### 4. Verify SOC 2 Controls Are Active

Run through the quick verification (see [`EVIDENCE-COLLECTION-GUIDE.md`](EVIDENCE-COLLECTION-GUIDE.md) Section 8 for the full checklist):

```bash
# CloudTrail is logging
aws cloudtrail get-trail-status --name mp-production-audit-trail

# VPC Flow Logs are active
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$(aws cloudformation describe-stacks --stack-name mp-production --query "Stacks[0].Outputs[?OutputKey=='outputVPC'].OutputValue" --output text)"

# Aurora encryption and deletion protection
aws rds describe-db-clusters --query "DBClusters[?contains(DBClusterIdentifier, 'mp-production')].{Encrypted:StorageEncrypted,DeletionProtection:DeletionProtection,BackupRetention:BackupRetentionPeriod}"

# ALB has HTTPS listener
aws elbv2 describe-listeners --load-balancer-arn $(aws elbv2 describe-load-balancers --names mp-production-ALB --query "LoadBalancers[0].LoadBalancerArn" --output text)
```

### 5. Run Initial CloudFormation Drift Detection

Establish a clean baseline:

```bash
aws cloudformation detect-stack-drift --stack-name mp-production
```

---

## Updating the Stack

Follow the [`CHANGE-MANAGEMENT-PROCEDURE.md`](CHANGE-MANAGEMENT-PROCEDURE.md) process, then:

```bash
# Create a changeset first (review before applying)
aws cloudformation create-change-set \
  --stack-name mp-production \
  --change-set-name update-$(date +%Y%m%d-%H%M%S) \
  --template-body file://vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml \
  --parameters ParameterKey=SSLCertificateArn,UsePreviousValue=true \
               ParameterKey=KeyPair,UsePreviousValue=true \
               ParameterKey=BastionAllowedCIDR,UsePreviousValue=true \
               ParameterKey=DBMasterUserPassword,UsePreviousValue=true \
  --capabilities CAPABILITY_NAMED_IAM

# Review the changeset
aws cloudformation describe-change-set \
  --stack-name mp-production \
  --change-set-name update-<timestamp>

# If changes look correct, execute
aws cloudformation execute-change-set \
  --stack-name mp-production \
  --change-set-name update-<timestamp>
```

---

## Connecting to Resources

### SSH to Bastion

```bash
ssh -i mp-production.pem ec2-user@<bastion-public-ip> -p <ssh-port>
```

### SSH to Web App (through Bastion)

```bash
ssh -i mp-production.pem -J ec2-user@<bastion-ip>:<ssh-port> ec2-user@<webapp-private-ip>
```

### Database (through Bastion SSH tunnel)

The database requires TLS (enforced by [`AuroraClusterParameterGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1019)):

```bash
# Open tunnel
ssh -i mp-production.pem -L 3306:<aurora-endpoint>:3306 ec2-user@<bastion-ip> -p <ssh-port> -N &

# Connect with TLS required
mysql -h 127.0.0.1 -P 3306 -u admin -p --ssl-mode=REQUIRED mp_saas
```

---

## Related Documents

- [Architecture Overview](ARCHITECTURE-OVERVIEW.md) — Network diagrams and resource map
- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — What controls are deployed
- [Change Management Procedure](CHANGE-MANAGEMENT-PROCEDURE.md) — Process for stack updates
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — Post-deploy verification checklist

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial guide created |
