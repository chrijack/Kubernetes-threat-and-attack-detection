# Lesson 4.2 — Kubernetes Security Monitoring with Loki, Prometheus, and Grafana

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Set up a comprehensive security monitoring and alerting stack for Kubernetes using Loki for log aggregation, Prometheus for metrics collection, and Grafana for visualization and alerting. Learn to detect and investigate security events based on MITRE ATT&CK techniques.

## Prerequisites

- Running Kubernetes cluster (minikube or otherwise)
- Helm installed and configured
- `kubectl` configured to access your cluster
- Basic understanding of Kubernetes namespaces and services

## Steps

### Step 1: Prepare Cluster Storage (Optional but Recommended)

Expand cluster disk space if needed:

```bash
df -h
```

If space is limited, extend the logical volume:

```bash
sudo pvresize /dev/sda3 && \
sudo lvextend -L +30G /dev/mapper/ubuntu--vg-ubuntu--lv && \
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

### Step 2: Add Helm Repositories

Add the necessary Helm chart repositories:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add kubescape https://kubescape.github.io/helm-charts
helm repo update
```

### Step 3: Start Minikube Cluster

```bash
minikube start
```

### Step 4: Install Prometheus Stack

Create a dedicated namespace and deploy kube-prometheus-stack:

```bash
kubectl create namespace prometheus

helm install -n prometheus kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false,prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

Verify the deployment:

```bash
kubectl --namespace prometheus get pods -l "release=kube-prometheus-stack"
```

### Step 5: Install Loki and Grafana Stack

Install the Loki stack for log aggregation:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack
```

The loki-stack includes Grafana and integrates with your cluster's logging.

### Step 6: Deploy Promtail as DaemonSet

Create a ConfigMap for Promtail configuration to send logs to Loki:

```yaml
# promtail-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
data:
  promtail.yaml: |
    server:
      http_listen_port: 3101
    positions:
      filename: /tmp/positions.yaml
    clients:
      - url: http://loki:3100/loki/api/v1/push
```

Deploy the ConfigMap:

```bash
kubectl apply -f promtail-config.yaml
```

### Step 7: Install Kubescape Operator

Deploy Kubescape for Kubernetes security scanning and metrics:

```bash
helm upgrade --install kubescape kubescape/kubescape-operator \
  -n kubescape --create-namespace \
  --set clusterName=$(kubectl config current-context) \
  --set kubescape.serviceMonitor.enabled=true \
  --set kubescape.serviceMonitor.namespace=prometheus \
  --set kubescape.submit=false \
  --set kubescape.enableHostScan=false \
  --set kubescape.downloadArtifacts=false \
  --set account="abc123"
```

### Step 8: Run Initial Security Scan

Perform a baseline security scan using Kubescape:

```bash
kubescape scan framework mitre
```

This scans the cluster against MITRE ATT&CK framework techniques relevant to Kubernetes.

### Step 9: Access Grafana

Retrieve the Grafana admin password:

```bash
kubectl get secret --namespace prometheus kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Forward the Grafana service to your local machine:

```bash
kubectl port-forward -n prometheus service/kube-prometheus-stack-grafana 3000:80
```

Open your browser to `http://localhost:3000` and log in with username `admin` and the retrieved password.

### Step 10: Access Prometheus (Optional)

To verify metrics collection, port-forward to Prometheus:

```bash
kubectl port-forward -n prometheus service/kube-prometheus-stack-prometheus 9090
```

Open `http://localhost:9090` to browse available metrics and verify the Kubescape ServiceMonitor is active.

### Step 11: Configure Loki Datasource in Grafana

In Grafana, add Loki as a datasource pointing to `http://loki:3100`.

### Step 12: Create Security Queries in Loki

Create LogQL queries to detect security-relevant events. For example, to detect unauthorized API server access:

```lokiql
{namespace="kube-system", container_name="kube-apiserver"} |~ "unauthorized|failed login"
```

### Step 13: Create Alerts in Grafana

1. Navigate to **Alerting** → **New Alert**
2. Select the Loki datasource
3. Define conditions using your security queries
4. Set notification channels and severity levels

Example alert condition: Trigger when unauthorized API access attempts exceed threshold within 5 minutes.

### Step 14: Map Events to MITRE ATT&CK Techniques

Use the MITRE ATT&CK matrix for Kubernetes to understand detected events:

Visit [MITRE ATT&CK - Cloud Control Matrix](https://attack.mitre.org/matrices/enterprise/cloud/) for reference.

Example mappings:
- Unauthorized login attempts → **Initial Access**
- Privilege escalation in containers → **Privilege Escalation**
- Suspicious network traffic → **Command and Control**

### Step 15: Investigate Alerts

When an alert triggers:

1. Navigate to the Loki panel in Grafana
2. Drill down into log details
3. Examine surrounding logs for context
4. Identify the source and nature of the event

### Step 16: Remediate Security Issues

Take action based on investigation findings. For example, revoke compromised credentials:

```bash
kubectl delete secret <secret-name> -n <namespace>
```

Restart affected pods to force credential refresh:

```bash
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

### Step 17: Import Grafana Dashboard

If a pre-built dashboard exists, import it into Grafana:

Reference: `grafana-dashboard.yaml`

Steps in Grafana:
1. Click **+** icon in sidebar
2. Select **Import**
3. Upload or paste the dashboard JSON/YAML

### Step 18: Document and Review

Record all security incidents, investigation findings, and remediation steps for future reference and compliance audits.

## Verification

Confirm the monitoring stack is operational:

```bash
# Check all pods are running
kubectl get pods -n prometheus
kubectl get pods -n kubescape

# Verify Prometheus scrape targets
kubectl port-forward -n prometheus service/kube-prometheus-stack-prometheus 9090
# Visit http://localhost:9090/targets

# Verify Loki is receiving logs
# Check Grafana Loki datasource and run a test query
```

## Cleanup

To remove the monitoring stack:

```bash
# Remove Kubescape
helm uninstall kubescape -n kubescape
kubectl delete namespace kubescape

# Remove Prometheus stack
helm uninstall kube-prometheus-stack -n prometheus
kubectl delete namespace prometheus

# Remove Loki stack
helm uninstall loki
```
