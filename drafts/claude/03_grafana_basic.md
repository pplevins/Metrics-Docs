# Grafana

## Basic

### What Is Grafana?

**Grafana** is an open-source platform for data visualization, monitoring, and alerting. It is the interface you'll use to explore your metrics, build dashboards, and set up alert rules that notify your team when something in your system needs attention.

In the context of this observability setup, Grafana connects to Prometheus as its data source and lets you query and visualize all the metrics your services are exposing.

Grafana is a rich platform with many features. For the full official documentation, visit:
**[https://grafana.com/docs/grafana/latest/](https://grafana.com/docs/grafana/latest/)**

---

### Basic Concepts and Terms

Before diving into building dashboards, here are the key terms you'll encounter in Grafana:

**Data Source**
A data source is the backend system that Grafana queries to retrieve data. In your setup, the data source is **Prometheus**. You configure a data source once, and all dashboards and panels in Grafana can use it.

**Dashboard**
A dashboard is a collection of panels organized on a single page. A dashboard gives you a consolidated view of a system, a service, or a specific concern (e.g., "API Service Health" or "Infrastructure Overview"). You can have as many dashboards as you need.

**Panel**
A panel is a single visualization unit within a dashboard. Each panel displays the result of one or more PromQL queries. A panel can be a time-series graph, a single stat, a table, a gauge, a bar chart, and more.

**Panel Query**
The PromQL expression inside a panel that retrieves the data to display. Each panel can have multiple queries, and their results can be combined or displayed as separate series.

**Row**
A row is a horizontal grouping element within a dashboard. Rows help you organize related panels and can be collapsed to simplify the view.

**Variable**
A variable is a dynamic placeholder in a dashboard that allows you to filter or switch what the dashboard displays - for example, switching between environments (`prod` / `staging`) or between service instances. Variables appear as dropdown selectors at the top of a dashboard.

**Alert Rule**
An alert rule is a condition you define that Grafana evaluates against your metrics. When the condition is met (e.g., error rate > 5% for 5 minutes), Grafana triggers an alert and sends a notification through the configured contact point.

**Contact Point**
A contact point is where alert notifications are sent - for example, an email address, a Slack channel, or a PagerDuty integration.

**Folder**
Dashboards in Grafana are organized into folders. Use folders to group dashboards by team, service, or environment.

---

### Template Dashboards

Rather than building every dashboard from scratch, Grafana supports **template dashboards** - pre-built dashboards that you can import and use immediately.

**Grafana.com Dashboard Library**
The Grafana community publishes hundreds of ready-made dashboards for common use cases (Node Exporter, Kubernetes, PostgreSQL, NGINX, and more) at:
**[https://grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/)**

Each dashboard on this site has an **ID** you can use to import it directly into your Grafana instance.

**How to import a template dashboard:**
1. In Grafana, go to **Dashboards → Import**.
2. Enter the dashboard ID from Grafana.com (or upload a JSON file).
3. Select your Prometheus data source when prompted.
4. Click **Import**.

Your organization may also maintain its own set of standard dashboard templates. See the **Templates and Common Panels** section of this documentation for examples and descriptions of the available templates in your environment.
