# Lesson 29.3 — Identifying Anomalies and Suspicious Patterns in Kubernetes Audit Logs

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Learn how to identify anomalous and suspicious patterns in Kubernetes audit logs and map them to the MITRE ATT&CK framework for security threat detection.

## Prerequisites

- Access to Kubernetes cluster audit logs
- Understanding of Kubernetes API audit events
- Familiarity with MITRE ATT&CK framework concepts

## Steps

### Step 1: Unusual Login Times or Locations

Identify login attempts from unexpected sources or times that may indicate unauthorized access.

**Log Example:**
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "user": {
    "username": "admin",
    "sourceIP": "203.0.113.5"
  },
  "verb": "get",
  "requestURI": "/api/v1/namespaces/default/pods",
  "stage": "ResponseComplete",
  "timestamp": "2023-10-04T03:05:01Z"
}
```

**MITRE ATT&CK Mapping:** Initial Access

**Action:** Investigate the context around the IP address and user. An unusual login from an IP address not typically used, or during off-hours, may indicate unauthorized access.

### Step 2: Multiple Failed Login Attempts

Detect brute-force attacks by monitoring failed authentication attempts from the same source.

**Log Example:**
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "status": {
    "code": 401,
    "reason": "Unauthorized"
  },
  "user": {
    "username": "unknown",
    "sourceIP": "203.0.113.8"
  },
  "verb": "create",
  "requestURI": "/api/v1/namespaces/default/pods",
  "stage": "ResponseComplete",
  "timestamp": "2023-10-04T03:10:23Z"
}
```

**MITRE ATT&CK Mapping:** Credential Access

**Action:** Multiple failed login attempts from the same IP can be a sign of a brute-force attack. Monitor and possibly block the IP and investigate the intent.

### Step 3: Abnormal Administrative Actions

Identify suspicious operations by users with elevated privileges, such as deletion of critical objects.

**Log Example:**
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "user": {
    "username": "admin"
  },
  "verb": "delete",
  "requestURI": "/api/v1/namespaces/default/secrets/db-secret",
  "stage": "ResponseComplete",
  "timestamp": "2023-10-04T03:15:43Z"
}
```

**MITRE ATT&CK Mapping:** Privilege Escalation, Defense Evasion

**Action:** Deleting secrets or any other critical object must be immediately investigated, especially if there's no scheduled activity to justify it.

### Step 4: Unexplained Configuration Changes

Monitor for unauthorized modifications to ConfigMaps and other Kubernetes objects.

**Log Example:**
```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "user": {
    "username": "dev"
  },
  "verb": "patch",
  "requestURI": "/api/v1/namespaces/default/configmaps/app-config",
  "stage": "ResponseComplete",
  "timestamp": "2023-10-04T03:20:30Z"
}
```

**MITRE ATT&CK Mapping:** Defense Evasion, Discovery

**Action:** Any changes to a ConfigMap or other Kubernetes objects that weren't part of scheduled updates or activities should be considered suspicious.

## Verification

Review audit logs in your cluster to confirm you can identify and categorize each anomaly type according to the MITRE ATT&CK framework. Cross-reference suspicious events with your change management records to identify legitimate vs. unauthorized activities.

## Cleanup

No additional cleanup is required for this lesson. Audit logs are automatically managed by your Kubernetes cluster.
