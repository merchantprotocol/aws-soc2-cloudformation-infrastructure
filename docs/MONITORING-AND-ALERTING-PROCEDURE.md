# Monitoring and Alerting Procedure

## SOC 2 Criteria: CC7.1, CC7.2

**Template**: [`cloudformation-launchtemplates-soc2.yaml`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml)

---

## Purpose

This procedure documents what is monitored in the Merchant Protocol AWS infrastructure, how alerts are configured, and how the team responds to alerts.

---

## 1. Monitoring Systems

| System | What It Monitors | Template Resource | Data Destination | Retention |
|--------|-----------------|------------------|-----------------|-----------|
| VPC Flow Logs | All network traffic (accept/reject) across the VPC | [`VPCFlowLog`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L520) | [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) (CloudWatch) | Configurable (default 365 days, [`FlowLogRetentionDays`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L180)) |
| CloudTrail | All AWS API calls (who, what, when, from where) | [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) | [`CloudTrailBucket`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L580) (encrypted S3) | 7 years (Glacier after 90 days) |
| CloudWatch Metrics | EC2 CPU, ALB request counts, Aurora performance | Built-in (AWS managed) | CloudWatch Metrics | 15 months (standard) |
| CloudWatch Alarms | Threshold breaches on key metrics | See Section 2 | CloudWatch Alarms | Indefinite |

---

## 2. CloudWatch Alarms

### 2.1 Alarms Deployed by Template

| Alarm | Template Resource | Metric | Threshold | Period | Evaluation | Severity |
|-------|------------------|--------|-----------|--------|-----------|----------|
| `{stack}-HighCPU` | [`HighCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | EC2 ASG average CPU | > 80% | 5 min | 2 consecutive | Medium |
| `{stack}-AuroraHighCPU` | [`DatabaseCPUAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | Aurora cluster CPU | > 80% | 5 min | 2 consecutive | Medium |
| `{stack}-UnauthorizedAPICalls` | [`UnauthorizedAPICallsAlarm`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667) | Unauthorized API calls | > 5 in period | 5 min | 1 | High |

### 2.2 Recommended Additional Alarms

These are not in the template but should be configured manually or in a separate monitoring stack:

| Alarm | Metric | Threshold | Why |
|-------|--------|-----------|-----|
| ALB 5xx error rate | `HTTPCode_ELB_5XX_Count` | > 10 in 5 min | Application errors or outage |
| ALB unhealthy targets | `UnHealthyHostCount` | > 0 for 5 min | Instance health issue |
| Aurora connections | `DatabaseConnections` | > 80% of max | Connection pool exhaustion |
| Aurora free memory | `FreeableMemory` | < 256 MB | Memory pressure |
| EFS burst credits | `BurstCreditBalance` | < 1 TB | I/O throttling risk |
| NAT Gateway errors | `ErrorPortAllocation` | > 0 | Network connectivity issue |

### 2.3 Alert Routing

Configure SNS topics to route alerts (see [`DEPLOYMENT-GUIDE.md`](DEPLOYMENT-GUIDE.md) post-deployment step 3):

```bash
# Create SNS topic for alerts
aws sns create-topic --name <stack-name>-alerts

# Subscribe team email
aws sns subscribe \
  --topic-arn <topic-arn> \
  --protocol email \
  --notification-endpoint ops-team@merchantprotocol.com

# Add SNS action to alarms
aws cloudwatch put-metric-alarm \
  --alarm-name <alarm-name> \
  --alarm-actions <topic-arn> \
  --ok-actions <topic-arn>
```

---

## 3. Log Review Procedures

### 3.1 Daily Review

Responsible: On-call engineer

- [ ] Check CloudWatch Alarms dashboard — any alarms in ALARM state?
- [ ] Review ALB access logs for unusual patterns (if enabled)
- [ ] Spot-check [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) for rejected traffic to sensitive ports (3306, 22)

### 3.2 Weekly Review

Responsible: Infrastructure admin

- [ ] Review [`CloudTrail`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L648) for:
  - Security group modifications (changes to [`EC2PublicBastion`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L667), [`RDSPrivate`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L740), etc.)
  - IAM policy changes (modifications to [`BastionInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L784), [`WebAppInstanceRole`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L807))
  - KMS key operations on [`DatabaseEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1055), [`FileSystemKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L1148), [`LogEncryptionKey`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L534)
  - CloudFormation stack updates
  - Any `ConsoleLogin` events from unexpected IPs
- [ ] Review [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) for:
  - Unusual outbound data volumes (data exfiltration indicators)
  - Connections to/from unexpected IP ranges
  - Rejected traffic patterns

### 3.3 Monthly Review

Responsible: Infrastructure admin + management

- [ ] Review alarm history — were any alarms missed or unacknowledged?
- [ ] Review alert response times — are SLAs being met?
- [ ] Check CloudFormation drift detection — any unauthorized changes?
- [ ] Verify all monitoring systems are operational (see [`EVIDENCE-COLLECTION-GUIDE.md`](EVIDENCE-COLLECTION-GUIDE.md) Section 8 pre-audit checklist)

---

## 4. Useful CloudWatch Insights Queries

Run these against the [`VPCFlowLogGroup`](../vpc-standard-2privatesubnets/cloudformation-launchtemplates-soc2.yaml#L490) log group:

### Rejected traffic to database ports
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT" and (dstPort = 3306 or dstPort = 5439)
| sort @timestamp desc
| limit 100
```

### Top talkers by data volume
```
fields @timestamp, srcAddr, dstAddr, bytes
| stats sum(bytes) as totalBytes by srcAddr, dstAddr
| sort totalBytes desc
| limit 20
```

### SSH connections to bastion
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter dstPort = 22 and action = "ACCEPT"
| sort @timestamp desc
| limit 50
```

### CloudTrail — security group changes (query CloudTrail log group or Athena)
```
fields @timestamp, userIdentity.arn, eventName, requestParameters.groupId
| filter eventName like /SecurityGroup/
| sort @timestamp desc
| limit 50
```

---

## 5. Alert Response Procedure

When an alarm fires:

1. **Acknowledge** the alarm within the response time for its severity
2. **Investigate** using the relevant logs and metrics
3. **Classify** — is this an incident (see [`INCIDENT-RESPONSE-PROCEDURE.md`](INCIDENT-RESPONSE-PROCEDURE.md)) or an operational issue?
4. **Resolve** the underlying cause
5. **Document** what happened and what was done
6. If the alarm was a false positive, tune the threshold and document the change

---

## Related Documents

- [SOC 2 Control Matrix](SOC2-CONTROL-MATRIX.md) — CC7.1 and CC7.2 control mapping
- [Incident Response Procedure](INCIDENT-RESPONSE-PROCEDURE.md) — When an alert indicates a security incident
- [Evidence Collection Guide](EVIDENCE-COLLECTION-GUIDE.md) — How to collect monitoring evidence for audit

---

## Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-03-17 | Infrastructure Team | Initial procedure created |
