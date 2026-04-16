# Lesson 28.3 — Falco Linux Installation and Runtime Security

> From the [Certified Kubernetes Security Specialist (CKS) Video Course](https://www.pearsonitcertification.com/store/certified-kubernetes-security-specialist-cks-video-9780138296476)

## Objective

Install Falco on a Linux system, verify it's running correctly, inspect default security rules, trigger a test event, and review the generated security alerts.

## Prerequisites

- Linux system with apt package manager (Debian/Ubuntu-based)
- sudo access
- Internet connectivity to download Falco packages and GPG keys
- Basic familiarity with command-line operations

## Steps

### Step 1: Trust Falco GPG Keys

```bash
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
```

This fetches Falco's official GPG key and saves it to your system's keyring to ensure package authenticity.

### Step 2: Add Falco APT Repository

```bash
sudo bash -c 'cat << EOF > /etc/apt/sources.list.d/falcosecurity.list
deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main
EOF'
```

Adds the Falco APT repository to enable package installation via apt.

### Step 3: Update Package Lists

```bash
sudo apt-get update -y
```

Updates your system's package list to recognize the new Falco repository.

### Step 4: Install Required Linux Headers

```bash
sudo apt-get install -y dkms make linux-headers-$(uname -r) dialog
```

Installs essential build tools and kernel headers needed for Falco kernel module compilation.

### Step 5: Install Falco

```bash
sudo apt-get install -y falco
```

Installs Falco and prompts for automatic driver selection during installation.

### Step 6: List Running Services

```bash
systemctl --type=service | grep "falco"
```

Lists systemd services and filters for Falco to verify the service is registered.

### Step 7: Check Falco Service Status

```bash
sudo systemctl status falco-modern-bpf
```

Verifies that the Falco service is running correctly.

### Step 8: List Falco Configuration Directory

```bash
ls /etc/falco
```

Lists the contents of the Falco configuration directory.

### Step 9: Navigate to Falco Rules Directory

```bash
cd /etc/falco/
```

### Step 10: Inspect Default Falco Rules

```bash
nano falco_rules.yaml
```

Opens the default Falco rules file for review. These rules define system behaviors that Falco monitors.

### Step 11: Trigger a Security Event

```bash
sudo cat /etc/shadow > /dev/null
```

Reads the sensitive `/etc/shadow` file, which should trigger a Falco security alert.

### Step 12: Review Falco Logs

```bash
sudo journalctl _COMM=falco -p warning
```

Displays Falco warning-level events from systemd journal.

### Step 13: Check System Log for Alerts

```bash
sudo grep Sensitive /var/log/syslog
```

Searches the system log for sensitive operation alerts detected by Falco.

## Verification

- Falco service is active: `sudo systemctl status falco-modern-bpf`
- Alert was logged: `sudo journalctl _COMM=falco -p warning` shows the /etc/shadow access event
- Syslog contains alert: `sudo grep Sensitive /var/log/syslog` returns matching entries

## Cleanup

To stop Falco:

```bash
sudo systemctl stop falco-modern-bpf
```

To uninstall Falco:

```bash
sudo apt-get remove -y falco
```
