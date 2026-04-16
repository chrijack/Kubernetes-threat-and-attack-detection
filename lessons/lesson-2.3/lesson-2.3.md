# Lesson 2.3 — Audit Logging Tuning and Batch Parameters

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Configure and tune Kubernetes API server audit logging with batch parameters to balance comprehensive auditing with system performance.

## Prerequisites

- Kubernetes cluster with kubelet access
- API server manifest file at `/etc/kubernetes/manifests/kube-apiserver.yaml`
- Root or sudo access to modify API server configuration

## Steps

### Step 1: Create the Audit Policy File

Create an audit policy file at `/etc/kubernetes/audit-policy.yaml`:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["pods", "configmaps"]
```

### Step 2: Edit the API Server Manifest

Edit the API server manifest file:

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add or modify the following flags under `spec.containers.command`:

```yaml
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-maxage=30
- --audit-log-maxbackup=3
- --audit-log-maxsize=100
- --audit-log-batch-buffer-size=10000
- --audit-log-batch-max-size=1000
- --audit-log-batch-max-wait=5s
- --audit-log-format=json
- --audit-log-truncate-enabled=true
- --audit-log-truncate-max-event-size=100000
```

### Step 3: Restart the API Server

The kubelet will automatically restart the API server:

```bash
sudo systemctl restart kubelet
```

### Step 4: Test Batching

Create multiple ConfigMaps to demonstrate event batching:

```bash
for i in {1..20}; do
  kubectl create configmap test-cm-$i --from-literal=key$i=value$i
done
```

Check the audit log entries:

```bash
sudo grep 'test-cm-' /var/log/kubernetes/audit/audit.log | wc -l
```

Note: The number of log entries will be less than 20, demonstrating that events were batched.

### Step 5: Test Truncation

Create a large ConfigMap to demonstrate event truncation:

```bash
kubectl create configmap large-cm --from-literal=key=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 200000)
```

Check the audit log:

```bash
sudo grep 'large-cm' /var/log/kubernetes/audit/audit.log | jq .
```

The event will be truncated to the specified max size (100000 bytes).

### Step 6: Monitor Performance

Check API server resource usage:

```bash
kubectl top pod -n kube-system | grep apiserver
```

Monitor audit log file size:

```bash
du -sh /var/log/kubernetes/audit/audit.log
```

Monitor log file growth:

```bash
watch -n 5 'ls -lh /var/log/kubernetes/audit/audit.log'
```

Check for audit-related errors:

```bash
sudo journalctl -u kubelet | grep -i audit | grep -i error
```

## Verification

Verify the configuration is applied by checking the API server logs:

```bash
sudo crictl logs $(sudo crictl ps -q --name kube-apiserver) | grep audit
```

Confirm audit log is being written:

```bash
cat /var/log/kubernetes/audit/audit.log
```

## Cleanup

Remove test ConfigMaps:

```bash
for i in {1..20}; do
  kubectl delete configmap test-cm-$i
done
kubectl delete configmap large-cm
```
