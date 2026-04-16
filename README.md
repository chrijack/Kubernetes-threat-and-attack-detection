# Kubernetes Threat and Attack Detection - Demo Scripts

**Course:** Kubernetes Threat and Attack Detection (Video Course)  
**Author:** Chris Jackson, CCIEx2 #6256 — Distinguished Architect, Cisco  
**Publisher:** Pearson IT Certification  
**Published:** April 4, 2025  
**ISBN-10:** 0-13-544724-0  
**ISBN-13:** 978-0-13-544724-6  

This repository contains demo scripts and lab exercises from the **Kubernetes Threat and Attack Detection** video course. The course covers robust strategies for detecting, analyzing, and responding to malicious activities and potential security breaches in Kubernetes environments — including container immutability, audit logging, runtime threat detection with Falco, and compromise investigation. It is part of the larger **Kubernetes Security Essentials (Video Collection)**.

For the complete CKS (Certified Kubernetes Security Specialist) course materials, see the [CKS Demo Scripts repository](https://github.com/chrijack/CKS-demo-scripts).

---

## Course Content

All lesson materials are located in the `lessons/` directory.

### Lesson 1: Ensure Immutability of Containers at Runtime

| Lesson | Topic | Link |
|--------|-------|------|
| 1.2 | Read-Only File System in Kubernetes Pods | [lessons/lesson-1.2/lesson-1.2.md](lessons/lesson-1.2/lesson-1.2.md) |
| 1.3 | Enforce Immutability with Validating Admission Policy | [lessons/lesson-1.3/lesson-1.3.md](lessons/lesson-1.3/lesson-1.3.md) |

### Lesson 2: Use Kubernetes Audit Logs to Monitor Access

| Lesson | Topic | Link |
|--------|-------|------|
| 2.2 | Kubernetes Audit Policy Configuration | [lessons/lesson-2.2/lesson-2.2.md](lessons/lesson-2.2/lesson-2.2.md) |
| 2.3 | Audit Logging Tuning and Batch Parameters | [lessons/lesson-2.3/lesson-2.3.md](lessons/lesson-2.3/lesson-2.3.md) |
| 2.4 | Kubernetes Audit Logging with Grafana Loki | [lessons/lesson-2.4/lesson-2.4.md](lessons/lesson-2.4/lesson-2.4.md) |

### Lesson 3: Detect Malicious Activity, Threats, and Attacks

| Lesson | Topic | Link |
|--------|-------|------|
| 3.1 | System Call Monitoring with strace, sysdig, and SCAP | [lessons/lesson-3.1/lesson-3.1.md](lessons/lesson-3.1/lesson-3.1.md) |
| 3.3 | Falco Linux Installation and Runtime Security | [lessons/lesson-3.3/lesson-3.3.md](lessons/lesson-3.3/lesson-3.3.md) |
| 3.4 | Installing Falco with Sidekick UI on Kubernetes 1.30 | [lessons/lesson-3.4/lesson-3.4.md](lessons/lesson-3.4/lesson-3.4.md) |
| 3.5 | Falco Event Analysis for Kubernetes Security | [lessons/lesson-3.5/lesson-3.5.md](lessons/lesson-3.5/lesson-3.5.md) |

### Lesson 4: Investigate and Identify Signs of Compromise

| Lesson | Topic | Link |
|--------|-------|------|
| 4.1 | MITRE Container Framework Demo | [lessons/lesson-4.1/lesson-4.1.md](lessons/lesson-4.1/lesson-4.1.md) |
| 4.2 | Kubernetes Security Monitoring with Loki, Prometheus, and Grafana | [lessons/lesson-4.2/lesson-4.2.md](lessons/lesson-4.2/lesson-4.2.md) |
| 4.3 | Identifying Anomalies and Suspicious Patterns in Kubernetes Audit Logs | [lessons/lesson-4.3/lesson-4.3.md](lessons/lesson-4.3/lesson-4.3.md) |

---

## Prerequisites

- Basic understanding of Kubernetes concepts and architecture
- Kubernetes cluster access (local or cloud-based)
- kubectl command-line tool
- Docker or container runtime knowledge
- Linux/Unix command-line familiarity

---

## Repository Structure

```
.
├── README.md
├── .gitignore
├── lessons/
│   ├── lesson-1.2/
│   ├── lesson-1.3/
│   ├── lesson-2.2/
│   ├── lesson-2.3/
│   ├── lesson-2.4/
│   ├── lesson-3.1/
│   ├── lesson-3.3/
│   ├── lesson-3.4/
│   ├── lesson-3.5/
│   ├── lesson-4.1/
│   ├── lesson-4.2/
│   └── lesson-4.3/
```

---

## Course Information

**Course URL:** https://www.pearsonitcertification.com/store/kubernetes-threat-and-attack-detection-video-course-9780135447246

**Description:** This course covers robust strategies for detecting, analyzing, and responding to malicious activities and potential security breaches in Kubernetes environments — including container immutability, audit logging, runtime threat detection with Falco, and compromise investigation. Part of the Kubernetes Security Essentials (Video Collection).

---

## License

MIT License

Copyright (c) 2025 Chris Jackson

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

**Attribution Required:** Demo scripts from the [Kubernetes Threat and Attack Detection (Video Course)](https://www.pearsonitcertification.com/store/kubernetes-threat-and-attack-detection-video-course-9780135447246) by Chris Jackson, published by Pearson IT Certification (ISBN: 978-0-13-544724-6).

---

## Related Resources

- [CKS Demo Scripts Repository](https://github.com/chrijack/CKS-demo-scripts) - Complete Certified Kubernetes Security Specialist course materials
- [Kubernetes Security Essentials (Video Collection)](https://www.pearsonitcertification.com) - Full video collection by Pearson IT Certification
