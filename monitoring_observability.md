# Monitoring & Observability for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Prometheus Metrics and PromQL Basics**

*Prometheus config (`prometheus.yml`)*
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

*Basic PromQL Queries*
```promql
# 1. Instant query: Get current memory usage for a specific pod
container_memory_usage_bytes{pod="frontend-app-xyz"}

# 2. Rate query: HTTP 500 errors per second over the last 5 minutes
rate(http_requests_total{status="500"}[5m])
```

---

## 2. Intermediate Examples

**Alertmanager Rules and Grafana Dashboards**

*Alertmanager Rule (`alerts.yml`)*
```yaml
groups:
- name: InstanceAlerts
  rules:
  - alert: HighCpuUsage
    # Condition: CPU usage over 80%
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    # Must be true for 5 minutes before firing
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "CPU usage is at {{ $value }}%"
```

*Grafana JSON Model Snippet (Panel)*
```json
{
  "title": "API Error Rate",
  "type": "timeseries",
  "targets": [
    {
      "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (endpoint)",
      "legendFormat": "{{endpoint}}"
    }
  ]
}
```

---

## 3. Advanced Examples

**Datadog Logs/Monitors and Tracing**

*Datadog Monitor (Terraform)*
```hcl
resource "datadog_monitor" "high_error_rate" {
  name               = "High Error Rate on Payment Service"
  type               = "query alert"
  # Alert if the 5m average of 5xx errors is > 10
  query              = "avg(last_5m):sum:trace.flask.request.errors{service:payment} > 10"
  message            = "Payment API errors are spiking. Notify @pagerduty"
  
  monitor_thresholds {
    critical = 10
    warning  = 5
  }
}
```

*Distributed Tracing Concept (OpenTelemetry/Jaeger/Datadog APM)*
- **Trace ID**: A unique identifier for a single request spanning multiple microservices.
- **Span ID**: Represents a single unit of work (e.g., one database query or one API call) within the Trace.
*Interview Context:* You must know how to explain that injecting a Trace ID in HTTP headers is how observability tools map the path of a request through 10 different microservices.

---

## 4. Interview Coding Exercises

### Problem 1: PromQL Aggregation
**Task:** Write a PromQL query that returns the percentage of HTTP 5xx errors out of total HTTP requests, grouped by `service`, over the last 5 minutes.

**Solution:**
```promql
sum by (service) (rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum by (service) (rate(http_requests_total[5m])) 
* 100
```
**Explanation:** 
1. Get the rate of 5xx errors.
2. Divide `/` by the rate of *all* requests.
3. Multiply by 100 to get a percentage.
4. `sum by (service)` groups the results so you see one percentage per microservice, instead of one per individual pod/container.

### Problem 2: Alert Routing
**Task:** Write an Alertmanager configuration to send `severity: critical` alerts to PagerDuty, and everything else to Slack.
**Solution:**
```yaml
route:
  receiver: 'slack-default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'

receivers:
- name: 'slack-default'
  slack_configs:
  - api_url: 'https://hooks.slack.com/...'
- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: 'YOUR_PD_KEY'
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: PromQL Counter Issue
**Scenario:** A developer writes this query to see how many requests happened in the last 10 minutes: `http_requests_total[10m]`. Grafana throws a data type error.
**Answer:** `http_requests_total` is a Counter. `[10m]` returns a Range Vector (an array of data points over time), but Grafana graphs require an Instant Vector (a single calculated value per timestamp). 
**Fix:** You must wrap it in a rate or increase function: `increase(http_requests_total[10m])`.

### Broken Configuration 2: Missing Metrics in Prometheus
**Scenario:** A new Java app exposes metrics on `/metrics` at port 8080. Prometheus is running in K8s, but the metrics aren't showing up.
**Answer:** Prometheus needs to know where to scrape. In Kubernetes, this is usually solved by adding Annotations to the Pod/Deployment:
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```
Or, if using Prometheus Operator, by creating a `ServiceMonitor` CRD that selects the app's Service labels.

---

## 6. Common Interview Questions

**Q: "Explain the Three Pillars of Observability."**
*Answer:* 
1. **Metrics:** Numeric representations of data measured over time (e.g., CPU %, Request Rate). (Prometheus, Datadog)
2. **Logs:** Immutable records of discrete events (e.g., "User X failed login"). (ELK, Loki, Splunk)
3. **Traces:** Representations of a single user's journey through the entire distributed system. (Jaeger, Datadog APM)

**Q: "What is the RED method?"**
*Answer:* A monitoring philosophy for services. You should monitor:
- **R**ate: The number of requests per second.
- **E**rrors: The number of those requests that are failing.
- **D**uration: The amount of time those requests take (Latency).

**Q: "What is the difference between a Gauge and a Counter in Prometheus?"**
*Answer:* A **Counter** goes only UP (e.g., total HTTP requests). If a service restarts, it resets to 0. You always use `rate()` or `increase()` with it. A **Gauge** can go UP and DOWN (e.g., current memory usage, temperature). You do *not* use `rate()` on a Gauge.

---

## 7. Cheat Sheet

| PromQL Concept | Explanation |
| :--- | :--- |
| `rate(metric[5m])` | Per-second average rate of increase of the time series in the range vector. |
| `irate(metric[5m])`| Calculates the per-second rate based ONLY on the two most recent data points. Highly volatile/responsive. |
| `increase(metric[5m])` | Absolute increase in the metric over the time window. |
| `histogram_quantile(0.95, sum(rate(...)))` | Calculates the 95th percentile (p95) latency from a histogram. |
| `=~` | Regex match in label selectors (e.g., `status=~"5.."`). |
