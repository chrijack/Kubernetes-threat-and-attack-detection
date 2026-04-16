# Lesson 1.3 — Enforce Immutability with Validating Admission Policy

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Enforce immutability on container filesystems using Kubernetes' Validating Admission Policy (VAP) with Kubescape. This ensures that containers in your cluster run with a read-only root filesystem, enhancing security by preventing unauthorized changes.

## Prerequisites

- Kubernetes cluster with VAP support enabled
- kubectl access to the cluster
- Kubescape CEL Admission Library (for optional advanced configurations)

## Steps

### Step 1: Create the Validating Admission Policy

Define a `ValidatingAdmissionPolicy` to deny the creation or update of any Pods or workload resources that do not specify a read-only root filesystem.

```bash
kubectl apply -f - <<EOT
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "kubescape-c-0017-deny-resources-with-mutable-container-filesystem"
  labels:
    controlId: "C-0017"
  annotations:
    controlUrl: "https://hub.armosec.io/docs/c-0017"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   [""]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["pods"]
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments","replicasets","daemonsets","statefulsets"]
    - apiGroups:   ["batch"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["jobs","cronjobs"]
  validations:
    - expression: "object.kind != 'Pod' || object.spec.containers.all(container, has(container.securityContext) && has(container.securityContext.readOnlyRootFilesystem) &&  container.securityContext.readOnlyRootFilesystem == true)"
      message: "Pods having containers with mutable filesystem not allowed! (see more at https://hub.armosec.io/docs/c-0017)"
    - expression: "['Deployment','ReplicaSet','DaemonSet','StatefulSet','Job'].all(kind, object.kind != kind) || object.spec.template.spec.containers.all(container, has(container.securityContext) && has(container.securityContext.readOnlyRootFilesystem) &&  container.securityContext.readOnlyRootFilesystem == true)"
      message: "Workloads having containers with mutable filesystem not allowed! (see more at https://hub.armosec.io/docs/c-0017)"
    - expression: "object.kind != 'CronJob' || object.spec.jobTemplate.spec.template.spec.containers.all(container, has(container.securityContext) && has(container.securityContext.readOnlyRootFilesystem) &&  container.securityContext.readOnlyRootFilesystem == true)"
      message: "CronJob having containers with mutable filesystem not allowed! (see more at https://hub.armosec.io/docs/c-0017)"
EOT
```

This policy checks that all containers in Pods, Deployments, ReplicaSets, DaemonSets, StatefulSets, Jobs, and CronJobs have `readOnlyRootFilesystem` set to `true`. If not met, resource creation or update is denied.

### Step 2: Create the Policy Binding

Bind the policy to namespaces labeled with `policy=enforced`.

```bash
kubectl apply -f - <<EOT
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: c0017-binding
spec:
  policyName: kubescape-c-0017-deny-resources-with-mutable-container-filesystem
  paramRef:
    name: basic-control-configuration
    parameterNotFoundAction: Deny
  validationActions:
  - Deny
  matchResources:
    namespaceSelector:
      matchLabels:
        policy: enforced
EOT
```

This binding ensures the policy is applied to any namespace with the label `policy=enforced`.

### Step 3: Create and Label a Test Namespace

```bash
kubectl create namespace policy-example
kubectl label namespace policy-example 'policy=enforced'
```

### Step 4: Test Policy Enforcement (Expected to Fail)

Attempt to create a Pod without specifying `readOnlyRootFilesystem`:

```bash
kubectl -n policy-example run nginx --image=nginx --restart=Never
```

Expected error:
```
The pods "nginx" is invalid: : ValidatingAdmissionPolicy 'kubescape-c-0017-deny-mutable-container-filesystem' with binding 'c0017-binding' denied request: Pods having containers with mutable filesystem not allowed! (see more at https://hub.armosec.io/docs/c-0017)
```

This confirms the policy is correctly blocking non-compliant resources.

### Step 5: Create a Compliant Pod

Create a Pod that complies with the policy by setting `readOnlyRootFilesystem` to `true`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: read-only-pod
  namespace: policy-example
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

This configuration adheres to the policy by ensuring the root filesystem is read-only while allowing writable directories as needed by the application.

### Step 6: Install the Kubescape CEL Library (Optional)

To explore more policy configurations, install the Kubescape CEL Admission Library:

```bash
kubectl apply -f https://github.com/kubescape/cel-admission-library/releases/latest/download/policy-configuration-definition.yaml
kubectl apply -f https://github.com/kubescape/cel-admission-library/releases/latest/download/basic-control-configuration.yaml
kubectl apply -f https://github.com/kubescape/cel-admission-library/releases/latest/download/kubescape-validating-admission-policies.yaml
```

These commands install the necessary CustomResourceDefinitions (CRDs) and policies from the Kubescape CEL library.

## Verification

- Pod without `readOnlyRootFilesystem` is rejected by the policy
- Compliant Pod with `readOnlyRootFilesystem: true` is created successfully
- Verify with: `kubectl -n policy-example get pods`

## Cleanup

```bash
kubectl delete namespace policy-example
kubectl delete validatingadmissionpolicy kubescape-c-0017-deny-resources-with-mutable-container-filesystem
kubectl delete validatingadmissionpolicybinding c0017-binding
```

## Resources

- [Kubescape Validating Admission Policy Library](https://kubernetes.io/blog/2023/03/30/kubescape-validating-admission-policy-library/)
- [Kubescape CEL Admission Library](https://github.com/kubescape/cel-admission-library)
