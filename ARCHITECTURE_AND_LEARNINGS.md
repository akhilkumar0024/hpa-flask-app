# Kubernetes HPA with External Metrics: Complete Architecture & Learnings Guide

This document serves as a comprehensive reference for understanding the architecture, data flow, component roles, and design patterns used in this project.

---

## 1. Project Overview & Core Objective

The primary objective of this project is to implement **Kubernetes Horizontal Pod Autoscaling (HPA) using External Metrics** rather than traditional hardware resource metrics (CPU or Memory).

### The Problem It Solves
- **Standard HPA (Resource Metrics)**: Scales pods based on CPU/Memory usage provided by Kubernetes `metrics-server`. This is ineffective when an application is bottlenecked by external factors (e.g., job queue depth, database backlogs, machine learning inference requests, or incoming event streams) where CPU consumption does not immediately spike.
- **External Metrics HPA (This Project)**: Scales pods based on **application or business metrics** (e.g., a prediction workload value stored in Redis) queried through Prometheus and translated into the Kubernetes External Metrics API via the Prometheus Adapter.

---

## 2. Architecture & Component Roles

```mermaid
flowchart TD
    subgraph Client Traffic & Application
        Client["Client / External Producer"] -->|1. POST /set-data (prediction=80)| Flask["Flask App (Gunicorn :5000)"]
        Flask -->|2. Store key 'prediction'| Redis[("Embedded Redis (:6379)")]
    end

    subgraph Metrics Collection
        Prometheus["Prometheus Server (:9090)"] -->|3. Scrapes /metrics every scrape_interval| K8sService["K8s Service: python:5000"]
        K8sService --> Flask
        Flask -->|Reads from Redis & sets Gauge| MetricsEndpoint["/metrics output:\ncustom_metric 80.0"]
    end

    subgraph Kubernetes Autoscaling Pipeline
        PromAdapter["k8s-prometheus-adapter"] -->|4. Queries PromQL {job='python-app'}| Prometheus
        K8sAPI["K8s API Server\n(external.metrics.k8s.io)"] -->|5. Registered APIService| PromAdapter
        HPA["Horizontal Pod Autoscaler (HPA)\nTarget Value = 50"] -->|6. Queries metric: custom_metric| K8sAPI
        HPA -->|7. Scale Pods (e.g. 1 -> 2 replicas)| Deployment["flask-app-deployment"]
        Deployment -->|8. Spawns new Pods| Pods["Flask Pod 1, Flask Pod 2..."]
    end
```

### Component Breakdown

| Component | File(s) | Key Responsibility |
| :--- | :--- | :--- |
| **Flask App** | `app.py`, `Dockerfile`, `start.sh` | Serves HTTP endpoints, records values into Redis, and exposes metrics at `/metrics`. |
| **Redis** | `start.sh`, `app.py` | Local in-memory store holding the scalar key `prediction` and the `logs` list. |
| **K8s Deployment** | `deployment.yaml` | Deploys the containerized Flask app (`sonal04/flaskapp`). Acts as the scaling target for HPA. |
| **K8s Service** | `service.yaml` | Provides stable cluster DNS `python.default.svc.cluster.local:5000` for Prometheus scraping. |
| **Prometheus Config** | `prometheus-values.yaml` | Helm configuration telling Prometheus to scrape `python.default.svc.cluster.local:5000` under job `python-app`. |
| **Prometheus Adapter** | `prometheus-adapter.yaml` | Helm configuration that translates Prometheus time-series into the Kubernetes External Metrics API (`external.metrics.k8s.io`). |
| **HPA** | `hpa.yaml` | Evaluates `custom_metric` against target value `50`, and scales replicas between 1 and 5. |
| **SQLite (`logs.db`)** | `app.py` | Local database writing timestamp logs on the `/log` endpoint. Completely optional / decoupled from autoscaling. |
| **Locust** | `locustfile.py` | Load-testing script used to generate simulated traffic to `/` and `/logs`. |

---

## 3. End-to-End Metrics & Autoscaling Flow

1. **Setting Data**:
   A client sends a POST request to `/set-data` with `prediction=80`.
   Flask executes:
   ```python
   redis_conn.set('prediction', str(prediction_value))
   ```
2. **Prometheus Scraping**:
   - Prometheus is configured with:
     ```yaml
     - job_name: 'python-app'
       static_configs:
         - targets: ['python.default.svc.cluster.local:5000']
     ```
   - Because Prometheus defaults the scrape path to `/metrics`, it automatically sends an HTTP `GET` to:
     `http://python.default.svc.cluster.local:5000/metrics`
3. **Metric Generation**:
   - Inside `app.py`:
     ```python
     @app.route('/metrics')
     def metrics():
         value = redis_conn.get('prediction')
         gauge.set(int(value))
         return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}
     ```
   - `gauge.set(80)` sets the internal gauge object in memory.
   - `generate_latest()` formats the metric into plain text:
     ```text
     # HELP custom_metric Custom Metric Description
     # TYPE custom_metric gauge
     custom_metric 80.0
     ```
