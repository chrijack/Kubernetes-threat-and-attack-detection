# Lesson 27.2 — Kubernetes Audit Policy Configuration

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Configure audit logging in a Kubernetes 1.30 cluster to capture and monitor API server events at various logging levels (Metadata, Request, RequestResponse, and None).

## Prerequisites

- Access to a Kubernetes 1.30 cluster
- Administrative access to modify the API server configuration
- `kubectl` command-line access

## Steps

### Step 1: Create the Audit Policy File

Create a new file named `audit-policy.yaml`:

```bash
sudo nano /etc/kubernetes/audit-policy.yaml
```

Add the following audit policy configuration:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all requests at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["*"]

  # Log pod and configmap operations in the default namespace at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["pods", "configmaps"]
    namespaces: ["default"]

  # Log all secret operations at the RequestResponse level.
  - level: RequestResponse
    resources:
    - group: "" # core API group
      resources: ["secrets"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]
```

Save and close the file.

### Step 2: Configure the API Server

Edit the API server manifest:

```bash
sudo nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add the following flags to the `command` section:

```yaml
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=30
```

Add the following volume mounts to the `volumeMounts` section:

```yaml
volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/audit/
    name: audit-log
    readOnly: false
```

Add the corresponding volumes to the `volumes` section:

```yaml
volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/audit-policy.yaml
    type: File
- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
```

Save and exit the editor.

### Step 3: Restart the API Server

The kubelet will automatically restart the API server when the manifest file changes. If needed, restart manually:

```bash
sudo systemctl restart kubelet
```

### Step 4: Verify the Audit Policy

Check if the API server is running with the new configuration:

```bash
sudo crictl logs $(sudo crictl ps -q --name kube-apiserver)
```

Look for lines indicating that the audit policy file was loaded successfully.

### Step 5: Test the Audit Policy

Perform the following actions to verify audit logging based on the policy:

**Create a secret** (logged at RequestResponse level):

```bash
kubectl create secret generic test-secret --from-literal=password=mysecretpassword
```

Check the audit log:

```bash
sudo grep 'test-secret' /var/log/kubernetes/audit/audit.log | jq .
```

**Create a pod in the default namespace** (logged at Request level):

```bash
kubectl run nginx --image=nginx
```

Check the audit log:

```bash
sudo grep 'nginx' /var/log/kubernetes/audit/audit.log | jq .
```

**Create a configmap in the default namespace** (logged at Request level):

```bash
kubectl create configmap test-cm --from-literal=key1=value1
```

Check the audit log:

```bash
sudo grep 'test-cm' /var/log/kubernetes/audit/audit.log | jq .
```

**Create a namespace** (logged at Metadata level):

```bash
kubectl create namespace test-audit-ns
```

Check the audit log:

```bash
sudo grep 'test-audit-ns' /var/log/kubernetes/audit/audit.log | jq .
```

**List pods** (logged at Metadata level):

```bash
kubectl get pods --all-namespaces
```

Check the audit log:

```bash
sudo grep 'list pods' /var/log/kubernetes/audit/audit.log | tail -n 1 | jq .
```

**Watch services as kube-proxy** (should not be logged):

```bash
kubectl get services --watch --as=system:kube-proxy
```

Press Ctrl+C after a few seconds to stop.

Check the audit log:

```bash
sudo grep 'watch services' /var/log/kubernetes/audit/audit.log | tail -n 5 | jq .
```

### Step 6: Analyze Audit Logs with jq Filters

Use the following jq filters to extract specific events from the audit log:

```bash
# Pod Deletion
(select(.verb == "delete" and .objectRef.resource=="pods") | "A pod named: " + .objectRef.name + " was deleted in: " + .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# Pod Exec
(select(.objectRef.resource=="pods" and .objectRef.subresource == "exec") | "A shell was opened into " + .objectRef.name + " pod " + "in "+ .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# Pod Creation
(select(.verb == "create" and .objectRef.resource =="pods") | "A pod named: " + .objectRef.name + " was created in: " + .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# ConfigMap creation
(select(.verb == "create" and .objectRef.resource =="configmaps") | "A configMap named: " + .objectRef.name + " was created in: " + .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# ConfigMap Deletion
(select(.verb == "delete" and .objectRef.resource =="configmaps") | "A configMap named: " + .objectRef.name + " was deleted in: " + .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# Secret creation
(select(.verb == "create" and .objectRef.resource =="secrets") | "A secret named: " + .objectRef.name + " was created in: " + .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# Secret Deletion
(select(.verb == "delete" and .objectRef.resource =="secrets") | "A secret named: " + .objectRef.name + " was deleted in: " + .objectRef.namespace + " namespace" + " - [" + .stageTimestamp + " ]"),

# ConfigMap creation with sensitive information
(select(.verb == "create" and .objectRef.resource =="configmaps" and (.requestObject.data | tostring | contains("username") or contains("password"))) | "A configMap named: " + .objectRef.name + " was created in: " + .objectRef.namespace + " namespace," + " with sensitive information " + " - [" + .stageTimestamp + " ]")
```

To use these filters with live audit logs:

```bash
sudo tail -f /var/log/kubernetes/audit/audit.log | jq -f ./jqfilters
```

## Verification

After running the tests, verify that:

- Different operations are logged at the appropriate levels based on the audit policy
- Secrets are logged with full RequestResponse detail
- Pod and configmap operations in the default namespace are logged at Request level
- General operations are logged at Metadata level only
- Kube-proxy watch operations are not logged

## Cleanup

Remove test objects created during verification:

```bash
kubectl delete secret test-secret
kubectl delete pod nginx
kubectl delete configmap test-cm
kubectl delete namespace test-audit-ns
```
