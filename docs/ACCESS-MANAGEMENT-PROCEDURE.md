# Access Management Procedure

## SOC 2 Criteria: CC6.1, CC6.2, CC6.3, CC6.6

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

This procedure defines how access to the Merchant Protocol AWS infrastructure is granted, reviewed, modified, and revoked. It applies to all personnel who interact with production infrastructure provisioned by our CloudFormation templates.

---

## 1. Access Roles

### 1.1 Infrastructure Access Levels

| Role | AWS Console | SSH (Bastion) | Database | Who |
|------|------------|---------------|----------|-----|
| Infrastructure Admin | Full (IAM admin) | Yes | Yes (via bastion tunnel) | DevOps lead |
| Developer | Read-only console | Yes (during approved windows) | Read-only (via bastion tunnel) | Engineering team |
| DBA | RDS console only | No | Read/Write (via bastion tunnel) | Database administrator |
| Auditor | Read-only console, CloudTrail, CloudWatch | No | No | External auditor |
| No Access | None | None | None | All others |

### 1.2 AWS IAM Policies

- All human users access AWS through **IAM Identity Center (SSO)** with MFA enforced
- No IAM user accounts with long-lived access keys for human users
- Service accounts (CI/CD) use IAM roles with scoped policies, not access keys
- EC2 instances use IAM instance profiles — no credentials stored on instances:
  - [`BastionInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L784) — SSM access only
  - [`WebAppInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L807) — SSM, EFS mount, and CloudWatch Logs

---

## 2. Granting Access

### 2.1 New Access Request

1. Requestor submits access request specifying:
   - Name and role
   - Access level needed (from table above)
   - Business justification
   - Duration (permanent or time-boxed)
2. Request approved by infrastructure admin and requestor's manager
3. Infrastructure admin provisions access:
   - **AWS Console**: Add user to appropriate SSO permission set
   - **SSH**: Add user's public key to bastion authorized_keys; update [`BastionAllowedCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L154) if their IP/VPN range is not already included
   - **Database**: Create database user with minimum required privileges via bastion SSH tunnel
4. Access grant logged in access register (see Section 5)

### 2.2 Emergency Access

1. Emergency access may be granted verbally by infrastructure admin
2. Must be documented within 24 hours
3. Emergency access is time-boxed to 72 hours maximum
4. Must be reviewed in next scheduled access review

---

## 3. Modifying Access

1. Role changes trigger access review for affected user
2. Access modifications follow the same approval process as new access
3. Previous access removed before new access granted (no privilege accumulation)

---

## 4. Revoking Access

### 4.1 Voluntary Departure

1. HR notifies infrastructure admin of departure date
2. On departure date:
   - Remove from IAM Identity Center
   - Remove SSH key from bastion
   - Drop database user accounts
   - Rotate any shared credentials the user had access to
3. Confirm revocation within 24 hours of departure

### 4.2 Involuntary Departure

1. Access revoked **immediately** upon notification from HR
2. All steps from 4.1 completed same day
3. Review CloudTrail logs for last 30 days of user activity (via [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648))

### 4.3 Role Change

1. Remove access for previous role
2. Grant access for new role per Section 2

---

## 5. Access Review

### 5.1 Quarterly Review

Performed every 90 days:

1. Infrastructure admin generates list of all users with access:
   - AWS IAM Identity Center users and permission sets
   - Bastion authorized_keys
   - Database user accounts (`SELECT user, host FROM mysql.user`)
2. For each user, verify:
   - [ ] User is still employed
   - [ ] Access level matches current role
   - [ ] No excessive privileges
   - [ ] MFA is enabled
3. Remove or adjust access as needed
4. Document review results and any changes made
5. Sign-off by infrastructure admin and management

### 5.2 Annual Review

In addition to quarterly review:

1. Review IAM policies and permission sets for least privilege
2. Review security group rules for any unauthorized changes
3. Review KMS key policies for [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)
4. Verify [`BastionAllowedCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L154) only contains current office/VPN CIDRs
5. Rotate EC2 key pairs

---

## 6. Access Register Template

Maintain this register for auditor review:

| Date | User | Action (Grant/Modify/Revoke) | Access Level | Approved By | Justification | Expiry |
|------|------|-----|------|------|------|------|
| | | | | | | |

---

## 7. Technical Controls Enforced by Template

These controls are enforced automatically by the [SOC 2 CloudFormation template](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml) and cannot be bypassed without a stack update (which is logged by [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648)):

| Control | Template Resource | Line |
|---------|------------------|------|
| SSH restricted to allowed CIDR | [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) SG with [`BastionAllowedCIDR`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L154) | L667, L154 |
| No direct SSH to app servers | [`EC2PrivateAPPServers`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L711) only allows SSH from bastion SG | L711 |
| No public database access | [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740) SG only allows traffic from app server SG | L740 |
| Database in private subnets | [`AuroraDBSubnetGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1011) references only private subnets | L1011 |
| Instance roles (no stored credentials) | [`BastionInstanceProfile`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L801), [`WebAppInstanceProfile`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L845) | L801, L845 |
| IMDSv2 required | [`BastionLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L871), [`WebAppLaunchTemplate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L987) (`MetadataOptions.HttpTokens: required`) | L871, L987 |

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — Full mapping of controls to TSC criteria
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — How to collect access control evidence for audit
- [Incident Response Procedure](INCIDENT-RESPONSE-PROCEDURE.md) — Access revocation during incidents

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial procedure created |
