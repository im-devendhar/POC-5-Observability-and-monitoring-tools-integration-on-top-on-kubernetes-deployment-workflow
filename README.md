# POC-5-Observability-and-monitoring-tools-integration-on-top-on-kubernetes-deployment-workflow

# EKS Application Deployment with Prometheus & Grafana Monitoring


Architecture Flow:

Application → Kubernetes Deployment → Service → Ingress → Prometheus → Grafana

---

# Prerequisites

Ensure the following tools are installed:

* AWS CLI
* kubectl
* Helm
* Docker
* Access to an EKS cluster

Verify connection to the cluster:

```bash
kubectl get nodes
```

---

# Step 1: Deploy the Application

You should already have a Docker image pushed to DockerHub or another registry.

Example:

```
dockerhub-user/my-app:latest
```

---

# Step 2: Create Kubernetes Manifests

## deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: dockerhub-user/my-app:latest
        ports:
        - containerPort: 8080
```

---

## service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

---

## ingress.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

---

# Step 3: Apply Kubernetes Manifests

Deploy the application to EKS:

```bash
kubectl apply -f deployment.yml
kubectl apply -f service.yml
kubectl apply -f ingress.yml
```

Verify deployment:

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

Your application should now be running in the cluster.

---

# Step 4: Install Prometheus & Grafana Using Helm

Add the Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the monitoring stack:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring --create-namespace
```

This installs:

* Prometheus
* Grafana
* Alertmanager
* Node Exporter
* Kubernetes monitoring dashboards

Verify installation:

```bash
kubectl get pods -n monitoring
```

---

# Step 5: Access Grafana

Port forward Grafana service:

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open browser:

```
http://localhost:3000
```

Retrieve admin password:

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" | base64 --decode
```

Login credentials:

```
Username: admin
Password: <decoded password>
```

---

# Step 6: Expose Metrics from the Application

Prometheus collects metrics from the `/metrics` endpoint.

Example:

```
http://my-app-service:8080/metrics
```

Your application must expose this endpoint using a Prometheus client library.

Examples:

Python → prometheus_client
NodeJS → prom-client
Spring Boot → actuator/prometheus

---

# Step 7: Create ServiceMonitor

Prometheus Operator uses a **ServiceMonitor** resource to discover application metrics.

Create:

## servicemonitor.yml

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - default
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

Apply it:

```bash
kubectl apply -f servicemonitor.yml
```

---

# Step 8: Verify Metrics in Prometheus

Port forward Prometheus:

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090 -n monitoring
```

Open:

```
http://localhost:9090
```

Navigate to:

```
Status → Targets
```

You should see your application service being scraped.

---

# Step 9: Visualize Metrics in Grafana

Open Grafana dashboards and run queries like:

```
http_requests_total
```

or

```
rate(http_requests_total[1m])
```

Create dashboards to monitor:

* Request rate
* CPU usage
* Memory usage
* Application performance

---

# Final Architecture

```
Application Pod
      │
      │ /metrics
      ▼
Prometheus
      │
      ▼
Grafana Dashboard
```

Infrastructure monitoring:

```
Node
 │
Node Exporter
 │
Prometheus
 │
Grafana
```

---

# Complete DevOps Workflow

```
Docker Image
     │
Deployment.yml
Service.yml
Ingress.yml
     │
Application running in EKS
     │
ServiceMonitor
     │
Prometheus scrapes metrics
     │
Grafana dashboards
```

---

# Future Improvements

For a complete observability stack you can also add:

* Loki → Log aggregation
* Alloy → Log and metric collector
* Alertmanager → Alert routing
* PagerDuty / Slack → Alert notifications
