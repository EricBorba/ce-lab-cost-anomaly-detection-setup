# Cost Anomaly Response Runbook

## Severity Levels

| Level | Impact | Response Time | Escalation |
|-------|--------|---------------|------------|
| P1 | > $100/day | 30 minutes | Team Lead + Manager |
| P2 | $50–100/day | 2 hours | Team Lead |
| P3 | $10–50/day | Next business day | Self-resolve |

---

## Step-by-Step Response

### 1. Acknowledge the Alert

- Reply to the SNS notification email
- Post in `#cloud-costs` Slack channel with anomaly ID and initial assessment

### 2. Identify the Root Cause

```bash
# Get recent anomalies (adjust date range)
aws ce get-anomalies \
  --date-interval '{"StartDate":"2026-06-01","EndDate":"2026-06-04"}' \
  --query 'Anomalies[].{ID: AnomalyId, Score: AnomalyScore, Impact: Impact.TotalImpact}'

# Check which service spiked
aws ce get-cost-and-usage \
  --time-period Start=2026-06-03,End=2026-06-04 \
  --granularity DAILY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE
```

### 3. Investigate Resources

```bash
# List recently launched EC2 instances
aws ec2 describe-instances \
  --filters "Name=launch-time,Values=2026-06-03*" \
  --query 'Reservations[].Instances[].{ID: InstanceId, Type: InstanceType, State: State.Name, LaunchTime: LaunchTime}'

# Check for large S3 operations via CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=s3.amazonaws.com \
  --start-time "2026-06-03T00:00:00Z" \
  --max-results 20
```

### 4. Take Corrective Action

- Terminate unused or accidental resources
- Revert configuration changes that caused the spike
- Apply resource tags for tracking and accountability
- If EC2: check for forgotten instances, stopped vs. terminated state

### 5. Post-Incident Documentation

- Record root cause in the incident log
- Update anomaly threshold if alert was a false positive:

```bash
# Update subscription threshold if needed
SUBSCRIPTION_ARN=$(aws ce get-anomaly-subscriptions \
  --query 'AnomalySubscriptions[?SubscriptionName==`all-services-immediate`].SubscriptionArn' \
  --output text)

aws ce update-anomaly-subscription \
  --subscription-arn "$SUBSCRIPTION_ARN" \
  --threshold-expression '{
    "Dimensions": {
      "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
      "Values": ["20"],
      "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
    }
  }'
```

- Share findings in the next team standup
- Update this runbook if new investigation patterns are identified

---

## Quick Reference — Investigation Commands

```bash
# List all anomaly monitors
aws ce get-anomaly-monitors \
  --query 'AnomalyMonitors[].{Name: MonitorName, Type: MonitorType}' \
  --output table

# List all subscriptions and their thresholds
aws ce get-anomaly-subscriptions \
  --query 'AnomalySubscriptions[].{Name: SubscriptionName, Frequency: Frequency}' \
  --output table

# Get anomaly history (last 90 days)
aws ce get-anomalies \
  --date-interval '{"StartDate":"2026-03-01","EndDate":"2026-06-04"}' \
  --query 'Anomalies[].{ID: AnomalyId, Start: AnomalyStartDate, Impact: Impact.TotalImpact, Score: AnomalyScore}' \
  --output table
```