4. **Prometheus Adapter Translation**:
   - The adapter queries Prometheus for all series matching `{job="python-app"}`.
   - It finds `custom_metric`, evaluates `metricsQuery: "<<.Series>>"`, and registers `custom_metric` under Kubernetes API:
     `/apis/external.metrics.k8s.io/v1beta1/namespaces/default/custom_metric`
5. **HPA Decision**:
   - The HPA controller queries the API every 15s.
   - Target value is `50`. Current value is `80`.
   - Formula:
     $$\text{desiredReplicas} = \left\lceil \text{currentReplicas} \times \left(\frac{\text{currentMetricValue}}{\text{targetMetricValue}}\right) \right\rceil = \left\lceil 1 \times \frac{80}{50} \right\rceil = 2$$
   - HPA scales `flask-app-deployment` from 1 to 2 pods.
6. **Scale Down**:
   - When the metric drops below 50, HPA waits 60 seconds (`stabilizationWindowSeconds: 60`) before terminating pods to avoid flapping.

---

## 4. Key Code & Configuration Learnings

### A. Python & `prometheus_client` Syntax
- **`Gauge`**: A metric that can increase or decrease arbitrarily (unlike a `Counter`, which only increases).
- **`generate_latest()`**: Inspects the default collector registry and serializes all active metrics into the Prometheus exposition text format.
- **`CONTENT_TYPE_LATEST`**: Constant set to `'text/plain; version=0.0.4; charset=utf-8'`.
- **Default HTTP Method**: In Flask, omitting `methods=[...]` on `@app.route(...)` defaults to `methods=['GET']`. Because Prometheus scrapes via `GET`, no explicit method declaration is needed.

### B. Redis & SQLite Separation
- **`prediction`**: Redis only holds **one** value for `prediction` at any time because `redis_conn.set('prediction', ...)` overwrites previous values.
- **SQLite (`logs.db`)**: Used exclusively inside `/log` to append timestamp strings. It is completely independent of the autoscaling pipeline and can be removed without breaking anything.

### C. Prometheus Adapter Syntax
```yaml
rules:
  default: false
  external:
    - seriesQuery: '{job="python-app"}'
      resources:
        template: <<.Resource>>
      metricsQuery: "<<.Series>>"
```
- **`seriesQuery`**: PromQL selector used to discover matching metric series.
- **`<<.Series>>`**: Dynamic template parameter representing the discovered metric series name (`custom_metric`).
- **`resources: template: <<.Resource>>`**: Boilerplate template associating labels with Kubernetes resources (not strictly used for external metrics).
- **Direct Naming**: Because no rename rule (`name.as`) was used, the metric retains the exact name defined in Python (`custom_metric`), which is why `hpa.yaml` refers to `custom_metric`.

---

## 5. Advanced Patterns: Handling Multiple Metrics & Calculations

When dealing with multiple data points (e.g., `prediction1 = 80`, `prediction2 = 100`):

### Pattern 1: Application-Level Calculation (Best for custom business logic)
Compute the value directly in Python before setting the gauge:
```python
p1 = int(redis_conn.get('prediction1') or 0)
p2 = int(redis_conn.get('prediction2') or 0)
combined_value = p1 + p2  # Or any custom formula
gauge.set(combined_value)
```

### Pattern 2: Prometheus Adapter / PromQL Calculation (DevOps Best Practice)
Expose raw metrics from Python using labels (`prediction_value{type="p1"}`), and let Prometheus Adapter calculate the aggregate:
```yaml
rules:
  external:
    - seriesQuery: '{job="python-app"}'
      resources:
        template: <<.Resource>>
      name:
        as: "total_predictions"       # Custom name required for calculated results
      metricsQuery: "sum(prediction_value{job='python-app'})"
```
In `hpa.yaml`:
```yaml
metrics:
- type: External
  external:
    metric:
      name: total_predictions
    target:
      type: Value
      value: 150
```

### Pattern 3: Multi-Metric Scaling in HPA
List multiple metrics in `hpa.yaml`. Kubernetes calculates replica requirements for each metric independently and picks the **maximum** replica count:
```yaml
metrics:
- type: External
  external:
    metric:
      name: prediction1
    target:
      type: Value
      value: 50
- type: External
  external:
    metric:
      name: prediction2
    target:
      type: Value
      value: 100
```

---

## 6. Cluster & Environment Assumptions
- **Pre-installed Prometheus**: The project assumes `kube-prometheus-stack` is installed in the `default` namespace (service DNS: `prometheus-kube-prometheus-prometheus.default.svc.cluster.local:9090`).
- **Prometheus Port**: The default port `9090` is used.
- **Helm Values**: `prometheus-values.yaml` and `prometheus-adapter.yaml` are Helm configuration files designed for installation via `helm install -f <values.yaml>`.
