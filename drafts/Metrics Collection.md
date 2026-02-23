# Observability Guide: Metrics Collection

Welcome to your system observability guide. To ensure your business runs without interruption, we need to monitor the health and performance of your software. This document explains how we connect your Services to our monitoring tools so you can gain real-time visibility into your system's performance.



## What is Metrics Collection?

Metrics collection is the continuous process of gathering numerical data about your Services. Think of it as checking your system's vital signs. Instead of waiting for a user to report a problem, metrics allow us to see exactly how much traffic your system is handling, how fast it is responding, and if any errors are occurring behind the scenes.

### Prometheus: The Engine
Prometheus is the industry-standard tool we use to gather and store your metrics. Instead of your Service constantly pushing data out, Prometheus is designed to periodically visit a specific web address on your Service—called an **Endpoint**—to collect the latest data. 

### Collector
A Collector is a small, lightweight piece of code that lives directly inside your application. Its only job is to count and measure things in real-time. For example, a Collector might act as a stopwatch to measure how long a database query takes, or a clicker counting how many times a user logs in. 

### Exporter
An Exporter acts as the bridge between your Service's internal Collectors and Prometheus.
* **What is it:** It is an agent that translates the raw data from your Collectors into a standardized format that Prometheus can read.
* **How to create it:** You include a Prometheus client library (available for Java, Python, Node.js, etc.) in your application's code. This library automatically creates the Exporter for you.
* **How to access it:** Once configured, the Exporter publishes your metrics to a dedicated Endpoint on your Service, typically located at a web address ending in `/metrics` (e.g., `api.yourservice.com/metrics`).
* **Adding metrics:** As your business logic grows, you can use the client library in your code to define new metrics to track, such as the number of items added to a shopping cart.
* **Viewing your metrics:** You can navigate to your `/metrics` Endpoint in any standard web browser. You will see a plain-text list of the exact data your Exporter is currently sharing.

### Labels
Labels are vital tags you attach to your metrics to categorize them. If you simply track "total server errors," it is hard to know where to fix them. By adding Labels, you can tag errors by environment (`environment="production"`), region (`region="europe"`), or customer tier. This makes your data highly filterable and searchable.

### The Four Golden Signals
When deciding what metrics to track, we highly recommend focusing on the "Four Golden Signals." These four categories provide the most accurate picture of your user's experience:

1. **Latency:** How long it takes your Service to process a request (e.g., page load time). High latency directly frustrates users.
2. **Traffic:** The amount of demand placed on your system (e.g., 500 active user sessions, or 1,000 requests per second).
3. **Errors:** The rate of requests that fail (e.g., HTTP 500 error codes). This tells you how often your Service is broken for the user.
4. **Saturation:** How "full" your Service is compared to its maximum capacity (e.g., 85% CPU usage or 90% database storage used). This helps you know when to upgrade your infrastructure.

---

## Advanced Collection Concepts

As your infrastructure grows more complex, our collection methods adapt using these advanced concepts:

* **Types of Service Discovery:** In modern cloud environments, Services are often automatically duplicated during busy hours and removed during quiet hours. Service Discovery is a feature that allows Prometheus to dynamically connect to your cloud provider (like AWS or Azure). This ensures Prometheus automatically finds the Endpoints of your new Services without you having to manually update any lists.
* **Prometheus Operator:** If your Services run on Kubernetes, the Prometheus Operator acts as an automated manager. It dramatically simplifies the configuration process. Instead of writing complex configuration files, you simply declare what Services you want monitored, and the Operator handles the entire technical setup in the background.
* **Collecting - OS vs. VM:** To get a full picture, we collect metrics from two different layers:
    * **OS (Operating System) Metrics:** This focuses on the immediate environment of your application, tracking the resources consumed specifically by your software.
    * **VM (Virtual Machine) Metrics:** This focuses on the underlying physical or virtual server (the hardware). High OS usage might mean your software needs optimization, while high VM usage means you need to buy larger servers.