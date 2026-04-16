# Lesson 26.2 — Read-Only File System in Kubernetes Pods

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective
Demonstrate how to configure Kubernetes Pods with read-only root filesystems and mount writable volumes for applications that require write access to specific directories.

## Prerequisites
- kubectl installed and configured
- Access to a running Kubernetes cluster
- Familiarity with Pod configurations and security contexts

## Steps

### Step 1: Create the Initial Pod Configuration

Create a YAML configuration to set up a Pod with a read-only root filesystem:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: read-only-pod
spec:
  containers:
  - name: read-only-container
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true	
EOF
```

### Step 2: Check Pod Status and Logs

Check the status of the Pod:

```bash
kubectl get pods
```

Check the Pod logs to observe errors from Nginx attempting to write to read-only directories:

```bash
kubectl logs read-only-pod
```

### Step 3: Modify the Pod Configuration for Writable Directories

Update the YAML configuration to include writable directories using `emptyDir` volumes:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: read-only-pod
spec:
  containers:
  - name: read-only-container
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: writable-logs
      mountPath: /var/log/nginx
    - name: writable-cache
      mountPath: /var/cache/nginx
    - name: writable-conf
      mountPath: /etc/nginx/conf.d
    - name: writable-run
      mountPath: /var/run
  volumes:
  - name: writable-logs
    emptyDir: {}
  - name: writable-cache
    emptyDir: {}
  - name: writable-conf
    emptyDir: {}
  - name: writable-run
    emptyDir: {}
EOF
```

### Step 4: Verify the Updated Pod

Confirm the updated Pod is running:

```bash
kubectl get pods
```

### Step 5: Test the Read-Only Filesystem

Exec into the Pod to test the read-only setting:

```bash
kubectl exec -it read-only-pod -- /bin/sh
```

Attempt to write to the root filesystem:

```bash
echo "This should fail" > /test.txt
```

This should produce an error, confirming the root filesystem is read-only.

Attempt to write to a mounted writable directory:

```bash
echo "This should succeed" > /var/log/nginx/test
```

Try running `apt-get update` to verify the read-only filesystem affects package management:

```bash
apt-get update
```

This should fail because it requires a writable filesystem.

## Verification

- Pod status shows Running with all containers ready
- Write attempts to root filesystem return "Read-only file system" errors
- Write attempts to mounted volumes succeed without errors
- Package management commands fail due to read-only filesystem restrictions

## Cleanup

Delete the Pod when finished:

```bash
kubectl delete pod read-only-pod
```
