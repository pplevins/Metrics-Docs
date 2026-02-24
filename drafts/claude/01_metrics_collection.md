# Metrics Collection

## What Is Metrics Collection and How It Works

Metrics collection is the process of continuously gathering numerical data points from your services and infrastructure - such as request counts, error rates, memory usage, and response times - so you can understand how your systems behave over time.

Think of it like a health monitor for your application: instead of waiting for something to break, you proactively track key indicators and catch issues early.

The collection flow works like this:

1. Your service **exposes** metrics through a dedicated endpoint.
2. **Prometheus** periodically **pulls** those metrics from your service.
3. The collected data is **stored** in Prometheus and made available for querying and visualization in **Grafana**.

The following sections cover the key components and concepts behind metrics collection - from exposing metrics in your service all the way to making them available for dashboards and alerts.

---

## Prometheus

**Prometheus** is an open-source monitoring system that collects and stores metrics as time-series data - meaning each data point is recorded with a timestamp. It is the backbone of the metrics collection pipeline.

Prometheus works Usually by **pulling** (also called "scraping") metrics from your services at regular intervals. To do this, it needs to know:

- **Where** to collect from (the address of your service's metrics endpoint).
- **How often** to collect (the scrape interval, typically every 15 or 30 seconds).

This configuration is defined in Prometheus's `prometheus.yml` file, but when running in a Kubernetes environment, the **Prometheus Operator** (covered in the Advanced section) handles this for you in a more automated and Kubernetes-native way.

> **Key concept - time-series data:** A time series is a sequence of values recorded at regular intervals over time. For example: "CPU usage at 10:00 = 32%, at 10:15 = 41%, at 10:30 = 38%." Prometheus stores all metrics this way, which allows you to analyze trends and detect anomalies.

---

## Exporter

An **exporter** is a standalone service (or component within your service) that collects metrics from a system that doesn't natively speak Prometheus's format, and translates them into a format Prometheus can understand.

**Common use cases for exporters:**
- Collecting OS-level metrics (CPU, memory, disk) - handled by the **Node Exporter**.
- Collecting database metrics (e.g., PostgreSQL, MySQL).
- Collecting metrics from third-party tools that don't have built-in Prometheus support.
### Common Types of Exporters

There are many exporters available, and choosing the right one depends on what you need to monitor and how much control you have over the target system.

**Node Exporter** is the most widely used exporter and is the standard way to collect OS-level hardware and kernel metrics from Linux hosts - CPU usage, memory, disk I/O, network throughput, and more. It runs as a lightweight daemon alongside your workload and exposes a `/metrics` endpoint that Prometheus scrapes.

**Blackbox Exporter** takes a different approach: instead of instrumenting a system from the inside, it probes it from the outside. You configure it to perform HTTP, HTTPS, DNS, TCP, or ICMP checks against a target, and it reports on availability, response time, and status codes. This makes it particularly valuable for monitoring closed or third-party systems where you have no access to the internals - if you can reach it over the network, you can monitor it.

**Database exporters** like the PostgreSQL Exporter (`postgres_exporter`) or MySQL Exporter (`mysqld_exporter`) connect to a database instance, query internal statistics tables, and expose those as Prometheus metrics. These are good examples of the broader category of official or community-maintained exporters that exist for most popular infrastructure components - message queues, caches, web servers, cloud services, and more.

**When deciding how to expose metrics, you generally have three options:**

- **Use an official or community exporter.** For well-known systems (databases, web servers, Kubernetes, cloud infrastructure), a maintained exporter almost certainly already exists. This is the lowest-effort path and should be your starting point.
- **Build your own exporter.** When you're working with a proprietary system, an internal tool, or a service with no existing exporter, you can build one using a Prometheus client library. This gives you full control over what's measured and how it's labeled.
- **Use only the Blackbox Exporter.** For fully closed systems where you can't instrument the internals or install a sidecar, the Blackbox Exporter gives you external observability with no access required. You won't get deep internal metrics, but you'll know whether the system is reachable and responding correctly.
### How to Add Custom Metrics to Your Exporter

If you're building your own exporter - or adding Prometheus instrumentation directly into your service - you'll use a **Prometheus client library** to define and expose metrics from within your code. Below is a conceptual walkthrough of the steps:

**1. Import the Prometheus client library** for your language.

**2. Define a metric** - choose the type that fits what you're measuring (see metric types below), give it a name and a description.

**3. Instrument your code** - update the metric's value at the right places in your code (e.g., increment a counter each time a request is received).

**4. Expose the metrics endpoint** - the client library will automatically serve your metrics at a `/metrics` HTTP endpoint on your service.

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

| Type          | What It Measures                                                  | Example                                  |
| ------------- | ----------------------------------------------------------------- | ---------------------------------------- |
| **Counter**   | A value that only goes up (resets on restart)                     | Total requests, total errors             |
| **Gauge**     | A value that can go up or down                                    | Current memory usage, active connections |
| **Histogram** | Distribution of values across configurable buckets                | Request latency, response sizes          |
| **Summary**   | Similar to histogram, but calculates quantiles on the client side | p50, p95, p99 latency                    |

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
- `# HELP` - a human-readable description of the metric.
- `# TYPE` - the metric type (counter, gauge, histogram, summary).
- The **metric name** followed by **labels** in `{}` and the **current value**.

---
## Collector

The **OpenTelemetry Collector** is a vendor-neutral, standalone service that acts as a central hub for receiving, processing, and forwarding observability data - including metrics, logs, and traces.

Instead of having each of your services send data directly to Prometheus, you can route telemetry through the Collector, which handles the translation, filtering, and forwarding to your backend of choice. A key distinction from the exporter model is that the Collector typically operates on a **push-based** model: your services actively push metrics to the Collector, which then forwards them onward to Prometheus. This is the opposite of Prometheus's native **pull-based** model, where Prometheus scrapes each target's `/metrics` endpoint on its own schedule. The Collector is part of the **OpenTelemetry** project - an open standard for instrumenting and collecting observability data across different languages and platforms.

The Collector works in a pipeline with three stages:

- **Receivers** - accept incoming data from your services or other sources (e.g., in Prometheus format, OTLP, StatsD).
- **Processors** - transform or filter the data (e.g., add labels, drop unwanted metrics, batch data for efficiency).
- **Exporters** - send the processed data to a backend (e.g., Prometheus, a remote storage system).

**When would you use the OpenTelemetry Collector?**
The Collector is particularly useful when you have multiple services sending metrics, when you need to standardize or enrich data before it reaches Prometheus, or when you want a single point for managing how observability data flows through your infrastructure. It's also a natural fit when your services already push metrics using OTLP or another standard protocol, and you don't want to retrofit a pull-based scraping model onto them. If your services are short-lived (e.g., batch jobs or serverless functions) that may not be reachable for scraping, a push-based approach through the Collector is often the more practical choice.

### Exporter vs. Collector - What's the Difference?

Both exporters and collectors play a role in getting metrics to Prometheus, but they serve different purposes:

|                      | Exporter                                                                                                               | OpenTelemetry Collector                                                                                |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **What it is**       | A component (built into or alongside your service) that exposes metrics in Prometheus format                           | A standalone service that receives, processes, and forwards metrics from multiple sources              |
| **Data flow model**  | **Pull-based** - Prometheus scrapes the exporter's `/metrics` endpoint on its own schedule                             | **Push-based** - services push metrics to the Collector, which forwards them to the backend            |
| **Where it runs**    | As part of your service or as a dedicated sidecar                                                                      | As a separate, shared service in your infrastructure                                                   |
| **Best for**         | Exposing metrics from a single service or system                                                                       | Aggregating and managing metrics from many services centrally                                          |
| **Flexibility**      | Limited to one service's metrics                                                                                       | Can receive data in multiple formats, apply transformations, and forward to multiple backends          |
| **Typical use case** | Instrumenting a single application or translating metrics from a third-party tool (e.g., Node Exporter for OS metrics) | Centralizing telemetry collection across a fleet of services, especially in heterogeneous environments |

**In practice:** For most straightforward setups, an exporter per service (or per system) is sufficient - Prometheus scrapes each one directly, and the pull model keeps things simple and easy to reason about. The OpenTelemetry Collector becomes valuable when your observability pipeline grows in complexity - for example, when you need to enrich metrics with additional labels, forward data to multiple destinations, consolidate telemetry from services using different instrumentation standards, or support services that push metrics rather than waiting to be scraped.

---
## Labels

**Labels** are key-value pairs attached to a metric that allow you to differentiate between different dimensions of the same metric.

For example, the metric `http_requests_total` is more useful when you know *which* HTTP method and *which* status code each request had. Labels let you capture that:

```
http_requests_total{method="GET", status="200"} 1027
http_requests_total{method="POST", status="500"} 3
```

With labels, a single metric becomes a family of related time series - one for each unique combination of label values. This lets you slice and filter your data when querying in Grafana.

> **Note:** The metric name is itself a label under the hood, called `__name__`. This means you can select a metric by name using a label selector: `{__name__="http_requests_total"}`. This is rarely needed day-to-day, but useful to know when building advanced queries.

**Best practices for labels:**
- Keep the number of label values bounded. Avoid using labels with high cardinality values (e.g., user IDs or request IDs), as each unique combination creates a separate time series, which can overload Prometheus.
- Use labels for dimensions you'll actually filter or group by: `environment`, `region`, `service_name`, `method`, `status_code`.
For more information about naming and label best practices, refer to the official [Metric and label naming](https://prometheus.io/docs/practices/naming/) documentation.

---

## Golden Signals

The **Golden Signals** are four key metrics that Google's Site Reliability Engineering (SRE) practices define as the most important indicators of a service's health. Monitoring these four signals gives you a strong baseline understanding of how your service is performing.

In Google's words:
> If you measure all four golden signals and page a human (trigger an alert) when one signal is problematic, your service will be at least decently covered by monitoring.

| Signal         | What It Tells You                                                                                                           | Example Metric                            |
| -------------- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| **Latency**    | How long it takes to serve a request                                                                                        | `http_request_duration_seconds`           |
| **Traffic**    | How much demand is being placed on your system                                                                              | `http_requests_total` per second          |
| **Errors**     | The rate of failed requests                                                                                                 | `http_requests_total{status=~"5.."}`      |
| **Saturation** | How "full" your service is (CPU, memory, queue depth). It's also about predicting in advance when the service will be full. | `process_cpu_seconds_total`, memory gauge |

When designing what to monitor in your service, start with the Golden Signals. They answer the most critical question: **"Is my service working correctly for my users?"**

---

# Advanced

## Types of Service Discovery

Instead of manually telling Prometheus the address of every service to collect from, **service discovery** allows Prometheus to automatically find your services as they come and go.

Prometheus supports several service discovery mechanisms:

| Type                     | How It Works                                                                     | Best For                                                                       |
| ------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Static configuration** | You manually list service addresses in the config file                           | Simple setups, fixed infrastructure                                            |
| **Kubernetes SD**        | Prometheus automatically discovers pods, services, and endpoints in your cluster | Kubernetes environments                                                        |
| **HTTP SD**              | Prometheus calls an HTTP endpoint that returns a list of targets in JSON format  | Custom or dynamic environments where targets are managed by an external system |
| **File-based SD**        | Prometheus reads a file (updated by another tool) to get service addresses       | Custom or hybrid setups                                                        |
| **Consul SD**            | Prometheus queries Consul (a service registry) to find services                  | Environments using Consul                                                      |
| **EC2 SD**               | Prometheus discovers AWS EC2 instances automatically                             | AWS-based infrastructure                                                       |

In a Kubernetes environment, **Kubernetes Service Discovery** is the standard approach. When combined with the Prometheus Operator (see below), Prometheus can automatically discover and collect from any service that has the right configuration applied - without any manual updates to the Prometheus config.

---

## Prometheus Operator

The **Prometheus Operator** is a tool for Kubernetes that simplifies deploying and managing Prometheus - and the broader monitoring ecosystem around it. Instead of manually editing Prometheus configuration files, you define what you want to monitor using standard Kubernetes resources, and the Operator takes care of the rest.

The Operator works by introducing a set of **Custom Resource Definitions (CRDs)** - extensions to Kubernetes that let you describe your monitoring setup in the same way you'd describe any other part of your infrastructure.

These CRDs fall into two groups:

### Instance-Based Resources

These resources manage the **deployment and lifecycle** of the monitoring components themselves:

| Resource          | What It Does                                                                                                                                         |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Prometheus`      | Deploys and manages a Prometheus instance in your cluster, including replicas, storage, and which scrape configurations to use                       |
| `Alertmanager`    | Deploys and manages an Alertmanager instance for handling and routing alert notifications                                                            |
| `PrometheusAgent` | A lightweight variant of Prometheus that only collects and forwards metrics (no local storage or alerting) - useful for large-scale or remote setups |
| `ThanosRuler`     | Deploys a Thanos Ruler instance for evaluating alerting and recording rules across multiple Prometheus instances                                     |

### Config-Based Resources

These resources define **what to collect and how** - they don't deploy anything themselves, but are picked up by the instance-based resources above and translated into Prometheus configuration automatically:

| Resource             | What It Does                                                                                                                                         |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ServiceMonitor`     | Tells Prometheus to collect from a Kubernetes **Service** (and the pods behind it), selected by label                                                |
| `PodMonitor`         | Similar to `ServiceMonitor`, but targets **Pods** directly rather than through a Service                                                             |
| `Probe`              | Defines monitoring for **ingresses or static targets** - typically used with a blackbox exporter to check endpoint availability                      |
| `ScrapeConfig`       | A lower-level resource for defining custom scrape configurations, including targets outside the cluster                                              |
| `PrometheusRule`     | Defines **alerting and recording rules** that Prometheus (or Thanos Ruler) will evaluate. Rules are loaded dynamically without restarting Prometheus |
| `AlertmanagerConfig` | Configures routing rules and receivers for a specific team or namespace within Alertmanager                                                          |

### How It All Connects

The `Prometheus` CRD uses **selectors** to discover which config-based resources apply to it. For example, a Prometheus instance can be configured to pick up all `ServiceMonitor` resources with a specific label - meaning each team can manage their own `ServiceMonitor` independently, and Prometheus will automatically include them without any central configuration change.

```
Prometheus CRD
  ├── serviceMonitorSelector  →  picks up matching ServiceMonitors
  ├── podMonitorSelector      →  picks up matching PodMonitors
  ├── probeSelector           →  picks up matching Probes
  ├── scrapeConfigSelector    →  picks up matching ScrapeConfigs
  └── ruleSelector            →  picks up matching PrometheusRules
```

### What This Means for You

If your service is running on Kubernetes and the Prometheus Operator is installed, you don't need to touch any Prometheus configuration files. You apply a `ServiceMonitor` resource alongside your service (matching the labels your Prometheus instance is watching), and Prometheus will automatically start collecting your metrics.

**Example ServiceMonitor:**

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-service-monitor
  labels:
    release: prometheus  # Must match the Prometheus instance's serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: my-service    # Selects Services with this label
  endpoints:
    - port: metrics      # The port name on your Service that exposes /metrics
      interval: 30s      # How often Prometheus collects
```

For the full reference on all available resources and their configuration options, see the [official Prometheus Operator documentation](https://prometheus-operator.dev/docs/).

---

## Collecting - VM vs. Kubernetes

How you configure Prometheus to collect infrastructure-level metrics depends on where your service is running. The two most common environments are a Virtual Machine (VM) and a Kubernetes cluster, and each requires a different scrape configuration in `prometheus.yml`.

### Collecting from a VM (Node Exporter)

On a VM, infrastructure metrics (CPU, memory, disk, network) are exposed by the **Node Exporter** - a Prometheus exporter that runs directly on the machine and serves metrics at `http://<machine-ip>:9100/metrics`.

To tell Prometheus to collect from it, you add a static scrape job to `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "node-exporter"
    static_configs:
      - targets:
          - "10.0.0.5:9100"   # The IP and port of the Node Exporter on your VM
          - "10.0.0.6:9100"   # Add one entry per machine
```

Each target is a specific machine address. Prometheus will collect from every address listed, and will automatically attach `job="node-exporter"` and `instance="10.0.0.5:9100"` labels to all metrics - so you always know which machine a metric came from.

**What gets collected:** Host-level metrics for each machine - CPU usage, memory, disk I/O, filesystem usage, and network traffic, all scoped to the individual VM.

### Collecting from Kubernetes (kubernetes_sd_config)

In a Kubernetes environment, services and pods are dynamic - they come and go, and their IP addresses change. Rather than maintaining a static list of addresses, you use `kubernetes_sd_config` in `prometheus.yml` to let Prometheus discover targets automatically from the Kubernetes API.

```yaml
scrape_configs:
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod               # Discover all pods in the cluster
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"           # Only collect from pods with this annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)             # Use the pod's custom /metrics path if specified
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace # Attach the pod's namespace as a label
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod       # Attach the pod name as a label
```

The `role` field tells Prometheus what Kubernetes objects to discover. The most common roles are:

| Role        | What Prometheus Discovers                                                                   |
| ----------- | ------------------------------------------------------------------------------------------- |
| `pod`       | All pods in the cluster (or namespace)                                                      |
| `endpoints` | The endpoints behind a Kubernetes Service                                                   |
| `node`      | All nodes in the cluster - used for collecting Node Exporter metrics running as a DaemonSet |

**What gets collected:** Metrics from the pods or nodes themselves - including your application metrics and, when using the `node` role with a Node Exporter DaemonSet, infrastructure metrics per cluster node.

> **Note:** When using the Prometheus Operator, you typically don't write `kubernetes_sd_config` directly. The Operator generates this configuration for you based on the `ServiceMonitor` and `PodMonitor` resources you apply. The raw `prometheus.yml` approach is relevant when running Prometheus without the Operator.

### Key Difference

The fundamental difference between the two approaches is **how targets are defined**:

|                                                 | VM (Static)                            | Kubernetes (Dynamic)                                                            |
| ----------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------- |
| **Target discovery**                            | Manually listed IP addresses           | Automatically discovered via Kubernetes API                                     |
| **Config update needed when a target changes?** | Yes - you must update `prometheus.yml` | No - Prometheus re-discovers automatically                                      |
| **Metrics scope**                               | Per-machine host metrics               | Per-pod or per-node metrics with Kubernetes context (namespace, pod name, etc.) |
| **Labels attached**                             | `job`, `instance` (IP address)         | `job`, `instance`, plus Kubernetes metadata (`namespace`, `pod`, `node`, etc.)  |
