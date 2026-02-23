# Grafana Alerts

## What Is Grafana Alerts?

**Grafana Alerts** is the built-in alerting system in Grafana. It allows you to define conditions on your metrics and automatically notify your team when something requires attention — before users start reporting problems.

An alert watches a metric query continuously. When the metric crosses a threshold you define (for example, error rate > 5% for 5 consecutive minutes), Grafana fires an alert and sends a notification to the configured contact point.

### Key Components of the Alerting System

**Alert Rule** — The definition of what condition to monitor, how long it must persist before firing, and what to evaluate. Alert rules are the core of the alerting system.

**Contact Point** — Where notifications are sent when an alert fires. Examples include email, Slack, PagerDuty, Microsoft Teams, and webhooks.

**Notification Policy** — Rules that determine which contact point receives which alerts. Policies route alerts based on labels (e.g., send `severity=critical` alerts to PagerDuty and `severity=warning` to Slack).

**Alert State** — Each alert rule has a current state:
- **Normal** — The condition is not met. Everything is fine.
- **Pending** — The condition is met, but the "for" duration hasn't elapsed yet.
- **Firing** — The condition has been met for the required duration. A notification has been sent.
- **Resolved** — The condition is no longer met. A resolution notification is sent.

---

## How to Create an Alert Rule in Grafana

### Step 1: Open the Alert Rule Editor

In the Grafana left sidebar, go to **Alerting → Alert rules**, then click **New alert rule**.

Alternatively, you can create an alert rule directly from a panel: open the panel editor, go to the **Alert** tab, and click **Create alert rule from this panel** — this pre-fills the query for you.

---

### Step 2: Define the Query

In the alert rule editor, write the **PromQL query** you want to evaluate. The query must return a **numeric scalar value** (a single number per evaluation cycle).

**Example:** Alert when the error rate exceeds 5%:
```promql
sum(rate(http_requests_total{service="my-api", status=~"5.."}[5m]))
/
sum(rate(http_requests_total{service="my-api"}[5m]))
* 100
```

You can use **multiple queries** in an alert rule (labeled A, B, C, etc.) and use **expressions** to combine them.

---

### Step 3: Set the Threshold Condition

Under **Expressions**, define when the alert should fire. Select:

- **Input:** The query result to evaluate (e.g., query A).
- **Condition:** The threshold operator: `IS ABOVE`, `IS BELOW`, `IS OUTSIDE RANGE`, etc.
- **Threshold value:** The numeric value to compare against (e.g., `5` for 5%).

**Example:** `A IS ABOVE 5` — fires when the error rate exceeds 5%.

---

### Step 4: Set the Evaluation Behavior

| Setting | What It Means |
|---|---|
| **Evaluate every** | How often Grafana checks the condition (e.g., every `1m`) |
| **For** | How long the condition must be continuously met before the alert fires (e.g., `5m`) |

The **"For" duration** is important. It prevents false alarms from brief spikes. Setting `for: 5m` means the alert only fires if the condition is consistently true for 5 full minutes.

> **Recommendation:** For most alerts, set `Evaluate every: 1m` and `For: 5m`. Adjust based on how fast-moving the metric is and how quickly you need to be notified.

---

### Step 5: Add Labels and Annotations

**Labels** on alert rules are used by notification policies to route alerts to the right contact point. Common labels:

```
severity = critical   # or: warning, info
team = backend
service = my-api
environment = production
```

**Annotations** provide additional context in the notification. Standard annotations:

| Annotation | Purpose |
|---|---|
| `summary` | A one-line description of the alert (e.g., "High error rate on Orders API") |
| `description` | More detail, including what to check (e.g., "Error rate is {{ $value }}%, exceeding the 5% threshold. Check recent deployments and upstream dependencies.") |
| `runbook_url` | A link to your team's runbook for this alert |

---

### Step 6: Set the Contact Point

Under **Notifications**, configure where this alert should be sent:

- You can use the **default notification policy** (which routes to your organization's default contact point).
- Or **override** the routing for this specific alert rule and send it directly to a specific contact point.

Click **Save rule** when done.

---

## How to Manage Alert Rules

### Viewing Alert Status

Go to **Alerting → Alert rules** to see all defined alert rules and their current states. You can filter by folder, data source, or label.

**Alerting → State and health** gives you a live view of all firing and pending alerts.

### Editing an Alert Rule

From **Alerting → Alert rules**, find the rule you want to edit and click the pencil icon. Make your changes and save.

### Silencing an Alert

If you're doing planned maintenance and want to prevent alerts from firing during that window, use a **Silence**:

1. Go to **Alerting → Silences**.
2. Click **Add silence**.
3. Set the start and end time for the silence window.
4. Add label matchers to specify which alerts to silence (e.g., `service=my-api`).
5. Add a comment explaining why (e.g., "Planned maintenance — deploying v2.1.0").
6. Click **Submit**.

During the silence window, matching alerts will still change state (they'll show as "Firing" in the UI) but no notifications will be sent.

### Muting Notifications (Mute Timings)

For recurring scheduled windows (e.g., weekly maintenance windows), use **Mute Timings** instead of creating a new silence each time:

1. Go to **Alerting → Mute timings**.
2. Click **New mute timing**.
3. Define the recurring schedule (day of week, time range, month, etc.).
4. Attach the mute timing to a notification policy in **Alerting → Notification policies**.

### Alert Rule Best Practices

- **Alert on symptoms, not causes.** For example, alert on "high error rate" (what users experience), not "high CPU" (which may or may not affect users). Add saturation alerts as secondary signals.
- **Set meaningful thresholds.** Avoid setting thresholds so sensitive that alerts fire constantly. Alerts that fire too often get ignored.
- **Always fill in summary and description annotations.** A notification that says "Alert: high error rate on my-api. Error rate is 8.3%. Check recent deployments and database connectivity." is far more actionable than one that just says "Alert firing."
- **Link to a runbook.** Use the `runbook_url` annotation to link directly to your team's documented response procedure for that alert.
- **Group related alerts.** Use labels consistently (`team`, `service`, `environment`) so your notification policy can route them correctly and group related alerts together in notifications.
- **Review and tune regularly.** As your system evolves, alert thresholds that made sense six months ago may no longer be appropriate. Schedule periodic reviews of alert rules.
