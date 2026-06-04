# Cost Anomaly Alert Flow

## Detection Pipeline

```
AWS Cost Anomaly Detection (ML model)
  │
  ├── Analyzes daily spending patterns per service
  ├── Compares current spending against historical baseline
  ├── When deviation exceeds learned threshold → anomaly created
  │
  └── Anomaly impact >= subscription threshold?
        │
        ├── YES (IMMEDIATE, $10+) → SNS → Email (eric.borba@gmail.com)
        └── YES (DAILY digest, $5+) → Direct Email (eric.borba@gmail.com)
```

## Monitors Configured

| Monitor | Type | Scope | ML Model |
|---------|------|-------|----------|
| All Services Cost Monitor | DIMENSIONAL | All AWS services | Separate model per service |

### Note: EC2-Scoped Monitor — Attempted but Not Possible

The lab called for a CUSTOM monitor scoped to EC2. The following was attempted:

```bash
# Attempt 1 — SERVICE dimension (failed)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "ec2-cost-monitor",
    "MonitorType": "CUSTOM",
    "MonitorSpecification": {
      "Dimensions": {
        "Key": "SERVICE",
        "Values": ["Amazon Elastic Compute Cloud - Compute"],
        "MatchOptions": ["EQUALS"]
      }
    }
  }'
# Error: Dimension not valid; must be scoped to 'LINKED_ACCOUNT'

# Attempt 2 — LINKED_ACCOUNT with own account ID (not meaningful)
# AWS CUSTOM monitors require LINKED_ACCOUNT dimension, which is designed
# for AWS Organizations (management account + member accounts).
# Using your own account ID would create a monitor with identical scope
# to the existing DIMENSIONAL monitor — no additional value.
```

**Why it doesn't work in a single account:** CUSTOM monitors in Cost Anomaly Detection are designed for AWS Organizations to isolate spending per member (linked) account. In a single-account setup without Organizations, the only valid dimension for CUSTOM monitors is `LINKED_ACCOUNT`, but scoping to your own account ID is functionally identical to the DIMENSIONAL monitor.

**EC2 anomalies are still detected:** The DIMENSIONAL monitor already builds a separate ML model for each service, including EC2. When an EC2 anomaly fires, the notification payload identifies EC2 as the root cause — the limitation is that you cannot create a subscription that *only* fires for EC2 anomalies without a Lambda filter layer.

## Subscriptions Configured

| Subscription | Frequency | Threshold | Subscriber | Monitor |
|-------------|-----------|-----------|------------|---------|
| all-services-immediate | IMMEDIATE | $10+ | SNS → Email | All Services |
| ec2-daily-digest | DAILY | $5+ | Direct Email | All Services |

### Frequency Constraint Discovered

AWS Cost Anomaly Detection only supports SNS as a subscriber for `IMMEDIATE` frequency. `DAILY` and `WEEKLY` frequencies require a direct email address — they cannot publish to an SNS topic.

```
IMMEDIATE → SNS topic supported ✓
DAILY     → Email address only  ✓ (SNS blocked by AWS)
WEEKLY    → Email address only  ✓ (SNS blocked by AWS)
```

## Sample Anomaly Notification Fields

- **Anomaly ID**: Unique identifier for the detected anomaly
- **Monitor Name**: Which monitor detected it
- **Root Cause**: Service and usage type driving the cost increase
- **Impact**: Estimated dollar amount above expected spending
- **Start Date**: When the anomaly began
- **Anomaly Score**: Confidence level of the detection (0–100)

## Alert Frequency Comparison

| Frequency | Behavior | Best For |
|-----------|----------|----------|
| IMMEDIATE | Alert sent as soon as anomaly is detected | Critical cost spikes (EC2, RDS) |
| DAILY | Single digest summarizing the day's anomalies | Routine monitoring, lower-priority services |
| WEEKLY | Weekly summary of all detected anomalies | Informational, low-spend environments |

**Production recommendation:** Use IMMEDIATE for high-cost services (EC2, RDS, data transfer) and DAILY for everything else. WEEKLY is suitable only for low-spend sandbox accounts.
