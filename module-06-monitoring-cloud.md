# Module 6: Monitoring: Azure Monitor & CloudWatch

> **Track:** Core Track
> **Duration:** 2–3 days
> **Level:** Beginner
> **Prerequisites:** Module 4A or Module 4B

---

## Table of Contents

1. [Why Monitoring Matters for Data Engineers](#1-why-monitoring-matters-for-data-engineers)
2. [Key Monitoring Concepts](#2-key-monitoring-concepts)
3. [Structured Logging in Python Pipelines](#3-structured-logging-in-python-pipelines)
4. [Azure Monitor and Log Analytics](#4-azure-monitor-and-log-analytics)
5. [AWS CloudWatch](#5-aws-cloudwatch)
6. [Setting Up Alerts](#6-setting-up-alerts)
7. [Introduction to Data Observability Tools](#7-introduction-to-data-observability-tools)
8. [Lab: Add Monitoring to an Existing Pipeline](#8-lab-add-monitoring-to-an-existing-pipeline)
9. [Quick Reference](#9-quick-reference)

---

## 1. Why Monitoring Matters for Data Engineers

Deploying a pipeline is not the end of the job. On client projects, data pipelines run continuously; daily ingestion jobs, hourly transformations, real-time streaming. Without monitoring, failures are discovered by the client before you are, which is the worst possible outcome.

Common scenarios that monitoring catches before the client does:
- An ingestion job failed silently; no data was loaded for two days
- A transformation is running 10x slower than usual because of data volume growth
- A dbt test is failing on a column that a business dashboard depends on
- An API the pipeline reads from started returning errors
- A pipeline ran successfully but produced zero rows (silent data failure)

Monitoring and alerting give your team visibility into the health of every pipeline, every run, every day; so you can fix problems proactively.

---

## 2. Key Monitoring Concepts

### SLA (Service Level Agreement)
An SLA defines the expected behavior of a pipeline: "the daily ingestion job must complete by 7:00 AM." Monitoring tracks whether SLAs are being met and alerts when they are not.

### Data Freshness
Data freshness measures how recent the data in a table is. If a table is expected to update every hour but the last update was six hours ago, that is a freshness violation.

### Error Rate
The percentage of pipeline runs that fail. A healthy pipeline should have an error rate close to zero. A spike in error rate signals a systemic problem.

### Latency
How long a pipeline takes to run. Tracking latency over time helps detect performance degradation before it becomes a failure.

### Observability vs Monitoring
**Monitoring** is watching predefined metrics (did the job succeed? how long did it take?).
**Observability** is the ability to understand the internal state of a system from its outputs (logs, metrics, traces). An observable system lets you answer questions you did not think to ask when you set up the monitoring.

---

## 3. Structured Logging in Python Pipelines

Before setting up monitoring infrastructure, make sure your pipelines emit useful logs. Unstructured logs (`print("done")`) are hard to search and parse. Structured logs (JSON format with consistent fields) are queryable, filterable, and alertable.

### 3.1 Basic Structured Logging

```python
import logging
import json
import time

# Configure logging to output JSON
class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
        }
        if hasattr(record, "extra"):
            log_entry.update(record.extra)
        return json.dumps(log_entry)

def get_logger(name: str) -> logging.Logger:
    logger = logging.getLogger(name)
    handler = logging.StreamHandler()
    handler.setFormatter(JsonFormatter())
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)
    return logger
```

### 3.2 Logging Pipeline Events

```python
logger = get_logger("ingestion-pipeline")

def run_ingestion(source: str, destination: str):
    start_time = time.time()

    logger.info("Pipeline started", extra={
        "source": source,
        "destination": destination,
        "run_id": "run-20240115-001"
    })

    try:
        record_count = extract_and_load(source, destination)

        duration = time.time() - start_time
        logger.info("Pipeline completed", extra={
            "source": source,
            "destination": destination,
            "record_count": record_count,
            "duration_seconds": round(duration, 2),
            "status": "success"
        })

    except Exception as e:
        duration = time.time() - start_time
        logger.error("Pipeline failed", extra={
            "source": source,
            "destination": destination,
            "duration_seconds": round(duration, 2),
            "status": "failed",
            "error": str(e)
        })
        raise
```

With JSON logs, you can query: "show me all runs with status=failed in the last 24 hours" or "what was the average record_count for this source last week?"

---

## 4. Azure Monitor and Log Analytics

Azure Monitor is the central observability platform for Azure. It collects metrics, logs, and traces from Azure resources and custom applications. Log Analytics is the query and storage engine underneath Azure Monitor.

### 4.1 Key Components

| Component | What it does |
|---|---|
| **Log Analytics Workspace** | Central store for all logs; queryable with KQL |
| **Azure Monitor Metrics** | Numeric time-series data (CPU, memory, request counts) |
| **Application Insights** | APM for applications; traces, exceptions, request tracking |
| **Diagnostic Settings** | Routes resource logs to Log Analytics |
| **Alerts** | Triggers notifications based on log queries or metric thresholds |
| **Workbooks** | Visual dashboards built on Log Analytics queries |

### 4.2 Sending Custom Logs from a Pipeline

Install the Azure Monitor Ingestion client:

```bash
pip install azure-monitor-ingestion azure-identity
```

```python
from azure.monitor.ingestion import LogsIngestionClient
from azure.identity import DefaultAzureCredential
import os
import json
from datetime import datetime, timezone

credential = DefaultAzureCredential()

client = LogsIngestionClient(
    endpoint=os.environ["AZURE_MONITOR_ENDPOINT"],
    credential=credential
)

def send_pipeline_log(status: str, record_count: int, duration: float):
    logs = [{
        "TimeGenerated": datetime.now(timezone.utc).isoformat(),
        "PipelineName": "daily-ingestion",
        "Status": status,
        "RecordCount": record_count,
        "DurationSeconds": duration
    }]

    client.upload(
        rule_id=os.environ["AZURE_DCR_RULE_ID"],
        stream_name=os.environ["AZURE_STREAM_NAME"],
        logs=logs
    )
```

### 4.3 Querying Logs with KQL

Kusto Query Language (KQL) is used to query Log Analytics. You do not need to master it; knowing the basics lets you build useful queries.

```kql
// Find all failed pipeline runs in the last 24 hours
PipelineLogs_CL
| where TimeGenerated > ago(24h)
| where Status == "failed"
| project TimeGenerated, PipelineName, DurationSeconds
| order by TimeGenerated desc

// Average duration per pipeline over the last 7 days
PipelineLogs_CL
| where TimeGenerated > ago(7d)
| where Status == "success"
| summarize avg(DurationSeconds) by PipelineName, bin(TimeGenerated, 1d)
| render timechart

// Count of runs per status per day
PipelineLogs_CL
| where TimeGenerated > ago(30d)
| summarize count() by Status, bin(TimeGenerated, 1d)
| render columnchart
```

---

## 5. AWS CloudWatch

CloudWatch is AWS's monitoring and observability service. It collects logs, metrics, and traces from AWS services and custom applications.

### 5.1 Key Components

| Component | What it does |
|---|---|
| **Log Groups** | Container for log streams; one per application or pipeline |
| **Log Streams** | Sequence of log events from a single source |
| **Metrics** | Numeric time-series data from AWS services or custom sources |
| **Alarms** | Trigger notifications when a metric crosses a threshold |
| **Dashboards** | Visual panels showing metrics and alarm states |
| **Insights** | Query log groups with CloudWatch Logs Insights query language |

### 5.2 Sending Custom Logs from a Pipeline

```python
import boto3
import json
import time
import os
from datetime import datetime

client = boto3.client("logs", region_name="ap-southeast-1")

LOG_GROUP = "/de-pipelines/daily-ingestion"
LOG_STREAM = f"run-{datetime.now().strftime('%Y%m%d-%H%M%S')}"

def setup_log_stream():
    try:
        client.create_log_group(logGroupName=LOG_GROUP)
    except client.exceptions.ResourceAlreadyExistsException:
        pass

    client.create_log_stream(
        logGroupName=LOG_GROUP,
        logStreamName=LOG_STREAM
    )

def send_log(message: dict):
    client.put_log_events(
        logGroupName=LOG_GROUP,
        logStreamName=LOG_STREAM,
        logEvents=[{
            "timestamp": int(time.time() * 1000),
            "message": json.dumps(message)
        }]
    )

setup_log_stream()
send_log({
    "status": "success",
    "record_count": 1250,
    "duration_seconds": 45.3,
    "pipeline": "daily-ingestion"
})
```

### 5.3 Querying with CloudWatch Logs Insights

```
# Find failed runs in the last 24 hours
fields @timestamp, status, pipeline, record_count
| filter status = "failed"
| sort @timestamp desc
| limit 50

# Average duration per pipeline
fields @timestamp, pipeline, duration_seconds
| filter status = "success"
| stats avg(duration_seconds) by pipeline

# Count runs by status per hour
fields @timestamp, status
| stats count() as run_count by status, bin(1h)
```

### 5.4 CloudWatch Metrics for AWS Services

AWS services emit metrics automatically to CloudWatch. For data engineering:

- **AWS Glue:** `glue.driver.aggregate.numFailedTasks`, job duration
- **AWS Lambda:** `Errors`, `Duration`, `Throttles`
- **S3:** `NumberOfObjects`, `BucketSizeBytes`
- **Redshift:** `DatabaseConnections`, `CPUUtilization`, `DiskSpaceUsedPercent`

---

## 6. Setting Up Alerts

### 6.1 Azure Monitor Alerts

**Log-based alert (triggers when a query returns results):**

1. Go to Azure Monitor; Alerts; Create Alert Rule
2. Set the scope (your Log Analytics workspace)
3. Set the condition: a KQL query that returns rows when something is wrong

```kql
// Alert fires when any pipeline fails
PipelineLogs_CL
| where TimeGenerated > ago(5m)
| where Status == "failed"
```

4. Set the threshold: alert if result count is greater than 0
5. Set the action group (who gets notified and how)

**Metric alert (triggers when a metric crosses a threshold):**
1. Create Alert Rule; condition type = Metric
2. Select the resource (e.g., a Function App or AKS cluster)
3. Select the metric (e.g., function execution failures)
4. Set threshold and evaluation window

### 6.2 AWS CloudWatch Alarms

**Metric alarm:**
```python
cloudwatch = boto3.client("cloudwatch", region_name="ap-southeast-1")

cloudwatch.put_metric_alarm(
    AlarmName="pipeline-failure-alert",
    ComparisonOperator="GreaterThanThreshold",
    EvaluationPeriods=1,
    MetricName="FailedRuns",
    Namespace="DEPipelines",
    Period=300,                   # 5 minutes
    Statistic="Sum",
    Threshold=0,
    ActionsEnabled=True,
    AlarmActions=["arn:aws:sns:ap-southeast-1:123456789:de-alerts"],
    AlarmDescription="Alert when any pipeline fails"
)
```

### 6.3 Sending Notifications to Slack

Both Azure and AWS support webhook-based alerts. Slack incoming webhooks receive a POST request and post a message to a channel.

**Python function to post to Slack:**
```python
import requests
import os

def notify_slack(message: str, status: str = "info"):
    webhook_url = os.environ["SLACK_WEBHOOK_URL"]

    color = {"success": "#36a64f", "failed": "#e01e5a", "info": "#0070d1"}.get(status, "#808080")

    payload = {
        "attachments": [{
            "color": color,
            "title": "Pipeline Notification",
            "text": message,
            "footer": "DE Monitoring"
        }]
    }

    response = requests.post(webhook_url, json=payload)
    response.raise_for_status()
```

**Use at the end of a pipeline:**
```python
try:
    run_pipeline()
    notify_slack(
        message="Daily ingestion completed: 1250 records loaded in 45s",
        status="success"
    )
except Exception as e:
    notify_slack(
        message=f"Daily ingestion FAILED: {str(e)}",
        status="failed"
    )
    raise
```

**In a pipeline YAML (always run even if previous steps fail):**

Azure Pipelines:
```yaml
- script: python notify_slack.py
  condition: always()
  env:
    SLACK_WEBHOOK_URL: $(slack-webhook-url)
    PIPELINE_STATUS: $(Agent.JobStatus)
  displayName: Send Slack notification
```

GitHub Actions:
```yaml
- name: Send Slack notification
  if: always()
  run: python notify_slack.py
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    PIPELINE_STATUS: ${{ job.status }}
```

---


## 7. Introduction to Data Observability Tools

Beyond infrastructure monitoring, a growing category of tools focuses specifically on the quality and reliability of data itself; not just "did the pipeline run" but "is the data correct."

### Great Expectations

Great Expectations is an open-source Python library for validating data. You define **expectations** (assertions about your data) and run them as a validation step in your pipeline.

```python
import great_expectations as ge

df = ge.read_csv("output.csv")

results = df.expect_column_values_to_not_be_null("customer_id")
results = df.expect_column_values_to_be_between("score", min_value=0, max_value=100)
results = df.expect_column_values_to_be_unique("transaction_id")

validation_result = df.validate()

if not validation_result.success:
    raise ValueError("Data validation failed; see results for details")
```

Great Expectations integrates with Azure Blob, S3, and most data warehouses for storing validation results and generating data docs (HTML reports).

### Monte Carlo

Monte Carlo is a commercial data observability platform. It monitors your data warehouse automatically; detecting anomalies in row counts, null rates, schema changes, and distribution shifts without you having to define every check manually.

> For this training, awareness of these tools is sufficient. Great Expectations is worth exploring hands-on if your client projects require data quality validation as part of the pipeline.

## 9. Lab: Add Monitoring to an Existing Pipeline

This lab adds monitoring, alerting, and a Slack notification to the pipeline built in Module 4A or 4B.

### Prerequisites
- A working pipeline from Module 4A or 4B
- A Slack workspace with permission to add an incoming webhook (or use a personal Slack)
- Azure Log Analytics workspace or AWS CloudWatch access

---

### Step 1: Create a Slack Webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Create a new app; choose **From scratch**
3. Go to **Incoming Webhooks; Activate Incoming Webhooks**
4. Click **Add New Webhook to Workspace**; choose a channel
5. Copy the webhook URL
6. Add it as a secret in your pipeline: `SLACK_WEBHOOK_URL`

---

### Step 2: Create a Notification Script

Create `notify.py` in your repo:

```python
import requests
import os
import sys

def notify_slack(status: str, details: str):
    webhook_url = os.environ["SLACK_WEBHOOK_URL"]
    pipeline_name = os.environ.get("PIPELINE_NAME", "unnamed-pipeline")

    color = "#36a64f" if status == "success" else "#e01e5a"
    emoji = "checkmark" if status == "success" else "x"

    payload = {
        "attachments": [{
            "color": color,
            "title": f"Pipeline: {pipeline_name}",
            "text": f"Status: *{status.upper()}*\n{details}",
            "footer": "DE Monitoring"
        }]
    }

    response = requests.post(webhook_url, json=payload)
    response.raise_for_status()
    print(f"Slack notification sent: {status}")

if __name__ == "__main__":
    status = sys.argv[1] if len(sys.argv) > 1 else "unknown"
    details = sys.argv[2] if len(sys.argv) > 2 else "No details provided"
    notify_slack(status, details)
```

---

### Step 3: Add Monitoring Steps to Your Pipeline

**Azure Pipelines:**
```yaml
variables:
  - group: de-practice-secrets   # Contains SLACK_WEBHOOK_URL
  - name: PIPELINE_NAME
    value: de-practice-pipeline

steps:
  - script: pip install -r requirements.txt
    displayName: Install dependencies

  - script: python src/main.py
    displayName: Run pipeline
    continueOnError: true        # Do not skip the notification step if this fails

  - script: |
      if [ "$(Agent.JobStatus)" = "Succeeded" ]; then
        python notify.py success "Pipeline completed successfully on build $(Build.BuildId)"
      else
        python notify.py failed "Pipeline failed on build $(Build.BuildId). Check logs at $(Build.BuildUri)"
      fi
    condition: always()
    env:
      SLACK_WEBHOOK_URL: $(SLACK_WEBHOOK_URL)
      PIPELINE_NAME: $(PIPELINE_NAME)
    displayName: Send Slack notification
```

**GitHub Actions:**
```yaml
env:
  PIPELINE_NAME: de-practice-pipeline

jobs:
  run-pipeline:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run pipeline
        id: run
        run: python src/main.py
        continue-on-error: true

      - name: Send success notification
        if: steps.run.outcome == 'success'
        run: python notify.py success "Pipeline completed successfully. Run ID ${{ github.run_id }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Send failure notification
        if: steps.run.outcome == 'failure'
        run: python notify.py failed "Pipeline FAILED. View run at ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Fail pipeline if run failed
        if: steps.run.outcome == 'failure'
        run: exit 1
```

---

### Step 4: Push and Trigger the Pipeline

Push your changes and trigger the pipeline. Verify:
- A Slack message appears in your channel when the pipeline succeeds
- Introduce a failure (e.g., bad Python syntax) and verify a failure notification is sent

---

### Step 5: Set Up a Log Query Alert (Optional)

**Azure:** If you have a Log Analytics workspace and your pipeline sends logs there, create an alert rule based on a KQL query that fires when a failure is logged (Section 6.1).

**AWS:** If your pipeline sends logs to CloudWatch, create a CloudWatch Alarm on a metric filter that counts error-level log entries (Section 6.2).

---


## 9. Quick Reference

| Tool | When to use |
|---|---|
| Azure Monitor / Log Analytics | Logs, metrics, and alerts for Azure resources and custom apps |
| AWS CloudWatch | Logs, metrics, and alarms for AWS services and custom apps |
| Great Expectations | Data quality validation within a pipeline |
| Monte Carlo | Automated data observability on a data warehouse (commercial) |
| Slack webhook | Sending notifications from pipelines to a channel |

| KQL pattern | What it does |
|---|---|
| `where TimeGenerated > ago(24h)` | Filter to last 24 hours |
| `where Status == "failed"` | Filter by field value |
| `summarize count() by Status` | Count by field |
| `summarize avg(Duration) by Pipeline` | Average by group |
| `render timechart` | Render as time series chart |

| CloudWatch Insights pattern | What it does |
|---|---|
| `filter status = "failed"` | Filter by field |
| `stats count() by status` | Count by field |
| `stats avg(duration) by pipeline` | Average by group |
| `sort @timestamp desc` | Sort newest first |

---

*Next module: Module 7: Containers for Data Engineers (Intermediate Extension)*
