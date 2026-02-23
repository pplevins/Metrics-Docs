# Metrics Collection

## What Is Metrics Collection and How It Works

Metrics collection is the process of continuously gathering numerical data points from your services and infrastructure — such as request counts, error rates, memory usage, and response times — so you can understand how your systems behave over time.

Think of it like a health monitor for your application: instead of waiting for something to break, you proactively track key indicators and catch issues early.

The collection flow works like this:

1. Your service **exposes** metrics through a dedicated endpoint.
2. **Prometheus** periodically **pulls** those metrics from your service.
3. The collected data is **stored** in Prometheus and made available for querying and visualization in Grafana.

---

## Prometheus

**Prometheus** is an open-source monitoring system that collects and stores metrics as time-series data — meaning each data point is recorded with a timestamp. It is the backbone of the metrics collection pipeline.

Prometheus works by **pulling** (also called "scraping") metrics from your services at regular intervals. To do this, it needs to know:

- **Where** to collect from (the address of your service's metrics endpoint).
- **How often** to collect (the scrape interval, typically every 15 or 30 seconds).

This configuration is defined in Prometheus's `prometheus.yml` file, but when running in a Kubernetes environment, the **Prometheus Operator** (covered in the Advanced section) handles this for you in a more automated way.

> **Key concept — time-series data:** A time series is a sequence of values recorded at regular intervals over time. For example: "CPU usage at 10:00 = 32%, at 10:15 = 41%, at 10:30 = 38%." Prometheus stores all metrics this way, which allows you to analyze trends and detect anomalies.

---

## Collector

A **collector** is the component inside your service that gathers metric values and makes them available for Prometheus to pull.

When you instrument your service (i.e., add monitoring code to it), you create collectors for the things you want to measure. For example, a collector might track how many HTTP requests your service has handled, or how long each database query took.

Collectors are typically provided by a **Prometheus client library**, available in most common programming languages (Go, Java, Python, Node.js, etc.). You import the library, define what you want to measure, and the library handles the rest — including formatting the data correctly for Prometheus.

---

## Exporter

An **exporter** is a standalone service (or component within your service) that collects metrics from a system that doesn't natively speak Prometheus's format, and translates them into a format Prometheus can understand.

**Common use cases for exporters:**
- Collecting OS-level metrics (CPU, memory, disk) — handled by the **Node Exporter**.
- Collecting database metrics (e.g., PostgreSQL, MySQL).
- Collecting metrics from third-party tools that don't have built-in Prometheus support.

### How to Create a Collector Inside Your Exporter

To expose metrics from your service, you'll use a Prometheus client library. Below is a conceptual walkthrough of the steps:

**1. Import the Prometheus client library** for your language.

**2. Define a metric** — choose the type that fits what you're measuring (see metric types below), give it a name and a description.

**3. Instrument your code** — update the metric's value at the right places in your code (e.g., increment a counter each time a request is received).

**4. Expose the metrics endpoint** — the client library will automatically serve your metrics at a `/metrics` HTTP endpoint on your service.

**Example (Python):**

```python
from prometheus_client import Counter, start_http_server

# Define a counter metric
REQUEST_COUNT = Counter(
    'http_requests_total',          # Metric name
    'Total number of HTTP requests' # Description
)

# Increment the counter in your request handler
def handle_request():
    REQUEST_COUNT.inc()
    # ... rest of your handler logic

# Start the metrics server on port 8000
start_http_server(8000)
```

### How to Access Your Exporter

Once your service exposes the `/metrics` endpoint, you can verify it's working by visiting it directly in a browser or with `curl`:

```
http://<your-service-address>:<port>/metrics
```

You'll see a plain-text response listing all the metrics your service is currently exposing.

### Adding Metrics to Your Exporter

The four core metric types in Prometheus are:

| Type | What It Measures | Example |
|---|---|---|
| **Counter** | A value that only goes up (resets on restart) | Total requests, total errors |
| **Gauge** | A value that can go up or down | Current memory usage, active connections |
| **Histogram** | Distribution of values across configurable buckets | Request latency, response sizes |
| **Summary** | Similar to histogram, but calculates quantiles on the client side | p50, p95, p99 latency |

**When to use which:**
- Use a **Counter** for things you count (events, errors, requests).
- Use a **Gauge** for things you measure at a point in time (current queue size, current temperature).
- Use a **Histogram** when you care about distribution and want to calculate percentiles (e.g., "95% of requests complete in under 200ms").

### Reviewing the Metrics Your Exporter Is Exposing

After accessing the `/metrics` endpoint, you'll see output like this:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1027
http_requests_total{method="POST",status="500"} 3
```

Each metric entry includes:
- `# HELP` — a human-readable description of the metric.
- `# TYPE` — the metric type (counter, gauge, histogram, summary).
- The **metric name** followed by **labels** in `{}` and the **current value**.

---

## Labels

**Labels** are key-value pairs attached to a metric that allow you to differentiate between different dimensions of the same metric.

For example, the metric `http_requests_total` is more useful when you know *which* HTTP method and *which* status code each request had. Labels let you capture that:

```
http_requests_total{method="GET", status="200"} 1027
http_requests_total{method="POST", status="500"} 3
```

With labels, a single metric becomes a family of related time series — one for each unique combination of label values. This lets you slice and filter your data when querying in Grafana.

**Best practices for labels:**
- Keep the number of label values bounded. Avoid using labels with high cardinality values (e.g., user IDs or request IDs), as each unique combination creates a separate time series, which can overload Prometheus.
- Use labels for dimensions you'll actually filter or group by: `environment`, `region`, `service_name`, `method`, `status_code`.

---

## Golden Signals

The **Golden Signals** are four key metrics that Google's Site Reliability Engineering (SRE) practices define as the most important indicators of a service's health. Monitoring these four signals gives you a strong baseline understanding of how your service is performing.

| Signal | What It Tells You | Example Metric |
|---|---|---|
| **Latency** | How long it takes to serve a request | `http_request_duration_seconds` |
| **Traffic** | How much demand is being placed on your system | `http_requests_total` per second |
| **Errors** | The rate of failed requests | `http_requests_total{status=~"5.."}` |
| **Saturation** | How "full" your service is (CPU, memory, queue depth) | `process_cpu_seconds_total`, memory gauge |

When designing what to monitor in your service, start with the Golden Signals. They answer the most critical question: **"Is my service working correctly for my users?"**

---

# Advanced

## Types of Service Discovery

Instead of manually telling Prometheus the address of every service to collect from, **service discovery** allows Prometheus to automatically find your services as they come and go.

Prometheus supports several service discovery mechanisms:

| Type | How It Works | Best For |
|---|---|---|
| **Static configuration** | You manually list service addresses in the config file | Simple setups, fixed infrastructure |
| **Kubernetes SD** | Prometheus automatically discovers pods, services, and endpoints in your cluster | Kubernetes environments |
| **Consul SD** | Prometheus queries Consul (a service registry) to find services | Environments using Consul |
| **EC2 SD** | Prometheus discovers AWS EC2 instances automatically | AWS-based infrastructure |
| **File-based SD** | Prometheus reads a file (updated by another tool) to get service addresses | Custom or hybrid setups |

In a Kubernetes environment, **Kubernetes Service Discovery** is the standard approach. When combined with the Prometheus Operator (see below), Prometheus can automatically discover and collect from any service that has the right configuration applied — without any manual updates to the Prometheus config.

---

## Prometheus Operator

The **Prometheus Operator** is a tool for Kubernetes that simplifies deploying and managing Prometheus. Instead of editing Prometheus configuration files directly, you use Kubernetes-native resources to tell Prometheus what to collect from.

The two most important custom resources introduced by the Prometheus Operator are:

**ServiceMonitor** — defines which Kubernetes Services Prometheus should collect from, and on which port and path. You apply a `ServiceMonitor` to your service, and Prometheus automatically picks it up and starts collecting.

**PodMonitor** — similar to `ServiceMonitor`, but targets individual Pods rather than Services.

**Why this matters for you:** If your service is running on Kubernetes and the Prometheus Operator is installed, you don't need to touch any Prometheus configuration files. You simply apply a `ServiceMonitor` resource alongside your service, and Prometheus will automatically start collecting your metrics.

**Example ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-service-monitor
  labels:
    release: prometheus  # Must match the Prometheus Operator's selector
spec:
  selector:
    matchLabels:
      app: my-service    # Selects Services with this label
  endpoints:
    - port: metrics      # The port name on your Service that exposes /metrics
      interval: 30s      # How often Prometheus collects
```

---

## Collecting — OS vs. VM

Depending on where your service runs, the approach to collecting infrastructure-level metrics (CPU, memory, disk, network) differs:

**Bare Metal / Virtual Machine (VM):**
Deploy the **Node Exporter** directly on the machine. The Node Exporter is a Prometheus exporter that reads OS-level metrics from the host and exposes them on a `/metrics` endpoint. Prometheus then collects from it like any other exporter.

```
Node Exporter runs on: http://<machine-ip>:9100/metrics
```

**Kubernetes (Containerized):**
Run the Node Exporter as a **DaemonSet** — a Kubernetes resource that ensures one Node Exporter pod runs on every node in the cluster. This way, every node's infrastructure metrics are automatically collected without manual setup per machine.

The **kube-state-metrics** exporter is also commonly deployed in Kubernetes to expose cluster-level metrics (pod status, deployment health, resource requests/limits) that the Node Exporter doesn't cover.

**Summary:**

| Environment | What to Deploy | What It Collects |
|---|---|---|
| VM / Bare Metal | Node Exporter (as a service) | CPU, memory, disk, network per host |
| Kubernetes | Node Exporter (as DaemonSet) | Same, but per every cluster node |
| Kubernetes | kube-state-metrics | Pod status, deployment health, resource limits |
