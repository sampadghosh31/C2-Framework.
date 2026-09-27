# Multi-Channel C2 Framework & Automated Host Assessor

A modular Command and Control (C2) framework developed as a final-year cybersecurity capstone project at **KIIT University**. The platform demonstrates modern defensive evasion strategies, resilient multi-channel covert communications (including Cloud APIs and DNS Tunneling), dynamic agent payload deployment, and automated target host vulnerability auditing.

---

## 📌 Executive Summary

Modern enterprise networks deploy strict perimeter controls, network intrusion detection systems (NIDS), and egress filtering. Traditional adversary infrastructure relying on static IP addresses or unrated domains is quickly identified and blocked.

This framework investigates how post-exploitation agents maintain persistent command-and-control channels by leveraging:
1. **High-Trust Egress Pipe Abuse:** Interacting with trusted cloud services (Google Drive and Microsoft OneDrive APIs) over standard HTTPS (Port 443).
2. **Covert Fallback Transport:** Encapsulating data within raw DNS queries over UDP Port 53 to bypass strict egress firewalls.
3. **Automated Post-Exploitation Auditing:** Processing incoming system profiling data through automated local patch assessment engines (WES-NG) upon initial check-in.

---

## 🏗️ System Architecture

The architecture consists of a Linux-based C Controller infrastructure managing remote Windows targets executing PowerShell agent modules.

```
                            +-----------------------------------+
                            |    C2 Controller (Linux Server)   |
                            |   - CLI Interface / Module Engine |
                            +-----------------+-----------------+
                                              |
         +------------------------------------+------------------------------------+
         |                                    |                                    |
         v                                    v                                    v
[ Google Drive API ]                 [ OneDrive API ]                   [ Authoritative DNS ]
  HTTPS (Port 443)                    HTTPS (Port 443)                 UDP Port 53 (A / TXT)
         |                                    |                                    |
         +------------------------------------+------------------------------------+
                                              |
                                              v
                            +-----------------------------------+
                            |     Victim Workstation Agent      |
                            |       (PowerShell Execution)      |
                            +-----------------------------------+
```

---

## 📡 Communication Channel Mechanics

### 1. Cloud Storage Channels (Google Drive & OneDrive)
* **Concept:** Serves as a proxy layer between controller and agent. All network traffic originates from and terminates at legitimate cloud provider endpoints (`*.googleapis.com`, `*.microsoft.com`).
* **Inbound Tasking:** The controller uploads tasking files or metadata via API tokens to dedicated directories. Agents periodically poll the endpoints over HTTPS to retrieve instructions.
* **Outbound Exfiltration:** Agents execute instructions locally, aggregate results into structured files, and upload them back to cloud storage for controller retrieval.
* **Evasion Impact:** Completely bypasses domain reputation checks and standard IP blocklists. Detection requires active TLS decryption and behavioral monitoring (e.g., unusual API call frequencies from non-standard binaries).

### 2. DNS Tunneling Channel (UDP Port 53)
* **Concept:** Uses standard host DNS resolution paths, which are rarely restricted outbound on internal networks.
* **Inbound Tasking (TXT Records):** Agents request TXT records for attacker-controlled domains (e.g., `task.c2domain.com`). The controller's DNS service responds with encoded execution commands inside the TXT payload.
* **Outbound Exfiltration (A Records):** Agent exfiltration data is chunked, Base64/Hex-encoded, and prepended as subdomains in rapid sequence A record queries (e.g., `<hex_data>.c2domain.com`). The controller intercepts incoming UDP 53 queries and reassembles the payload stream.

---

## 📂 Codebase & Component Analysis

### Server Infrastructure (`/server`)
* **`c2_core.h`**: Global system definitions, session tracking models (`C2Agent`), data buffer layouts (`Chunk`), and live network connection states.
* **`c2_modules.c`**: Multithreaded execution logic handling background listeners (`pthread`), raw DNS byte parsing, and `libcurl` integrations for OAuth2 cloud API interaction.
* **`main.c`**: Operator Command Line Interface (CLI) loop for tracking target heartbeats, managing active sessions, and queuing interactive tasks.

### Agent & Payload Engine (`/templates`)
* **`template_drive.ps1` / `template_onedrive.ps1`**: PowerShell templates implementing REST API interactions and polling queues.
* **`template_dns.ps1`**: Autonomous DNS query-response loop parsing incoming TXT payloads and executing local system commands.
* **`script_discovery.ps1`**: Baseline host reconnaissance module injected into template headers during dynamic payload generation.

---

## 🔍 Automated Host Vulnerability Pipeline

Upon successful agent initialization, system diagnostic data is routed back to the controller and parsed automatically:

```
[ Agent Host Profiling ] ──► [ Transport Stream ] ──► [ Reassembled: received_file.txt ]
                                                                   │
                                                                   v
                                                     [ WES-NG Python Engine ]
                                                                   │
                                                                   v
                                                    [ Missing Hotfixes / CVE List ]
```

1. **Telemetry Collection:** Host identification markers (Hostname, OS Version, Architecture, installed KB Hotfixes) are transmitted to the controller.
2. **Subprocess Execution:** The controller forks a Python process invoking WES-NG (Windows Exploit Suggester - Next Generation):
   ```bash
   python3 wes.py received_file.txt --muc-lookup
   ```
3. **Assessment Output:** The output is matched against the Microsoft Update Catalog to highlight unpatched vulnerabilities and potential Local Privilege Escalation (LPE) paths without performing noisy network vulnerability scans on the target.

---

## 🛡️ Defensive Analysis & Mitigation Strategies

To defend against the techniques demonstrated in this framework, security operations centers (SOC) should deploy:

| Attack Surface | Primary Detection Vector | Defensive Countermeasure |
| :--- | :--- | :--- |
| **Cloud API Abuses** | Process-to-network telemetry anomalies | Monitor un-signed process invocations (`powershell.exe`) interacting with REST API endpoints; implement SSL/TLS inspection. |
| **DNS Tunneling** | High volume of subdomains under single TLD | Deploy DNS entropy analytics, flag unusually large TXT responses, and restrict outbound Port 53 traffic exclusively to internal resolvers. |
| **Host Reconnaissance** | Process creation alerts for native discovery tools | Audit execution of system profiling commands (`systeminfo`, `wmic`) spawned by non-standard parent shells. |

---

## 🚀 Architectural Refactoring & Future Work

As an academic Proof of Concept (PoC), several areas were identified for future production-level refactoring:
- [ ] **Transport Layer Abstraction:** Decouple transport drivers using C function pointers to allow plug-and-play addition of new egress protocols (e.g., WebSockets, ICMP).
- [ ] **Concurrency & Thread Safety:** Transition global array session tracking to mutex-protected thread-safe data structures.
- [ ] **Dynamic Memory Management:** Replace fixed static allocation buffers (`BUF_SIZE`) with dynamic memory allocations guarded by explicit boundary checks.

---

## ⚠️ Disclaimer

*This software was developed solely for academic research, educational demonstrations, and authorized security evaluation as part of a final-year project at KIIT University. Misuse of this software to target systems without explicit prior authorization is illegal.*
