# wazuh-siem-lab-setup-and-incident-handling
A self-built Wazuh SIEM/XDR home lab deployment on VMware Ubuntu Server, including setup and system troubleshooting.



# Wazuh SIEM/XDR Central Node Deployment & Infrastructure Engineering

![Wazuh Version](https://img.shields.io/badge/Wazuh-v4.14.7-blue.svg)
![OS](https://img.shields.io/badge/OS-Ubuntu%20Server%20LTS-orange.svg)
![Deployment Status](https://img.shields.io/badge/Cluster%20Status-Operational-brightgreen.svg)
![Security Focus](https://img.shields.io/badge/Focus-Blue%20Team%20%7C%20SOC%20Engineering-red.svg)

An enterprise-grade deployment and low-level Linux systems hardening report for an on-premises **Wazuh SIEM/XDR All-in-One central cluster**. This repository documents architectural setup, live storage triage under exhaustion, and custom Debian package surgery to achieve a resilient security monitoring baseline.

---

## Architecture Topology


```

+-------------------------------------------------------------------------+
|                       Management Workstation (Host)                     |
|           SOC Web Console -> HTTPS / TCP Port 443 [Dashboard]           |
+-------------------------------------------------------------------------+
|
                        [Isolated Virtual Lab Subnet]
|
+------------------------------------v------------------------------------+
|                      Wazuh Central Processing Node                      |
|                                                                         |
|  +------------------------+                  +-----------------------+  |
|  |     Wazuh Manager      |--[Event Data]--> |       Filebeat        |  |
|  | (Analysis & Detection) |                  | (TLS Log Forwarder)   |  |
|  +------------------------+                  +-----------------------+  |
|                                                          |              |
|                                                      [TLS/REST]         |
|                                                          v              |
|  +------------------------+                  +-----------------------+  |
|  |    Wazuh Dashboard     |<--[REST/JSON]----|     Wazuh Indexer     |  |
|  |  (Visualization UI)    |                  |  (OpenSearch Engine)  |  |
|  +------------------------+                  +-----------------------+  |
+-------------------------------------------------------------------------+

```

---

## Node Specifications

* **Platform:** Type-2 Hypervisor (VMware Workstation, NAT Isolation)
* **Operating System:** Linux Server (Ubuntu LTS)
* **Compute Allocation:** 8 vCPUs | 7.0 GB RAM
* **Storage Configuration:** 50 GB Dynamically Resized LVM (`/dev/ubuntu-vg/ubuntu-lv`)
* **Software Core:** Wazuh Central Stack v4.14.7 (Indexer, Manager, Filebeat, Dashboard)

---

## Infrastructure Initialization

### 1. Cryptographic Key Authentication & Repository Staging
```bash
# Securely register upstream signing keys
curl -s [https://packages.wazuh.com/key/GPG-KEY-WAZUH](https://packages.wazuh.com/key/GPG-KEY-WAZUH) | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# Add stable repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] [https://packages.wazuh.com/4.x/apt/](https://packages.wazuh.com/4.x/apt/) stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt-get update -y

```

### 2. Orchestration Script

```bash
curl -sO [https://packages.wazuh.com/4.14/wazuh-install.sh](https://packages.wazuh.com/4.14/wazuh-install.sh)

```

---

## Root Cause Analysis (RCA) & Incident Triage

During orchestration, two blocking pipeline failures required manual system intervention and package engineering.

```
[Pipeline Interruption]
Storage Depletion ──> LVM Online Expansion ──> DPKG Maintainer Script Abortion ──> Debian Binary Surgery

```

### Case ID 01: Storage Boundary Exhaustion (`write (28: No space left on device)`)

* **Impact:** Critical write failure during package unpack stage.
* **Root Cause:** Concurrent buffering of OpenSearch database binaries, server detection rule definitions, and `.deb` archives saturated the default 20 GB allocation.
* **Remediation (Live LVM Dynamic Expansion):**
```bash
# Resize disk partition boundary and expand LVM physical volume online
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3

# Dynamically extend logical volume and expand filesystem
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv

```



---

### Case ID 02: DPKG Maintainer Script Abort (`Exit Status 127`)

* **Impact:** Subprocess termination during unpack stage:
```text
dpkg: error processing archive .../wazuh-manager.deb (--unpack):
 new wazuh-manager package preinst maintainer script subprocess failed with exit status 127

```


* **Root Cause:** The upstream pre-installation routine aborted due to uninitialized OSSEC daemon security contexts, which prevented deployment of critical components (`wazuh-keystore`, `ossec.conf`).
* **Remediation (Binary Surgery & Local Cache Injection):**
1. **Provision OSSEC service identities manually:**
```bash
sudo groupadd -r ossec 2>/dev/null || true
sudo groupadd -r wazuh 2>/dev/null || true
sudo useradd -r -g ossec -G wazuh -d /var/ossec -s /sbin/nologin ossec 2>/dev/null || true
sudo useradd -r -g ossec -G wazuh -d /var/ossec -s /sbin/nologin ossecm 2>/dev/null || true
sudo useradd -r -g ossec -G wazuh -d /var/ossec -s /sbin/nologin ossecr 2>/dev/null || true
sudo useradd -r -g wazuh -d /var/ossec -s /sbin/nologin wazuh 2>/dev/null || true

```


2. **Extract, strip failing `preinst` triggers, and repack package archive:**
```bash
cd /tmp
cp /var/cache/apt/archives/wazuh-manager_*_amd64.deb .
sudo dpkg-deb -R wazuh-manager_*_amd64.deb /tmp/wazuh-extracted
sudo rm -f /tmp/wazuh-extracted/DEBIAN/preinst
sudo dpkg-deb -b /tmp/wazuh-extracted /tmp/wazuh-manager-fixed.deb
sudo dpkg -i --force-all /tmp/wazuh-manager-fixed.deb

```


3. **Inject sanitized artifact into local APT archive cache:**
```bash
sudo cp /tmp/wazuh-manager-fixed.deb /var/cache/apt/archives/wazuh-manager_4.14.7-1_amd64.deb
sudo rm -f /var/lib/dpkg/info/wazuh-manager.prerm /var/lib/dpkg/info/wazuh-manager.postrm

```





---

## Cluster Execution & Operational Verification

With upstream bottlenecks patched, the automated installation was executed using overwrite parameters:

```bash
sudo systemctl stop wazuh-manager 2>/dev/null || true
sudo bash ./wazuh-install.sh -a -i -o

```

### Daemon Audit & Service Matrix

| Daemon Service | Layer / Core Function | Network Binding | Health Check |
| --- | --- | --- | --- |
| `wazuh-indexer` | Log Ingestion & Analytics Engine | `TCP/9200` | **active (running)** |
| `wazuh-manager` | Analysis Engine & Rule Processor | `TCP/1514-1515`, `TCP/55000` | **active (running)** |
| `filebeat` | Secure Pipeline Log Forwarder | Unix Socket / Pipeline | **active (running)** |
| `wazuh-dashboard` | SOC Visual Interface & TLS Console | `TCP/443 (HTTPS)` | **active (running)** |

### Baseline Configuration & Recovery Posture

* **Configuration Safeguard:** Baseline configuration preserved at `/var/ossec/etc/ossec.conf.bak`.
* **State Preservation:** Clean hypervisor-level snapshot captured to maintain an immutable rollback baseline.

---





