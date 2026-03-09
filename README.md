Below is your **fully updated README**, rewritten cleanly and professionally, with **Grafana and Prometheus exposed using AWS Load Balancer (Ingress)** instead of localhost port‑forward.

All local host steps, port‑forward commands, and localhost URLs have been removed.

This README now reflects your **production-style ALB exposure** for both Grafana and Prometheus.

You can copy‑paste this entire file into your repository.

***

# POC-5-Observability-and-Monitoring-Tools-Integration-on-Top-of-Kubernetes-Deployment-Workflow

# EKS Application Deployment with Prometheus and Grafana Monitoring (ALB Exposed)

Architecture Flow:

Application → Kubernetes Deployment → Service → Ingress (AWS ALB) → Prometheus → Grafana

***

# Prerequisites

Ensure the following tools are installed:

*   AWS CLI
*   kubectl
*   Helm
*   Docker
*   Access to an EKS cluster
*   AWS Load Balancer Controller installed in the cluster

Verify cluster access:

```bash
kubectl get nodes
```

***

# Step 1: Deploy the Application

Ensure your Docker image is pushed to DockerHub or another registry.

Example:

    dockerhub-user/my-app:latest

***

# Step 2: Create Kubernetes Manifests

## deployment.yaml

(Add your deployment YAML here)

***

## service.yaml

(Add your service YAML here)

***

## ingress.yaml

(Add your ingress YAML here)

***

## namespace.yaml

(Add namespace resource here)

***

# Step 3: Apply Kubernetes Manifests

Apply manifests in correct order:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl get pods -n <namespace>
kubectl get svc -n <namespace>
kubectl get ingress -n <namespace>
```

Ingress will automatically provision an AWS ALB.

***

# Step 4: Install Prometheus and Grafana Using Helm

Add Helm repo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install monitoring stack:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

Verify pods:

```bash
kubectl get pods -n monitoring
```

***

# Step 5: Expose Grafana Using AWS Load Balancer (Ingress)

Grafana service name:

    monitoring-grafana

Create `grafana-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitoring-grafana
            port:
              number: 80
```

Apply it:

```bash
kubectl apply -f grafana-ingress.yaml
```

Retrieve ALB URL:

```bash
kubectl get ingress -n monitoring
```

Open in browser:

    http://<your-alb-dns>.amazonaws.com

***

# Step 6: Retrieve Grafana Admin Password

Run:

```bash
kubectl get secret monitoring-grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Credentials:

    Username: admin
    Password: <decoded-password>

Use these credentials on the ALB URL to log in.

***

# Step 7: Expose Prometheus Using AWS Load Balancer (Ingress)

Prometheus service name:

    monitoring-kube-prometheus-prometheus

Create `prometheus-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitoring-kube-prometheus-prometheus
            port:
              number: 9090
```

Apply:

```bash
kubectl apply -f prometheus-ingress.yaml
```

Get ALB URL:

```bash
kubectl get ingress -n monitoring
```

Open:

    http://<your-prometheus-alb>.amazonaws.com

Prometheus does not require a password.

***

# Step 8: Expose Metrics from the Application

Prometheus scrapes metrics from an endpoint like:

    /metrics

Your application must expose this endpoint using Prometheus client libraries:

*   Python: prometheus\_client
*   NodeJS: prom-client
*   Spring Boot: actuator + micrometer-registry-prometheus

***

# Step 9: Create ServiceMonitor

ServiceMonitor is required for Prometheus Operator to discover the application's metrics endpoint.

Example `servicemonitor.yaml`:

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

Apply:

```bash
kubectl apply -f servicemonitor.yaml
```

***

# Step 10: Verify Metrics in Prometheus

Open the Prometheus ALB URL.

Check the targets section:

    Status → Targets

Your application should appear as an active scrape target.

***

# Step 11: Visualize Metrics in Grafana

Open Grafana using the ALB URL, login with admin credentials, and create dashboards or use built‑in ones.

Example PromQL queries:

    http_requests_total
    rate(http_requests_total[1m])
    container_cpu_usage_seconds_total
    container_memory_usage_bytes

***

# Final Architecture

Application Pod  
→ Service  
→ /metrics endpoint  
→ Prometheus (ALB exposed)  
→ Grafana (ALB exposed)

Infrastructure monitoring:

Node → Node Exporter → Prometheus → Grafana

***

# Complete DevOps Workflow

Docker Image  
↓  
Kubernetes YAML: Deployment, Service, Ingress  
↓  
Application running in EKS  
↓  
ServiceMonitor  
↓  
Prometheus scrapes metrics  
↓  
Grafana dashboards  
↓  
Grafana and Prometheus exposed using AWS ALB

***

# Future Improvements

*   Loki for log aggregation
*   Promtail or Alloy for log forwarding
*   Alertmanager for alert routing
*   Slack or PagerDuty integration for alert notifications

***

If you want, I can also produce a **diagram** and **end-to-end architecture image** for this POC.
