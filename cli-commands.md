# CLI Commands — Lab M7.06: Cost Anomaly Detection Setup

All commands executed chronologically. Variables used throughout:
- `TOPIC_ARN` — ARN of the `cost-anomaly-alerts` SNS topic
- `ALL_MONITOR_ARN` — ARN of the DIMENSIONAL anomaly monitor
- `SUBSCRIPTION_ARN` — ARN of the `ec2-daily-digest` subscription

---

## Step 1: Create SNS Topic and Email Subscription

```bash
TOPIC_ARN=$(aws sns create-topic \
  --name cost-anomaly-alerts \
  --query 'TopicArn' --output text)
echo "Topic ARN: $TOPIC_ARN"

aws sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint eric.borba@gmail.com

aws sns list-subscriptions-by-topic --topic-arn "$TOPIC_ARN"
```

---

## Step 2: All-Services Anomaly Monitor

AWS allows only one DIMENSIONAL monitor per account. An existing monitor was found and reused:

```bash
# Attempt to create (failed — limit of one DIMENSIONAL monitor per account)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "all-services-monitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'
# Error: Limit exceeded on dimensional spend monitor creation

# Retrieved existing monitor instead
ALL_MONITOR_ARN=$(aws ce get-anomaly-monitors \
  --query 'AnomalyMonitors[?MonitorType==`DIMENSIONAL`].MonitorArn' \
  --output text)
echo "All-Services Monitor ARN: $ALL_MONITOR_ARN"

aws ce get-anomaly-monitors \
  --query 'AnomalyMonitors[].{Name: MonitorName, Type: MonitorType, ARN: MonitorArn}' \
  --output table
```

---

## Step 3: EC2-Scoped Monitor — Attempted, Not Possible

Two approaches were tried and both failed due to AWS single-account limitations:

```bash
# Attempt 1 — SERVICE dimension in CUSTOM monitor (not supported)
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

# Attempt 2 — LINKED_ACCOUNT with own account ID
# Would create a monitor with identical scope to the DIMENSIONAL monitor.
# Only meaningful in AWS Organizations (management + linked member accounts).
# Skipped — no additional value in a single-account setup.
```

**Resolution:** Both subscriptions use the existing DIMENSIONAL monitor. EC2 anomalies are still detected and labeled in notification payloads, but subscriptions cannot be filtered to EC2 only without a Lambda filter layer.

---

## Step 4: Create Anomaly Subscriptions

```bash
# Subscription 1 — Immediate, $10 threshold, SNS
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "all-services-immediate",
    "MonitorArnList": ["'"$ALL_MONITOR_ARN"'"],
    "Subscribers": [{"Address": "'"$TOPIC_ARN"'", "Type": "SNS"}],
    "Frequency": "IMMEDIATE",
    "ThresholdExpression": {
      "Dimensions": {
        "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
        "Values": ["10"],
        "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
      }
    }
  }'

# Subscription 2 — First attempt with SNS (failed for DAILY frequency)
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "ec2-daily-digest",
    "MonitorArnList": ["'"$ALL_MONITOR_ARN"'"],
    "Subscribers": [{"Address": "'"$TOPIC_ARN"'", "Type": "SNS"}],
    "Frequency": "DAILY",
    "ThresholdExpression": {
      "Dimensions": {
        "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
        "Values": ["5"],
        "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
      }
    }
  }'
# Error: Daily or weekly frequencies only support Email subscriptions

# Subscription 2 — Fixed with direct email
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "ec2-daily-digest",
    "MonitorArnList": ["'"$ALL_MONITOR_ARN"'"],
    "Subscribers": [{"Address": "eric.borba@gmail.com", "Type": "EMAIL"}],
    "Frequency": "DAILY",
    "ThresholdExpression": {
      "Dimensions": {
        "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
        "Values": ["5"],
        "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
      }
    }
  }'

aws ce get-anomaly-subscriptions \
  --query 'AnomalySubscriptions[].{Name: SubscriptionName, Frequency: Frequency}' \
  --output table
```

---

## Step 5: Demonstrate Frequency Update

```bash
SUBSCRIPTION_ARN=$(aws ce get-anomaly-subscriptions \
  --query 'AnomalySubscriptions[?SubscriptionName==`ec2-daily-digest`].SubscriptionArn' \
  --output text)
echo "Subscription ARN: $SUBSCRIPTION_ARN"

aws ce update-anomaly-subscription \
  --subscription-arn "$SUBSCRIPTION_ARN" \
  --frequency "WEEKLY"
echo "Changed to WEEKLY"

aws ce update-anomaly-subscription \
  --subscription-arn "$SUBSCRIPTION_ARN" \
  --frequency "DAILY"
echo "Changed back to DAILY"
```
