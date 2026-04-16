# Lesson 3.4 — Installing Falco with Sidekick UI on Kubernetes 1.30

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Deploy Falco container runtime security monitoring with Falco Sidekick and Sidekick UI on a Kubernetes 1.30 cluster running on Ubuntu 22.04. Monitor runtime events and visualize alerts in real-time through the Sidekick UI.

## Prerequisites

- Linux Ubuntu 22.04 server
- Kubernetes 1.30 installed and configured
- Helm 3.x installed
- sudo/root access on the Linux server
- kubectl configured to access the cluster

## Steps

### Step 1: Install Required Linux Headers

Linux headers are required for Falco to function properly with your kernel.

```bash
sudo apt update
sudo apt install linux-headers-$(uname -r) -y
```

Verify installation:

```bash
uname -r
ls /usr/src/linux-headers-$(uname -r)
```

### Step 2: Add Falco Helm Repository

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

### Step 3: Create Falco Namespace

```bash
kubectl create ns falco
```

### Step 4: Install Falco

Deploy Falco as a DaemonSet in the Kubernetes cluster using Helm:

```bash
helm install falco falcosecurity/falco \
  --namespace falco \
  --version 4.0.0 \
  --set tty=true \
  --set collectors.containerd.enabled=true \
  --set collectors.containerd.socket=/run/k3s/containerd/containerd.sock
```

Wait for Falco pods to be ready:

```bash
kubectl wait pods --for=condition=Ready -l app.kubernetes.io/name=falco -n falco --timeout=150s
```

### Step 5: Install Falco Sidekick

Falco Sidekick receives events from Falco and forwards them to various outputs:

```bash
helm install falco-sidekick falcosecurity/falco-sidekick \
  --namespace falco
```

### Step 6: Install Falco Sidekick UI

Deploy the Sidekick UI for visualization:

```bash
helm install falco-sidekick-ui falcosecurity/falco-sidekick-ui \
  --namespace falco
```

### Step 7: Expose Sidekick UI

Convert the Sidekick UI service to NodePort for external access:

```bash
kubectl patch svc falco-sidekick-ui -n falco -p '{"spec": {"type": "NodePort"}}'
```

Retrieve the NodePort:

```bash
kubectl get svc falco-sidekick-ui -n falco
```

Access the UI using your server's IP and NodePort:

```
http://<server-ip>:<node-port>
```

### Step 8: Verify Falco Logs

Check that Falco is running and capturing events:

```bash
kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco
```

### Step 9: Customize Falco Configuration (Optional)

Edit the Falco configuration map:

```bash
kubectl edit cm falco-config -n falco
```

After editing, restart Falco to apply changes:

```bash
kubectl rollout restart daemonset falco -n falco
```

## Verification

1. Verify all components are running:

```bash
kubectl get pods -n falco
```

2. Access the Sidekick UI in your browser using the NodePort URL.

3. Monitor real-time Falco alerts as container runtime events are detected.

## Cleanup

To remove Falco, Sidekick, and Sidekick UI:

```bash
helm uninstall falco-sidekick-ui -n falco
helm uninstall falco-sidekick -n falco
helm uninstall falco -n falco
kubectl delete ns falco
```
