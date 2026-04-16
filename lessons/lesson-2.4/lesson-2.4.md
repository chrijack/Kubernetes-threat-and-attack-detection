# Lesson 2.4 — Kubernetes Audit Logging with Grafana Loki

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Configure Kubernetes audit logging, set up persistent storage with Longhorn, and visualize audit logs using Grafana and Loki.

## Prerequisites

- Kubernetes cluster with control plane and at least two worker nodes
- `kubectl` and `Helm` installed locally
- Sufficient disk space on the control plane node

## Steps

### Step 1: Prepare the Control Plane Node

Check available disk space:

```bash
df -h
```

Expand the logical volume on the control plane (add 30GB):

```bash
sudo pvresize /dev/sda3 && \
sudo lvextend -L +30G /dev/mapper/ubuntu--vg-ubuntu--lv && \
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

### Step 2: Install Longhorn Storage

Add the Longhorn Helm repository:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Create the namespace for Longhorn:

```bash
kubectl create namespace longhorn-system
```

Install Longhorn:

```bash
helm install longhorn longhorn/longhorn --namespace longhorn-system
```

### Step 3: Configure Kubernetes Audit Log Backend

Create the audit directory on the control plane:

```bash
sudo mkdir -p /var/log/kubernetes/audit/
```

Create the audit policy file at `/etc/kubernetes/audit-policy.yaml`:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    omitStages:
      - RequestReceived
```

Edit the kube-apiserver manifest to enable audit logging:

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the following flags to the `spec.containers[0].command` section:

```yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-maxage=30
- --audit-log-maxbackup=10
- --audit-log-maxsize=100
```

Mount the audit directory in the kube-apiserver Pod spec:

```yaml
volumeMounts:
- name: audit-logs
  mountPath: /var/log/kubernetes/audit/
volumes:
- name: audit-logs
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
```

### Step 4: Install Grafana Helm Chart

Add the Grafana Helm repository:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Step 5: Install Loki

Create a namespace for Loki:

```bash
kubectl create ns loki
```

Create a `values.yml` file:

```yaml
minio:
  enabled: true
```

Install Loki:

```bash
helm upgrade --install --namespace loki logging grafana/loki -f values.yml \
  --set loki.auth_enabled=false
```

### Step 6: Install Grafana

Install Grafana in the same namespace:

```bash
helm upgrade --install --namespace=loki loki-grafana grafana/grafana
```

### Step 7: Configure Grafana Service for External Access

Edit the Grafana service to use NodePort:

```bash
kubectl edit svc -n loki loki-grafana
```

Update the service spec:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loki-grafana
  namespace: loki
spec:
  type: NodePort
  ports:
    - name: service
      port: 80
      protocol: TCP
      targetPort: 3000
      nodePort: 32000
  selector:
    app.kubernetes.io/instance: loki-grafana
    app.kubernetes.io/name: grafana
```

### Step 8: Access Grafana

Retrieve the admin password:

```bash
kubectl get secret --namespace loki loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

Access Grafana in your browser:

```
http://192.168.56.10:32000
```

Login with:
- Username: `admin`
- Password: (from the secret above)

### Step 9: Configure Loki as a Data Source

In Grafana, add Loki as a data source using the URL:

```
http://loki-gateway.loki.svc.cluster.local
```

### Step 10: Query Audit Logs

Open the Explore tab in Grafana and run the following queries:

Query logs from the logging namespace:

```
{namespace="logging"} |= ""
```

Query etcd logs:

```
{container="etcd",namespace="kube-system"}
```

Query for etcd errors:

```
{container="etcd",namespace="kube-system"} |= "error"
```

Query audit logs for delete operations:

```
{job="audit-logs"} | json | apiVersion = `audit.k8s.io/v1` | verb = `delete`
```

### Step 11: Generate Audit Log Events

Create a test deployment:

```bash
kubectl create deployment test-deploy --image=nginx
```

Create a ConfigMap:

```bash
kubectl create configmap test-config --from-literal=username=admin
```

Create a Secret:

```bash
kubectl create secret generic test-secret --from-literal=password=my-password
```

## Verification

Verify all components are running:

```bash
kubectl get all -n loki
kubectl get all -n longhorn-system
```

Tail the audit logs on the control plane:

```bash
sudo tail -f /var/log/kubernetes/audit/audit.log | jq .
```

Verify audit events appear in Grafana Loki by querying the logs as described in Step 10.

## Cleanup

Delete the test resources:

```bash
kubectl delete deployment test-deploy
kubectl delete configmap test-config
kubectl delete secret test-secret
```

Remove Grafana, Loki, and Longhorn:

```bash
helm uninstall loki-grafana --namespace loki
helm uninstall logging --namespace loki
kubectl delete namespace loki
helm uninstall longhorn --namespace longhorn-system
kubectl delete namespace longhorn-system
```

Clean up the audit log directory on the control plane:

```bash
sudo rm -rf /var/log/kubernetes/audit/
```
