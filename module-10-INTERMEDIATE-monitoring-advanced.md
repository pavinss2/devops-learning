# Module 10: Advanced Monitoring: Prometheus & Grafana

> **Track:** Intermediate Extension
> **Duration:** 2–3 days
> **Level:** Intermediate
> **Prerequisites:** Module 6: Monitoring: Azure Monitor & CloudWatch, Module 7: Containers for Data Engineers

---

## Table of Contents

1. [Why Prometheus and Grafana](#1-why-prometheus-and-grafana)
2. [Prometheus Fundamentals](#2-prometheus-fundamentals)
3. [Instrumenting a Python Pipeline with Prometheus](#3-instrumenting-a-python-pipeline-with-prometheus)
4. [Running Prometheus Locally](#4-running-prometheus-locally)
5. [Grafana Fundamentals](#5-grafana-fundamentals)
6. [Running Grafana Locally](#6-running-grafana-locally)
7. [Connecting Grafana to Prometheus](#7-connecting-grafana-to-prometheus)
8. [Connecting Grafana to Azure Monitor and CloudWatch](#8-connecting-grafana-to-azure-monitor-and-cloudwatch)
9. [Building a Pipeline Health Dashboard](#9-building-a-pipeline-health-dashboard)
10. [Grafana Alerting](#10-grafana-alerting)
11. [Lab: Instrument a Pipeline and Build a Health Dashboard](#11-lab-instrument-a-pipeline-and-build-a-health-dashboard)
12. [Quick Reference](#12-quick-reference)

---

## 1. Why Prometheus and Grafana

In Module 6 you learned the cloud-native monitoring tools; Azure Monitor and CloudWatch. These are excellent for monitoring resources within a single cloud. However on more complex engagements you may encounter situations where:

- The client runs workloads across **both Azure and AWS** and wants a unified view
- The client uses **Kubernetes** (Module 8) and wants infrastructure-level metrics alongside application metrics
- The client has **self-hosted or on-premises** components that Azure Monitor and CloudWatch cannot reach
- The team wants **richer dashboards and custom alerting** beyond what the cloud-native tools provide out of the box

Prometheus and Grafana are the industry standard open-source answer to these situations. They are platform-agnostic, free, and widely adopted across enterprise environments. Understanding them makes you a stronger consultant on complex multi-cloud or Kubernetes-based engagements.

**How they relate to Module 6:**
Prometheus and Grafana do not replace Azure Monitor or CloudWatch; they complement them. Grafana can connect to Azure Monitor, CloudWatch, and Prometheus simultaneously, giving you a single dashboard that pulls from all sources.

---

## 2. Prometheus Fundamentals

### 2.1 What Prometheus Does

Prometheus is a monitoring system that collects **metrics**; numeric measurements that change over time. Unlike logs (which are text events), metrics are numbers: how many requests per second, how long a function took, how many records were processed.

Prometheus works by **scraping**; it periodically calls an HTTP endpoint on your application (typically `/metrics`) and records the current values of all exposed metrics. It stores these time-series values in its own database and makes them queryable with PromQL.

### 2.2 The Prometheus Data Model

Every metric in Prometheus has:
- A **name** (e.g., `pipeline_records_processed_total`)
- A **value** (a number)
- **Labels** (key-value pairs for filtering, e.g., `{pipeline="ingestion", status="success"}`)
- A **timestamp**

### 2.3 Metric Types

| Type | What it measures | Example |
|---|---|---|
| **Counter** | A value that only increases; resets on restart | Total records processed; total errors |
| **Gauge** | A value that goes up and down | Current queue size; memory usage |
| **Histogram** | Distribution of values in configurable buckets | Request duration; pipeline run time |
| **Summary** | Similar to histogram but calculates quantiles client-side | p50, p95, p99 latency |

For data engineering pipelines, you will mostly use **Counters** (total runs, total records, total errors) and **Histograms** (pipeline duration).

### 2.4 PromQL Basics

PromQL (Prometheus Query Language) is used to query and aggregate metrics. You do not need to master it; the basics get you far.

```promql
# Current value of a counter
pipeline_records_processed_total

# Filter by label
pipeline_records_processed_total{pipeline="ingestion", status="success"}

# Rate of increase per second over the last 5 minutes (for counters)
rate(pipeline_records_processed_total[5m])

# Total records processed in the last hour
increase(pipeline_records_processed_total[1h])

# Average pipeline duration
histogram_quantile(0.95, rate(pipeline_duration_seconds_bucket[5m]))

# Count of failed runs per pipeline
sum by (pipeline) (pipeline_runs_total{status="failed"})
```

---

## 3. Instrumenting a Python Pipeline with Prometheus

The `prometheus_client` library lets you expose metrics from any Python application.

### 3.1 Install

```bash
pip install prometheus_client
```

### 3.2 Define and Expose Metrics

```python
# metrics.py
from prometheus_client import Counter, Histogram, start_http_server
import time

# Define metrics
PIPELINE_RUNS = Counter(
    "pipeline_runs_total",
    "Total number of pipeline runs",
    ["pipeline", "status"]   # Labels
)

RECORDS_PROCESSED = Counter(
    "pipeline_records_processed_total",
    "Total records processed",
    ["pipeline"]
)

PIPELINE_DURATION = Histogram(
    "pipeline_duration_seconds",
    "Pipeline run duration in seconds",
    ["pipeline"],
    buckets=[10, 30, 60, 120, 300, 600]   # Custom bucket boundaries
)

def start_metrics_server(port: int = 8000):
    """Start the HTTP server that exposes metrics to Prometheus."""
    start_http_server(port)
    print(f"Metrics available at http://localhost:{port}/metrics")
```

### 3.3 Use Metrics in Your Pipeline

```python
# pipeline.py
import time
from metrics import PIPELINE_RUNS, RECORDS_PROCESSED, PIPELINE_DURATION, start_metrics_server

def run_ingestion(pipeline_name: str = "ingestion"):
    start = time.time()

    try:
        # Simulate processing
        records = [{"id": i} for i in range(1000)]
        time.sleep(2)   # Simulate work

        # Record success metrics
        RECORDS_PROCESSED.labels(pipeline=pipeline_name).inc(len(records))
        PIPELINE_RUNS.labels(pipeline=pipeline_name, status="success").inc()

        duration = time.time() - start
        PIPELINE_DURATION.labels(pipeline=pipeline_name).observe(duration)

        print(f"Processed {len(records)} records in {duration:.2f}s")

    except Exception as e:
        PIPELINE_RUNS.labels(pipeline=pipeline_name, status="failed").inc()
        raise

if __name__ == "__main__":
    start_metrics_server(port=8000)
    while True:
        run_ingestion()
        time.sleep(30)   # Run every 30 seconds
```

### 3.4 Viewing the Metrics Endpoint

Run the script and open [http://localhost:8000/metrics](http://localhost:8000/metrics) in your browser. You will see all metrics in Prometheus text format:

```
# HELP pipeline_runs_total Total number of pipeline runs
# TYPE pipeline_runs_total counter
pipeline_runs_total{pipeline="ingestion",status="success"} 5.0
pipeline_runs_total{pipeline="ingestion",status="failed"} 1.0

# HELP pipeline_records_processed_total Total records processed
# TYPE pipeline_records_processed_total counter
pipeline_records_processed_total{pipeline="ingestion"} 5000.0
```

This endpoint is what Prometheus scrapes.

---

## 4. Running Prometheus Locally

The easiest way to run Prometheus locally is with Podman (or Docker on client environments).

### 4.1 Prometheus Configuration File

Create `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s       # Scrape every 15 seconds

scrape_configs:
  - job_name: "de-pipeline"
    static_configs:
      - targets: ["host.containers.internal:8000"]
        # host.containers.internal resolves to your host machine from inside a container
        # Docker equivalent: host.docker.internal:8000
```

### 4.2 Run Prometheus

```powershell
# Podman
podman run -d `
  --name prometheus `
  -p 9090:9090 `
  -v ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml `
  prom/prometheus

# Docker equivalent
# docker run -d --name prometheus -p 9090:9090 `
#   -v ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml `
#   prom/prometheus
```

Open [http://localhost:9090](http://localhost:9090) to access the Prometheus UI. Go to **Status; Targets** to confirm it is scraping your pipeline successfully.

Try a PromQL query in the Prometheus UI:

```promql
pipeline_runs_total
```

---

## 5. Grafana Fundamentals

Grafana is a visualization platform that connects to many data sources and renders interactive dashboards. Key concepts:

| Concept | Description |
|---|---|
| **Data source** | A connection to a metrics or log store; Prometheus, Azure Monitor, CloudWatch, etc. |
| **Panel** | A single visualization; a chart, gauge, table, or stat |
| **Dashboard** | A collection of panels arranged on a page |
| **Alert rule** | A condition evaluated against a data source that triggers a notification |
| **Contact point** | Where alert notifications are sent; Slack, email, PagerDuty, etc. |

---

## 6. Running Grafana Locally

```powershell
# Podman
podman run -d `
  --name grafana `
  -p 3000:3000 `
  grafana/grafana:latest

# Docker equivalent
# docker run -d --name grafana -p 3000:3000 grafana/grafana:latest
```

Open [http://localhost:3000](http://localhost:3000). Default credentials: `admin` / `admin`. You will be prompted to change the password on first login.

### Running Prometheus and Grafana Together with Podman Compose

For a more convenient local setup, use a `compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

```powershell
podman-compose up -d
# docker compose up -d
```

---

## 7. Connecting Grafana to Prometheus

1. In Grafana, go to **Connections; Data Sources; Add data source**
2. Select **Prometheus**
3. Set the URL to `http://prometheus:9090` (if using Compose) or `http://host.containers.internal:9090` (if running separately)
4. Click **Save & Test**; you should see a green success message

You can now build panels using PromQL queries against your pipeline metrics.

---

## 8. Connecting Grafana to Azure Monitor and CloudWatch

### 8.1 Azure Monitor

1. Add a new data source; search **Azure Monitor**
2. Fill in:
   - **Directory (tenant) ID:** Your Azure tenant ID
   - **Application (client) ID:** A service principal client ID
   - **Client Secret:** The service principal's secret
3. Click **Save & Test**

Once connected, build panels that query Azure Monitor Metrics or Log Analytics (using KQL queries) alongside your Prometheus panels.

### 8.2 CloudWatch

1. Add a new data source; search **CloudWatch**
2. Set **Authentication Provider** to **Access & secret key**
3. Enter your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`
4. Set your default region (e.g., `ap-southeast-1`)
5. Click **Save & Test**

Once connected, browse CloudWatch metric namespaces and build panels for Glue jobs, Lambda functions, Redshift, and your custom pipeline metrics.

### 8.3 The Unified View

With Prometheus, Azure Monitor, and CloudWatch all connected as data sources, a single Grafana dashboard can show:

- Custom pipeline metrics from Prometheus (records processed, run duration, error counts)
- Azure Monitor metrics (Databricks job status, ADF pipeline runs)
- CloudWatch metrics (Glue job errors, Lambda invocations)

This cross-cloud unified view is the main reason to use Grafana over cloud-native dashboards on multi-cloud client engagements.

---

## 9. Building a Pipeline Health Dashboard

A practical pipeline health dashboard has these panels:

| Panel | Type | Data source | Query |
|---|---|---|---|
| Total runs today | Stat | Prometheus | `increase(pipeline_runs_total[24h])` |
| Failed runs today | Stat | Prometheus | `increase(pipeline_runs_total{status="failed"}[24h])` |
| Success rate | Gauge | Prometheus | `rate(pipeline_runs_total{status="success"}[1h]) / rate(pipeline_runs_total[1h]) * 100` |
| Records processed (trend) | Time series | Prometheus | `rate(pipeline_records_processed_total[5m])` |
| Pipeline duration (p95) | Time series | Prometheus | `histogram_quantile(0.95, rate(pipeline_duration_seconds_bucket[5m]))` |
| Failed runs table | Table | CloudWatch / Log Analytics | Query for failed run log entries |

### Creating a Panel in Grafana

1. Open your dashboard; click **Add; Visualization**
2. Select your data source (e.g., Prometheus)
3. Enter your PromQL query in the query editor
4. Choose the visualization type (Time series, Stat, Gauge, Table)
5. Configure the title, units, and thresholds
6. Click **Apply**

---

## 10. Grafana Alerting

Grafana's alerting system evaluates queries against any connected data source and sends notifications when conditions are met.

### 10.1 Creating an Alert Rule

1. Go to **Alerting; Alert rules; New alert rule**
2. Set the data source and query (e.g., `increase(pipeline_runs_total{status="failed"}[5m]) > 0`)
3. Set the condition: **IS ABOVE 0**
4. Set the evaluation interval: every 1 minute
5. Add labels and annotations (e.g., `summary="Pipeline failure detected"`)
6. Assign to a contact point

### 10.2 Setting Up a Slack Contact Point

1. Go to **Alerting; Contact points; Add contact point**
2. Select **Slack**
3. Paste your Slack webhook URL (same one from Module 6)
4. Save and test

### 10.3 Notification Policies

Notification policies route alerts to the right contact point based on labels. For example, route alerts labelled `severity=critical` to PagerDuty and `severity=warning` to Slack.

---

## 11. Lab: Instrument a Pipeline and Build a Health Dashboard

This lab instruments a Python pipeline with Prometheus metrics, runs Prometheus and Grafana locally using Podman Compose, and builds a dashboard showing pipeline health with an alert that fires on simulated failure.

### Prerequisites
- Podman Desktop installed (Module 7)
- Podman Compose installed (`pip install podman-compose`)
- A Slack webhook URL (from Module 6 lab)

---

### Step 1: Create the Project

Create a folder called `monitoring-practice`. Inside it create the following files:

`requirements.txt`:
```
prometheus_client
```

`metrics.py`: (copy from Section 3.2)

`pipeline.py`: (copy from Section 3.3)

`prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "de-pipeline"
    static_configs:
      - targets: ["host.containers.internal:8000"]
```

`compose.yml`:
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

---

### Step 2: Run the Pipeline

Install dependencies and start the pipeline:

```powershell
pip install -r requirements.txt
python pipeline.py
```

Verify the metrics endpoint is live: open [http://localhost:8000/metrics](http://localhost:8000/metrics) in your browser.

---

### Step 3: Start Prometheus and Grafana

In a new terminal:

```powershell
cd monitoring-practice
podman-compose up -d
```

Open [http://localhost:9090](http://localhost:9090). Go to **Status; Targets** and confirm `de-pipeline` shows as UP.

Run a PromQL query to verify data is coming in:

```promql
pipeline_runs_total
```

---

### Step 4: Connect Grafana to Prometheus

1. Open [http://localhost:3000](http://localhost:3000); log in with `admin` / `admin`
2. Go to **Connections; Data Sources; Add data source**
3. Select **Prometheus**; set URL to `http://prometheus:9090`
4. Click **Save & Test**

---

### Step 5: Build the Dashboard

Create a new dashboard with three panels:

**Panel 1: Total successful runs (Stat)**
```promql
increase(pipeline_runs_total{status="success"}[1h])
```

**Panel 2: Total failed runs (Stat)**
```promql
increase(pipeline_runs_total{status="failed"}[1h])
```

**Panel 3: Records processed over time (Time series)**
```promql
rate(pipeline_records_processed_total[1m]) * 60
```

Save the dashboard with the name "Pipeline Health".

---

### Step 6: Create a Failure Alert

1. Go to **Alerting; Contact points**; add a Slack contact point with your webhook URL
2. Go to **Alerting; Alert rules; New alert rule**
3. Query: `increase(pipeline_runs_total{status="failed"}[5m]) > 0`
4. Condition: IS ABOVE 0
5. Evaluation: every 1 minute
6. Contact point: your Slack contact point
7. Save the rule

---

### Step 7: Simulate a Failure and Observe the Alert

In `pipeline.py`, temporarily force a failure by raising an exception inside `run_ingestion`. Wait for the next run cycle (30 seconds). Observe:

- The failed runs stat panel updates in Grafana
- A Slack notification arrives from Grafana

Revert the change and confirm the alert resolves.

---

### Step 8: Clean Up

```powershell
podman-compose down
```

Stop the pipeline script with `Ctrl+C`.

---

## 12. Quick Reference

| Concept | Description |
|---|---|
| Prometheus scrape | HTTP GET to `/metrics` endpoint every N seconds |
| Counter | Metric that only increases; use for totals |
| Gauge | Metric that goes up and down; use for current state |
| Histogram | Distribution of values; use for durations |
| PromQL `rate()` | Per-second rate of increase over a time window |
| PromQL `increase()` | Total increase over a time window |
| PromQL `sum by()` | Aggregate across label values |
| Grafana data source | Connection to a metrics or log store |
| Grafana panel | Single visualization on a dashboard |
| Grafana alert rule | Condition that triggers a notification |
| Grafana contact point | Destination for alert notifications |

| Command | What it does |
|---|---|
| `podman run -d prom/prometheus` | Start Prometheus container |
| `podman run -d grafana/grafana` | Start Grafana container |
| `podman-compose up -d` | Start all services from compose.yml |
| `podman-compose down` | Stop all services |

---

*You have completed the full DevOps Training Track for Data Engineers.*
