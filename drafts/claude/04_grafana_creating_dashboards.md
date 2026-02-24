# Creating a Dashboard in Grafana

## 1. Using a Built-In Dashboard (Import a JSON)

The fastest way to get a working dashboard is to import a pre-built one. Grafana dashboards can be exported and imported as **JSON files**, making it easy to share dashboards across teams or environments.

**To import a dashboard from a JSON file:**

1. In the Grafana left sidebar, go to **Dashboards → Import**.
2. Click **Upload JSON file** and select your `.json` dashboard file.
   Alternatively, paste the JSON content directly into the text field.
3. Review the dashboard settings - name, folder, and data source.
4. Click **Import**.

The dashboard will appear immediately with live data from Prometheus.

> **Tip:** If a panel shows "No data" after import, check that the data source is correctly selected and that the metrics used in the dashboard are actually being collected from your service.

---

## 2. How to Write a Panel

A panel is the building block of any dashboard. Here's how to add and configure one:

**To add a new panel:**
1. Open a dashboard, then click **Add panel** (the `+` icon at the top).
2. Select **Add a new panel**.
3. In the panel editor, write your PromQL query in the query field at the bottom.
4. Choose a visualization type from the right-side panel.
5. Give the panel a clear title and, optionally, a description.
6. Click **Apply** to save the panel to the dashboard.

### Recommended Visualization Types by Use Case

| Use Case | Recommended Visualization |
|---|---|
| Metrics over time (trends, spikes) | **Time series** (line graph) |
| Current value at a glance | **Stat** (single big number) |
| Progress toward a limit or threshold | **Gauge** |
| Comparing values across categories | **Bar chart** |
| Showing detailed data with multiple columns | **Table** |
| Status indicators (OK / warning / critical) | **Stat** with color thresholds |
| Rate of change over time | **Time series** |

**General rule:** Use a **time series** when you want to see how a value changes over time. Use a **stat** or **gauge** when you want to see the current state right now.

---

## 3. How to Write a Query

Grafana panels retrieve data using **PromQL** - Prometheus Query Language. PromQL is a powerful expression language that lets you filter, aggregate, and compute metrics.

### PromQL Basics

**Select a metric by name:**
```promql
http_requests_total
```

**Filter by label:**
```promql
http_requests_total{status="200", service="my-api"}
```

**Calculate a rate (for counters):**
Counters only go up, so you use `rate()` to calculate how fast they're increasing per second:
```promql
rate(http_requests_total[5m])
```
This calculates the per-second average rate of increase over the last 5 minutes.

**Aggregate across multiple series:**
```promql
sum(rate(http_requests_total[5m])) by (status)
```
This sums the request rate across all services, grouped by HTTP status code.

**Calculate a percentage:**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100
```
This calculates the percentage of requests that returned a 5xx error.

**Calculate a latency percentile (from a histogram):**
```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```
This calculates the 95th percentile (p95) request latency - meaning 95% of requests completed within this duration.

### Query Tips

- Always use `rate()` or `increase()` with **counters**. Plotting raw counter values produces a line that always goes up and is not useful.
- Use `[5m]` or `[10m]` as the range in `rate()`. Shorter windows show more detail; longer windows are smoother.
- Use the `=~` operator for regex matching on labels: `{status=~"5.."}` matches all 5xx codes.
- Use the `!=` operator to exclude a label value: `{environment!="dev"}`.

---

## 4. Overview vs. Drill Down

Good dashboards are built in **layers**: a high-level overview that shows system health at a glance, and deeper drill-down views for investigating specific issues.

**Overview dashboards** give you a bird's-eye view of your entire system or a service. They answer: "Is everything OK right now?" Panels on overview dashboards typically use aggregated metrics (summed across all instances), stat panels for key numbers, and time-series graphs for trends.

**Drill-down dashboards** let you investigate a specific service, instance, or time window in detail. They answer: "What exactly is happening with *this* service?" Drill-down panels typically break metrics down by label (e.g., by individual pod or endpoint), show more granular time windows, and may include table panels with detailed breakdowns.

**Best practice:** Link your overview dashboard panels to corresponding drill-down dashboards using Grafana's **panel data link** feature. When a team member sees an anomaly on the overview, they can click directly into the detail view without having to search for the right dashboard.

---

## 5. Dashboard Organization and Best Practices

A well-organized dashboard is easy to read, easy to act on, and self-explanatory to someone who didn't build it.

### Adding Descriptions to Panels

Every panel should have a clear **title** and, for non-obvious panels, a **description**. In Grafana, you can add a description in the panel editor under the **Panel** tab → **Description** field. The description appears as a tooltip (`ℹ`) on the panel header.

A good description answers:
- What is this metric measuring?
- Why does it matter?
- What should I do if it looks wrong?

**Example:** Instead of a panel titled "p99 Latency" with no description, write: *"p99 request latency for the Orders API. Values above 500ms may indicate database slowness or upstream dependency issues."*

### The "Make-Sense" Approach

Before publishing a dashboard, ask yourself: **"Would a new team member understand what this dashboard is telling them?"**

- Use plain, descriptive panel titles. Avoid abbreviations that are not universally known.
- Arrange panels in a logical flow: left-to-right and top-to-bottom, from most important to most detailed.
- Group related panels using **rows** with clear row titles (e.g., "Request Traffic", "Error Rates", "Latency", "Infrastructure").
- Use consistent color conventions: red for errors/critical, yellow for warnings, green for healthy.

### Panel Drill Down (Data Links)

You can configure any panel to link to another dashboard when clicked. This is called a **data link** and is how you connect overview panels to their corresponding drill-down dashboards.

**To add a data link:**
1. Open the panel editor.
2. Under **Panel → Data links**, click **Add link**.
3. Set the **URL** to the target dashboard URL. You can include variables (e.g., `${__value.labels.service}`) to pass context dynamically.
4. Give the link a descriptive label like "View service detail →".

### General Best Practices

- **Fewer, better panels are more useful than many cluttered ones.** If you wouldn't look at a panel regularly, don't include it.
- **Set meaningful thresholds and color ranges** on stat and gauge panels so the visual signal is immediately clear.
- **Use consistent time ranges** across panels in the same dashboard. If one panel shows the last hour and another shows the last day, comparisons become confusing.
- **Name dashboards clearly** and organize them into folders by service, team, or environment.
- **Version your dashboards** - export them as JSON and store them in version control alongside your service code.
- **Avoid duplicating panels** across dashboards. If multiple dashboards need the same panel, consider whether they should link to a shared dashboard instead.

For a deeper dive, follow the official [Grafana dashboard best practices](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/) guide.