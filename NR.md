
***

# **README: Monitoring an EKS Cluster Using New Relic**

## **1. Overview**

This document explains how to integrate an existing Amazon EKS cluster with New Relic to enable cluster-level monitoring, pod-level insights, and Prometheus metrics ingestion.

The setup includes:

*   Existing EKS cluster
*   Applications already deployed
*   Existing Prometheus + Grafana monitoring
*   New Relic Kubernetes integration using Helm
*   Automatic discovery of workloads, pods, namespaces
*   Optional Prometheus metrics ingestion
*   Optional alert creation using NRQL

This guide assumes you already have a working EKS cluster and `kubectl` configured.

***

## **2. Prerequisites**

*   Existing EKS cluster
*   IAM role and access to the cluster
*   `kubectl` installed and configured
*   `helm` installed
*   New Relic account
*   New Relic **Ingest License Key** (Type: `INGEST – LICENSE`)

To find the correct key:

1.  Log in to New Relic
2.  Go to **Administration → API Keys**
3.  Look for: **INGEST – LICENSE**

This is the only key accepted for Kubernetes monitoring.

***

## **3. Add New Relic Helm Repository**

```bash
helm repo add newrelic https://helm-charts.newrelic.com
helm repo update
```

***

## **4. Install New Relic Kubernetes Integration**

The Helm chart deploys:

*   New Relic Infrastructure Agent
*   Kubernetes State Metrics integration
*   Kubelet metrics scraper
*   Prometheus metrics adapter (optional)
*   Metadata injection webhook

Run the following command with your actual license key:

```bash
helm upgrade --install newrelic-bundle newrelic/nri-bundle \
  --namespace newrelic --create-namespace \
  --set global.licenseKey=<NEW_RELIC_LICENSE_KEY> \
  --set global.cluster=eks-cluster \
  --set newrelic-infrastructure.privileged=true \
  --set newrelic-infrastructure.hostPID=true \
  --set newrelic-infrastructure.hostNetwork=true \
  --set global.lowDataMode=true
```

### Why these flags are required?

*   `privileged=true`
*   `hostPID=true`
*   `hostNetwork=true`

EKS blocks access to Kubelet metrics without these settings. These are required for pod/container monitoring.

***

## **5. Verify Installation**

```bash
kubectl get pods -n newrelic
```

Expected running components include:

*   `nri-kubernetes`
*   `nri-prometheus`
*   `nri-kubelet`
*   `kube-state-metrics`
*   `nri-metadata-injection`

If pods are in CrashLoopBackOff, verify the license key and privilege settings.

***

## **6. View Cluster Metrics in New Relic**

Navigate to:

**New Relic → Kubernetes → Clusters → eks-cluster**

You should now see:

*   Cluster CPU/Memory usage
*   Nodes
*   Pods
*   Deployments
*   Namespaces
*   DaemonSets
*   StatefulSets
*   Events

Workloads are discovered automatically.  
No manual workload creation is required.

***

## **7. (Optional) Enable Prometheus Metrics Ingestion**

If your application or services expose Prometheus metrics using `/metrics`, you can annotate them for New Relic auto‑scraping.

Example annotation in Deployment or Service:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
```

After applying this update, Prometheus-format metrics will appear under:

**Query Your Data → NRQL Editor**

You can test with:

```sql
FROM Metric SELECT * WHERE metricName LIKE '%http%'
```

***

## **8. Basic NRQL Alerts (Recommended)**

### Pod CrashLoopBackOff

```sql
FROM K8sPodSample
SELECT count(*)
WHERE reason = 'CrashLoopBackOff'
```

### Node CPU Above 80%

```sql
FROM Metric
SELECT average(cpuPercent)
WHERE clusterName = 'eks-cluster'
FACET nodeName
```

### Memory Usage Above 80%

```sql
FROM Metric
SELECT average(memoryWorkingSetBytes / memoryLimitBytes * 100)
WHERE clusterName = 'eks-cluster'
```

### High Application Latency (Prometheus)

```sql
FROM Metric
SELECT percentile(http_server_request_duration_seconds, 95)
```

***

## **9. Troubleshooting**

### Pods are restarting with Error status

Check for incorrect license key type.  
Ensure you are using **INGEST – LICENSE**, not a USER key.

### “Cannot connect to kubelet” errors

Ensure you set:

    --set newrelic-infrastructure.hostPID=true
    --set newrelic-infrastructure.hostNetwork=true

### Metrics not visible in UI

Wait 2–3 minutes for the first data ingestion.  
Check logs:

```bash
kubectl logs -n newrelic <pod-name>
```

***

## **10. Conclusion**

The New Relic Kubernetes integration provides:

*   Full cluster visibility
*   Pod and node performance metrics
*   Automatic workload detection
*   Prometheus metrics ingestion
*   Alerting and dashboards based on NRQL

No additional configuration is required after the Helm installation.  
Your EKS cluster is now fully monitored with New Relic.

***


*   A complete end‑to‑end EKS + New Relic Observability POC document
*   CI/CD integration with New Relic deployments
