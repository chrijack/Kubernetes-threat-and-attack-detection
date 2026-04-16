# Lesson 28.5 — Falco Event Analysis for Kubernetes Security

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Learn how to install, configure, and use Falco to detect security issues in a Kubernetes cluster. This lesson covers important principles in rule configuration relevant to the CKS exam and provides real-world examples of detecting malicious behavior.

## Prerequisites

- Kubernetes cluster up and running
- kubectl configured and accessible
- Falco installed on the cluster nodes

## Steps

### Step 1: Understand Falco Rule Structure and Key Threats

Falco rules are defined in YAML files with these main components:

- **Rules**: Define conditions for alerts
- **Macros**: Reusable snippets for rule conditions
- **Lists**: Collections of items for use in rules

Key threats in a Kubernetes environment include:

- Privilege escalation
- Container breakouts
- Suspicious network activity
- File system tampering
- Unusual process execution

### Step 2: Locate and Modify Falco Rules

Review the main Falco rules file:

```bash
nano /etc/falco/falco_rules.yaml
```

List available syscall rules:

```bash
falco --list=syscall | grep name
```

Review the "Terminal shell in container" rule:

```yaml
- rule: Terminal shell in container
  desc: >
    A shell was used as the entrypoint/exec point into a container with an attached terminal. Parent process may have 
    legitimately already exited and be null (read container_entrypoint macro). Common when using "kubectl exec" in Kubernetes. 
    Correlate with k8saudit exec logs if possible to find user or serviceaccount token used (fuzzy correlation by namespace and pod name). 
    Rather than considering it a standalone rule, it may be best used as generic auditing rule while examining other triggered 
    rules in this container/tty.
  condition: >
    spawned_process 
    and container
    and shell_procs 
    and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: A shell was spawned in a container with an attached terminal (evt_type=%evt.type user=%user.name user_uid=%user.uid user_loginuid=%user.loginuid process=%proc.name proc_exepath=%proc.exepath parent=%proc.pname command=%proc.cmdline terminal=%proc.tty exe_flags=%evt.arg.flags %container.info)
  priority: NOTICE
  tags: [maturity_stable, container, shell, mitre_execution, T1059]
```

### Step 3: Create Rule Overrides

Open the local rules file for editing:

```bash
vim /etc/falco/falco_rules.local.yaml
```

Add the following content to modify the "Terminal shell in container" rule:

```yaml
- rule: Terminal shell in container
  condition: and not k8s.ns.name="dev"
  override:
    condition: append
```

Save and exit the file.

**Explanation:**

1. We use the `falco_rules.local.yaml` file to make modifications to existing rules. This is the recommended approach for customizing Falco rules.

2. The `override` section is used to modify an existing rule without duplicating the entire rule definition. This keeps custom modifications separate and easier to manage.

3. The `append` keyword adds an additional condition to the existing rule's condition, appending it to the end of the original.

4. The added condition `and not k8s.ns.name="dev"` excludes pods running in the "dev" namespace from triggering this rule. This allows developers to use terminal shells in containers within the dev namespace without generating alerts.

5. By placing this override in `falco_rules.local.yaml`:
   - The original rule in `falco_rules.yaml` remains untouched
   - Our modification takes precedence over the original rule
   - Our changes won't be overwritten during Falco updates

This approach follows best practices:

- Keeps custom changes separate from the default ruleset
- Is more maintainable and less error-prone than copying entire rules
- Allows for easier tracking and updating of modifications over time

Restart Falco to apply the changes:

```bash
systemctl restart falco-modern-bpf
```

Cordon node02 to make testing easier:

```bash
kubectl cordon worker-node02
```

In a split terminal, monitor Falco logs:

```bash
tail -f /var/log/syslog | grep falco
```

Create a test namespace and pod:

```bash
kubectl create namespace dev
kubectl run test-pod --image=busybox -n dev -- sleep 3600
kubectl exec -it test-pod -n dev -- /bin/sh
```

Create a pod in the default namespace:

```bash
kubectl run default-pod --image=busybox -- sleep 3600
kubectl exec -it default-pod -- /bin/sh
```

### Step 4: Create Custom Macros and Rules

Create a macro to detect suspicious network activity:

```yaml
- macro: suspicious_outbound_connection
  condition: >
    (((evt.type = connect and evt.dir=<) or
      (evt.type in (sendto,sendmsg) and evt.dir=< and
       fd.l4proto != tcp and fd.connected=false and fd.name_changed=true)) and
     (fd.typechar = 4 or fd.typechar = 6) and
     (fd.ip != "0.0.0.0" and fd.net != "127.0.0.0/8" and not fd.snet in (rfc_1918_addresses)) and
     (evt.rawres >= 0 or evt.res = EINPROGRESS))
```

