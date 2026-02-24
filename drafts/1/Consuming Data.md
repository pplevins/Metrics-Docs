# Observability Guide: Consuming Your Data



Understanding how your data travels from your code to your screen helps you trust the monitoring process. Once you have configured your Exporters (as detailed in the Metrics Collection guide), the consumption of that data follows a highly automated, secure pipeline.

### The High-Level Flow

1. **Generate (The Service Level):** Your application goes about its normal business operations. As users interact with your Service, your internal Collectors silently gather timing and counting data without slowing down your software.
2. **Expose (The Exporter Level):** The Exporter gathers all the Collector data, translates it into the Prometheus format, and updates your secure `/metrics` Endpoint. This data represents a snapshot of your system's health at that exact second.
3. **Collect (The Prometheus Level):** On a strict schedule (usually every 15 to 30 seconds), Prometheus reaches out to your Endpoint, reads the snapshot, and securely stores the data in its time-series database. 
4. **Consume and Visualize (The Grafana Level):** Grafana - your visual dashboarding tool - connects to the Prometheus database. When you open a dashboard, Grafana instantly queries the stored data and transforms the raw numbers into the charts, graphs, and alerts you use to make business decisions.