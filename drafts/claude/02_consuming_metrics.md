# Consuming Metrics

## Overview

Once your services are exposing metrics and Prometheus is collecting them, the next step is **consuming** that data — turning raw numbers into actionable insights.

"Consuming" refers to the process of querying the collected metrics, visualizing them in dashboards, and setting up alerts so you're notified when something goes wrong.

---

## High-Level Flow

Here is the end-to-end journey of your metrics data, from your service to your screen:

```
Your Service
    │
    │  exposes metrics at /metrics endpoint
    ▼
Prometheus
    │
    │  periodically pulls metrics from your service
    │  stores them as time-series data
    ▼
Grafana (Data Source: Prometheus)
    │
    │  queries Prometheus using PromQL
    │  visualizes data in dashboards
    │  triggers alerts based on defined rules
    ▼
You (and your team)
```

### Step-by-Step

**1. Your service exposes metrics.**
Your service (or its exporter) makes metrics available at its `/metrics` HTTP endpoint. Each metric has a name, optional labels, and a current value.

**2. Prometheus collects the metrics.**
At regular intervals (typically every 15–30 seconds), Prometheus pulls the latest metric values from your service's `/metrics` endpoint and stores them with a timestamp. Over time, this builds up a rich historical record.

**3. Grafana connects to Prometheus as a data source.**
Grafana is configured to query Prometheus directly. Prometheus acts as Grafana's **data source** — the backend that Grafana sends queries to and receives data from.

**4. You query the data using PromQL.**
Grafana uses **PromQL** (Prometheus Query Language) to retrieve and transform the stored metrics. A PromQL query lets you filter by labels, calculate rates, aggregate across multiple services, and more. You write these queries inside Grafana panels.

**5. Grafana visualizes the results.**
The query results are displayed in **panels** — graphs, gauges, tables, and other visual formats — organized into **dashboards** that give you an at-a-glance view of your system's health.

**6. Alerts notify you of problems.**
You can define **alert rules** in Grafana that continuously evaluate your metrics and send notifications (via email, Slack, PagerDuty, etc.) when a condition is met — for example, when error rate exceeds 5% or when CPU usage is critically high.

---

## Key Components in the Consumption Layer

| Component | Role |
|---|---|
| **Prometheus** | Stores and serves the collected metrics |
| **PromQL** | The query language used to retrieve and compute metrics |
| **Grafana** | The visualization and alerting platform |
| **Data Source** | The configured connection between Grafana and Prometheus |
| **Dashboard** | A collection of panels displaying your metrics |
| **Alert Rule** | A condition that, when met, triggers a notification |

The following sections of this documentation cover **Grafana** in depth — how to set it up, build dashboards, write effective queries, and configure alerts.