Create a rule using the macro:

```yaml
- rule: Suspicious Outbound Connection from Container
  desc: Detect outbound connections to potentially suspicious ports
  condition: suspicious_outbound_connection and k8s.ns.name != ""
  output: "Suspicious outbound connection detected (pod=%k8s.pod.name namespace=%k8s.ns.name srcip=%fd.cip dstip=%fd.sip dstport=%fd.sport proto=%fd.l4proto procname=%proc.name)"
  priority: WARNING
  tags: [network, container, mitre_command_and_control]
```

Use lists for flexible rules:

```yaml
- list: allowed_processes
  items: [nginx, apache2, mysql, sleep, cilium-envoy]

- rule: Unexpected Process in Container
  desc: Detect execution of unexpected processes in containers
  condition: spawned_process and container and not proc.name in (allowed_processes)
  output: "Unexpected process (%proc.name) detected in pod (pod=%k8s.pod.name) container (id=%container.id)"
  priority: WARNING
  tags: [process, container, mitre_execution]
```

### Step 5: Copy Modified Rules to Local File

To prevent updates from overwriting changes, copy modified rules to `falco_rules.local.yaml`:

```bash
vim /etc/falco/falco_rules.local.yaml
```

Add the modified and custom rules to this file.

### Step 6: Create Rules for Real-World Threat Scenarios

#### Detecting File Tampering

```yaml
- rule: Detect File Write to /etc/passwd
  desc: Detect attempts to modify /etc/passwd
  condition: open_write and container and fd.name=/etc/passwd
  output: "File write to /etc/passwd detected (user=%user.name file=%fd.name command=%proc.cmdline)"
  priority: CRITICAL
  tags: [filesystem, sensitive]
```

#### Detecting Malicious Script Downloads

```yaml
- rule: Detect Malicious Script Download
  desc: Detect curl or wget downloading files
  condition: container and (proc.name = "curl" or proc.name = "wget")
  output: "File download attempt detected (user=%user.name command=%proc.cmdline)"
  priority: CRITICAL
  tags: [network, download]
```

### Step 7: Test the Rules

Start Falco:

```bash
systemctl start falco
```

Monitor Falco output:

```bash
tail -f /var/log/falco.log
```

Trigger the rules to test detection:

```bash
# For the shell rule
kubectl run test-pod --image=busybox -- sh -c "while true; do sleep 30; done"
kubectl exec -it test-pod -- /bin/sh

# For the suspicious connection rule
kubectl run netcat-pod --image=busybox -- nc -zv 93.184.216.34 80

# For the unexpected process rule
kubectl run process-pod --image=nginx
kubectl exec -it process-pod -- apt-get update

# For the file tampering rule
kubectl exec -it -n dev test-pod -- /bin/sh -c "echo testuser:x:0:0::/root:/bin/bash >> /etc/passwd"

# For the malicious script download rule
kubectl exec -it -n dev test-pod -- /bin/sh -c "wget http://example.com"
```

### Step 8: Review and Refine

After testing, review the Falco logs and refine the rules as needed. Consider:

- Adjusting rule priorities
- Fine-tuning conditions to reduce false positives
- Adding exceptions for legitimate activities

## Verification

Verify that rules are functioning correctly by:

1. Monitoring Falco logs for expected alerts when trigger conditions are met
2. Checking that overridden rules behave as expected (e.g., dev namespace exclusion)
3. Confirming custom rules fire for their intended threat scenarios
4. Validating that benign activities do not trigger false positives

## Cleanup

After testing, clean up the test resources:

```bash
kubectl delete namespace dev
kubectl delete pod test-pod default-pod process-pod netcat-pod
kubectl uncordon worker-node02
systemctl stop falco
```

## Summary

This comprehensive guide demonstrates key techniques for configuring and using Falco to detect security issues in Kubernetes:

- Understanding Falco rule structure (rules, macros, lists)
- Locating and editing existing rules
- Creating rule overrides and appends
- Creating custom macros and rules
- Using lists for flexible rule definitions
- Detecting real-world threat scenarios (file tampering, malicious downloads, unexpected processes, suspicious connections)
- Testing and refining rules

**Key Takeaways:**

- Focus on understanding rule syntax and structure
- Create rules that target specific security concerns in Kubernetes
- Use macros and lists effectively
- Balance detection capability with performance impact
- Always use `falco_rules.local.yaml` for customizations to preserve upgradability

By mastering these techniques, you'll be well-prepared for the Falco-related portions of the CKS exam and equipped to enhance Kubernetes security in real-world scenarios.
