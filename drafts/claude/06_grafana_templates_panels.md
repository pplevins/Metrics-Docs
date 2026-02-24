# Templates and Common Panels

## Common Panel Examples

The following are examples of panels you'll commonly find in service dashboards. Each example includes the panel type, purpose, and a sample PromQL query.

---

### Request Rate (Traffic)

**Panel type:** Time series
**Purpose:** Shows how many requests your service is receiving per second. This is the "Traffic" Golden Signal and gives you a baseline understanding of load.

```promql
sum(rate(http_requests_total{service="$service", environment="$environment"}[5m])) by (status)
```

**Tips:** Break down by `status` label so you can see the split between successful and failed requests at a glance.

---

### Error Rate (%)

**Panel type:** Time series or Stat
**Purpose:** Shows the percentage of requests that resulted in an error (5xx responses). This is the "Errors" Golden Signal.

```promql
sum(rate(http_requests_total{service="$service", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="$service"}[5m]))
* 100
```

**Tips:** Add a threshold at `1` (yellow) and `5` (red) to make degradation visually obvious.

---

### Request Latency - p50, p95, p99

**Panel type:** Time series
**Purpose:** Shows how long your service takes to respond. Multiple percentiles give a complete picture - p50 is the typical experience, p99 reveals the worst-case experience.

```promql
# p50 (median)
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le))

# p95
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le))

# p99
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le))
```

**Tips:** Display all three as separate series on the same graph. Use seconds as the unit (Grafana will auto-format to ms/s).

---

### CPU Usage

**Panel type:** Time series
**Purpose:** Tracks how much CPU your service or node is consuming over time. Part of the "Saturation" Golden Signal.

For a pod in Kubernetes:
```promql
sum(rate(container_cpu_usage_seconds_total{pod=~"$pod", namespace="$namespace"}[5m])) by (pod)
```

For a node (using Node Exporter):
```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle", instance="$instance"}[5m])) * 100)
```

---

### Memory Usage

**Panel type:** Time series or Gauge
**Purpose:** Shows current and historical memory consumption. Useful for spotting memory leaks and approaching limits.

For a pod in Kubernetes:
```promql
sum(container_memory_working_set_bytes{pod=~"$pod", namespace="$namespace"}) by (pod)
```

**Tips:** Set the unit to bytes (Grafana will auto-convert to MB/GB). Add a threshold at your pod's memory limit to visualize how close you are to the ceiling.

---

### Active Connections / Queue Depth

**Panel type:** Time series or Stat
**Purpose:** Shows the current number of active connections to your service or the current size of a processing queue. This is a Saturation metric.

```promql
my_service_active_connections{environment="$environment"}
```

**Tips:** Since this is a gauge metric, you can plot it directly without `rate()`. Add an alert if this consistently approaches your configured connection limit.

---

### Service Availability (Uptime)

**Panel type:** Stat
**Purpose:** Displays a simple "up/down" indicator showing whether your service is reachable by Prometheus.

```promql
up{job="my-service", environment="$environment"}
```

The value is `1` if the service is up and Prometheus can reach it, and `0` if it cannot. Apply color thresholds: `0` = red, `1` = green.

---

### Pod Restart Count

**Panel type:** Stat or Time series
**Purpose:** Tracks how many times your pods have restarted. Frequent restarts indicate crashes or OOM (Out of Memory) kills.

```promql
sum(increase(kube_pod_container_status_restarts_total{namespace="$namespace"}[1h])) by (pod)
```

---

## Explanation of Existing Templates

This section describes the standard dashboard templates available in your environment. Templates are pre-built dashboards that cover common monitoring needs and can be imported directly.

> **Note:** Your organization maintains a library of standard templates. Contact your observability team or refer to your internal Grafana folder structure for the current list. The table below describes the typical templates you should expect to find.

---

### Service Overview Template

**Purpose:** A top-level health dashboard for a single service. Covers all four Golden Signals: traffic, error rate, latency, and saturation.

**Key panels included:**
- Request rate (RPS) broken down by status code.
- Error rate percentage with alerting thresholds.
- p50 / p95 / p99 latency time series.
- CPU and memory usage for the service's pods.
- Pod restart count over the last hour.
- Service `up` status indicator.

**When to use it:** Import this template as the first dashboard for any new service you onboard to monitoring.

---

### Infrastructure / Node Overview Template

**Purpose:** A host-level dashboard showing the health of the machines or nodes running your services. Uses data from the Node Exporter.

**Key panels included:**
- CPU usage per node.
- Memory usage and available memory per node.
- Disk I/O (read/write) per node.
- Network traffic (ingress/egress) per node.
- Disk space usage and projected time to full.

**When to use it:** Use this dashboard to monitor the underlying infrastructure your services run on, and to investigate resource contention or saturation at the host level.

---

### Kubernetes Cluster Template

**Purpose:** A cluster-level dashboard showing the health and resource utilization of your Kubernetes environment.

**Key panels included:**
- Pod status across all namespaces (running / pending / failed).
- Resource requests vs. actual usage (CPU and memory) per namespace.
- Node readiness status.
- Recent pod restarts across the cluster.

**When to use it:** Use this dashboard as a starting point when investigating cluster-wide issues, resource exhaustion, or scheduling problems.

---

### Database Template (PostgreSQL / MySQL)

**Purpose:** Monitors database performance and health, using the corresponding database exporter.

**Key panels included:**
- Active connections and connection pool utilization.
- Query throughput (queries per second).
- Query latency (average and p99).
- Replication lag (if applicable).
- Cache hit rate.

**When to use it:** Import this template alongside any service that depends on a relational database to monitor database-side performance separately from application performance.
