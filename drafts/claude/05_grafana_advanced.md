# Grafana Advanced

## Using Variables in Dashboards

**Variables** make your dashboards dynamic and reusable. Instead of creating a separate dashboard for each environment, service, or instance, you define variables that act as filters - and your dashboard adapts to whichever option is selected.

Variables appear as **dropdown selectors at the top of a dashboard**. When you change the selected value, all panels that use that variable update automatically.

---

### Common Use Cases for Variables

- Switch between `production`, `staging`, and `development` environments.
- Filter panels to show data for a specific service or pod.
- Select a time aggregation window (e.g., `5m`, `10m`, `30m`) to use in `rate()` queries.
- Filter by region, cluster, or any other label dimension in your metrics.

---

### How to Create a Variable

1. Open your dashboard and click the **Dashboard settings** icon (gear icon, top right).
2. Go to the **Variables** tab.
3. Click **Add variable**.
4. Configure the variable:

| Field | What to Set |
|---|---|
| **Name** | The variable name used in queries (e.g., `environment`) |
| **Label** | The display label shown in the dropdown (e.g., "Environment") |
| **Type** | See types below |
| **Query** | For query-type variables, a PromQL expression to populate the options |
| **Multi-value** | Allow selecting multiple values at once |
| **Include All option** | Adds an "All" option to select all values at once |

5. Click **Update** and then save the dashboard.

---

### Variable Types

**Query** - The most common type. Grafana queries Prometheus to populate the list of options dynamically.

Example: To create a variable that lists all available `service` label values:
```promql
label_values(http_requests_total, service)
```
This queries Prometheus for all unique values of the `service` label in the `http_requests_total` metric.

**Custom** - You manually define the list of options.

Example: A list of environments: `production,staging,development`

**Constant** - A fixed value used across the dashboard (useful for things like a data source URL or a shared prefix).

**Interval** - A time interval variable, useful for passing into `rate()` or `increase()` functions.

Example options: `1m,5m,10m,30m,1h`

---

### Using Variables in Queries

Once a variable is defined, you reference it in panel queries using the `$variable_name` syntax (or `${variable_name}` for clarity inside label selectors).

**Example:** With a variable named `environment`, your query becomes:

```promql
sum(rate(http_requests_total{environment="$environment"}[5m])) by (service)
```

When the user selects `production` in the dropdown, Grafana substitutes `$environment` with `production` automatically.

**Multi-value variables:** When **Multi-value** is enabled, the selected values are joined with a regex OR pattern. Use `=~` (regex match) instead of `=` in your query:

```promql
http_requests_total{service=~"$service"}
```

---

### Variable Chaining

Variables can depend on each other. For example, you can have a `cluster` variable, and a `namespace` variable that only shows namespaces within the selected cluster:

```promql
label_values(kube_pod_info{cluster="$cluster"}, namespace)
```

This creates a cascading filter experience where selecting a cluster automatically updates the available namespaces.

---

### Best Practices for Variables

- Always use variables for **environment**, **service**, and **instance** filters - this avoids the need for duplicate dashboards.
- Enable **Include All option** when you want users to be able to see aggregated data across all values.
- Use descriptive **Labels** (what's shown in the UI) even if the variable **Name** is short.
- Keep the number of variables reasonable - too many dropdowns at the top of a dashboard make it harder to use.
