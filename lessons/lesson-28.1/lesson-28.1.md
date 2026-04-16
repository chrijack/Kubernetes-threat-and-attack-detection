# Lesson 28.1 — System Call Monitoring with strace, sysdig, and SCAP

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Demonstrate how to monitor and analyze system calls made by processes and containers using strace, sysdig, and SCAP (via Falco). Learn to interpret syscall output and filter for specific system call types to detect process behavior and security events.

## Prerequisites

- Linux host with sudo access
- kubectl configured to access a Kubernetes cluster
- Basic understanding of Linux processes and file descriptors

## Steps

### Step 1: Install System Call Monitoring Tools

#### Install strace:
```bash
sudo apt update
sudo apt install strace
```

#### Install sysdig:
```bash
apt-get update
sudo apt-get -y install linux-headers-$(uname -r)
apt install dkms sysdig
modprobe sysdig-probe
```

#### Install Falco (for SCAP):
```bash
curl -s https://falco.org/repo/falcosecurity-packages.asc | sudo apt-key add -
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://download.falco.org/packages/deb stable main"
sudo apt-get update
sudo apt-get install -y falco
```

### Step 2: Deploy a Test Pod

Deploy a simple busybox pod in Kubernetes for demonstration:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: syscall-demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: [ "sleep", "3600" ]
EOF
```

Verify the pod is running:

```bash
kubectl get pods -o wide
```

### Step 3: Using strace for Process-Level System Call Monitoring

#### Access the pod and install strace:
```bash
kubectl exec -it syscall-demo -- sh
apk add strace
```

#### Run strace on a simple command:
```bash
strace ls
```

#### Expected output:
```bash
execve("/bin/ls", ["ls"], 0x7ffe3b54f0c0 /* 22 vars */) = 0
brk(NULL)                               = 0x55c2b5808000
arch_prctl(ARCH_SET_FS, 0x7f0b53cbedc0) = 0
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, /* 18 entries */, 32768)  = 456
close(3)                                = 0
exit_group(0)                           = 0
```

#### Understanding the output:

- **`execve("/bin/ls", ["ls"], ...)`**: Loads the ls binary into memory. Return value 0 indicates success.
- **`brk(NULL)`**: Memory management syscall that allocates or frees memory by setting the program's data segment.
- **`openat(AT_FDCWD, ".", ...)`**: Opens the current directory in read-only mode. Return value 3 is the file descriptor assigned.
- **`getdents64(3, ...)`**: Reads directory entries from file descriptor 3. Returns 456 bytes of data.
- **`close(3)`**: Closes file descriptor 3. Return value 0 indicates success.
- **`exit_group(0)`**: Exits the process with success status.

#### Filter for specific syscalls:
```bash
strace -e openat ls
```

This shows only openat syscalls:
```bash
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
```

#### Save strace output to a file:
```bash
strace -o output.txt ls
```

### Step 4: Using sysdig for Container-Level System Call Monitoring

#### Capture syscalls in real-time from the pod:
```bash
sudo sysdig -pc -k 'container.name=syscall-demo'
```

Flags:
- `-p`: Custom output format for syscalls
- `-c`: Show container information
- `-k 'container.name=syscall-demo'`: Filter for the specific container

#### Expected output:
```bash
15:00:01.123456880 5 syscall-demo (ls) > open fd=3 dirflags=0x0 path=.
15:00:01.123456880 5 syscall-demo (ls) < open fd=3 <res=0>
15:00:01.123456880 5 syscall-demo (ls) > read fd=3 size=1024
15:00:01.123456880 5 syscall-demo (ls) < read res=1024 data="..."
15:00:01.123456880 5 syscall-demo (ls) > close fd=3
```

#### Save syscall capture to a file for offline analysis:
```bash
sudo sysdig -w capture.scap -pc -k 'container.name=syscall-demo'
```

#### Read a saved capture file:
```bash
sudo sysdig -r capture.scap
```

#### Filter for specific syscall types:

File access syscalls:
```bash
sudo sysdig -pc -k 'container.name=syscall-demo and evt.type=open'
```

Network syscalls:
```bash
sudo sysdig -pc -k 'container.name=syscall-demo and evt.type=connect'
```

### Step 5: Using Falco/SCAP for Security-Focused System Call Monitoring

Falco uses SCAP to capture and analyze syscalls against security rules. Edit the Falco rules to detect sensitive file access:

#### Example rule in `/etc/falco/falco_rules.yaml`:
```yaml
- rule: Open Sensitive Files
  desc: Detect attempts to open sensitive system files
  condition: evt.type=openat and fd.name in ("/etc/shadow", "/etc/passwd")
  output: Sensitive file opened: %fd.name
  priority: CRITICAL
```

#### Restart Falco to apply the rule:
```bash
sudo systemctl restart falco
```

#### Monitor Falco alerts in real-time:
```bash
sudo tail -f /var/log/falco/falco.log
```

#### Expected output when sensitive files are accessed:
```bash
Sensitive file opened: /etc/shadow
Sensitive file opened: /etc/passwd
```

## Verification

### Test strace with different syscall types:

```bash
# Monitor file operations
strace -e openat,read,write,close ls

# Monitor process operations
strace -e fork,exec,exit_group sleep 1

# Monitor network operations
strace -e socket,connect,bind nc localhost 8080
```

### Verify sysdig is capturing container syscalls:

```bash
# Get real-time syscall count for a container
sudo sysdig -pc 'container.name=syscall-demo' | wc -l
```

### Verify Falco is detecting rules:

```bash
# Check Falco service status
sudo systemctl status falco

# Trigger a sensitive file access detection
kubectl exec -it syscall-demo -- cat /etc/passwd

# Check logs for the alert
sudo tail -f /var/log/falco/falco.log
```

## Cleanup

```bash
# Delete the test pod
kubectl delete pod syscall-demo

# Stop Falco if not needed
sudo systemctl stop falco
```

## Key Syscalls Reference

| Syscall | Purpose |
|---------|---------|
| `execve` | Execute a program |
| `openat` | Open a file or directory |
| `read` | Read from a file descriptor |
| `write` | Write to a file descriptor |
| `close` | Close a file descriptor |
| `fork` | Create a new process |
| `exit_group` | Exit a process |
| `socket` | Create a network socket |
| `connect` | Establish a network connection |
| `getdents64` | Read directory entries |
| `brk` | Allocate/deallocate memory |

## Tool Comparison

| Tool | Scope | Use Case |
|------|-------|----------|
| **strace** | Single process | Detailed syscall tracking for specific programs |
| **sysdig** | Container/system-wide | Broader syscall monitoring and forensics |
| **Falco/SCAP** | Security policies | Detect anomalous or security-relevant syscall patterns |
