# Observability Guide: Grafana Visualizations

Grafana is your primary interface for understanding your system. It takes the massive amounts of data collected by Prometheus and turns it into clear, actionable visual insights. 



## 1. Basic Concepts

For a comprehensive technical deep-dive, you can always visit the [official Grafana documentation](https://grafana.com/docs/). Before diving in, familiarize yourself with these core terms:

* **Data Source:** The database Grafana pulls information from. For our purposes, your Data Source is Prometheus.
* **Panel:** A single visual element, such as a line chart, a pie chart, or a simple number display.
* **Dashboard:** A collection of Panels organized together on a single screen to tell a complete story about a Service.
* **Alert Rule:** A background monitor that watches your Panels. If a metric crosses a dangerous threshold, the Alert Rule sends a notification to your team.
* **Template Dashboards:** Pre-configured, industry-standard Dashboards that you can load instantly, saving you the time of building them from scratch.

---

## 2. Creating a Dashboard

### Using a Built-in Dashboard (Importing JSON)
You rarely need to build a Dashboard from a blank screen. The observability community provides thousands of pre-built templates formatted as JSON files. To use one, simply click **Import** in the Grafana menu, upload the JSON file or paste its ID, and Grafana will automatically generate a fully populated Dashboard.

### How to Write a Panel and Choose Visualizations
When adding a new Panel, choosing the right visual format is critical for readability:
* **Time Series (Line Chart):** Best for showing trends over time, such as tracking website traffic over a 7-day period.
* **Stat (Single Number):** Best for high-priority current values, like your system's total uptime percentage or current active users.
* **Bar Gauge:** Best for comparing the current capacity of multiple identical items, like looking at the disk space across five different database servers.
* **Table:** Best for viewing lists of detailed information, such as the top 10 most frequent error codes.

### How to Write a Query
To populate a Panel, you must ask Prometheus for the specific data using a language called PromQL (Prometheus Query Language). Think of it as a search bar for your metrics. 
For example, typing the metric name `http_requests_total` into the query box will immediately draw a line chart of your overall website traffic.

### Overview vs. Drill Down Dashboards
* **Overview Dashboards:** Designed for executives and quick health checks. They sit at a high level and answer: "Is the platform operating normally?" They rely heavily on Stat panels and traffic/error summaries.
* **Drill Down Dashboards:** Designed for engineers during an incident. They answer: "Exactly which Endpoint on which server is failing?" They are deeply granular, showing memory usage, specific API latencies, and underlying infrastructure health.

### Dashboard Organization and Best Practices
* **Use Descriptions:** Always fill out the "Description" field in your Panel settings. A new team member should be able to hover over a chart and read exactly what it measures.
* **The "Make-Sense" Approach:** Read your Dashboard like a newspaper. Put the most critical, high-level business metrics at the very top. Place the complex, technical infrastructure metrics at the bottom.
* **Panel Drill Down:** Utilize Grafana's "Data Links" feature. You can make an Overview chart clickable so that clicking on a spike in errors automatically opens the detailed Drill Down Dashboard for that specific Service.

---

## 3. Advanced Dashboard Features

### Using Variables in Dashboards
Variables are a powerful way to keep your Grafana workspace clean. Instead of creating a separate Dashboard for "Payment Service," "Login Service," and "Cart Service," you can create one single "Service Health" Dashboard. 
By adding a Variable, a drop-down menu appears at the top of the screen. You can simply select "Login Service" from the drop-down, and all the Panels will instantly update to show data only for that specific Service.

---

## 4. Templates and Common Panel Examples

### Common Panels for Client Dashboards
When building your first Dashboards, we recommend including these standard Panels:
* **System Uptime:** A green Stat panel showing the percentage of time your Service has been available today.
* **Request Volume:** A Time Series chart showing user requests per minute, helping you identify your busiest hours.
* **Error Rate:** A Time Series chart tracking 400 and 500-level HTTP errors, alerting you to broken links or server crashes.

### Explanation of Existing Templates
We provide several built-in templates out-of-the-box. The most common is the **Node Exporter Template**. This Dashboard automatically visualizes hardware metrics—like CPU, RAM, and Network bandwidth—for any server connected to our system, requiring absolutely zero custom query writing on your part.

---

## 5. Grafana Alerts



### What is Grafana Alerts?
Dashboards are great, but you cannot stare at them 24/7. Grafana Alerts act as your automated watchdogs. They continuously monitor your PromQL queries in the background and send messages to your communication tools (like Slack, Microsoft Teams, or email) the moment something breaks.

### How to Make an Alert Rule
To create an Alert, you define three things:
1. **The Condition:** The metric and the threshold (e.g., "If `cpu_usage` goes above `90%`").
2. **The Duration:** How long the condition must be true before alarming (e.g., "...for more than `5 minutes`"). This prevents false alarms from temporary 1-second spikes.
3. **The Action:** Where to route the message (e.g., "Send a critical alert to the DevOps Slack channel").

### How to Manage Alert Rules
You can manage all your alerts from the "Alerting" tab in the main Grafana menu. Here, you can silence alerts during a planned maintenance window, adjust the sensitivity of rules that trigger too often, and view a historical log of past alert notifications.