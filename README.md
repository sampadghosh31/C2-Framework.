
🛡️ The Mission
# C2-Framework.
A modular C2 Framework developed during my final year cyber project at KIIT, featuring DNS/GitHub communication channels, custom agent modules, and a dual GUI/CLI interface
-------------------------------------------------------------------------------------------
This framework describes a modular Command and Control (C2) architecture designed to demonstrate how post-exploitation communication, cover networks, and automated host auditing operate. Below is a detailed breakdown of the system architecture, its communication channels, payload deployment mechanics, and defensive analysis considerations.

1. Multi-Channel C2 Mechanics
A central challenge in network defense and red-teaming design is maintaining persistent communication with an agent when primary egress routes are monitored or restricted. This system employs three distinct egress strategies:

                            +--------------------------+
                            |    C2 Controller (C)     |
                            +------------+-------------+
                                         |
         +-------------------------------+-------------------------------+
         |                               |                               |
         v                               v                               v
[Google Drive API]               [OneDrive API]                [DNS Tunnel (UDP 53)]
 HTTPS (Port 443)                HTTPS (Port 443)               TXT / A Query-Response
         |                               |                               |
         +-------------------------------+-------------------------------+
                                         |
                                         v
                            +--------------------------+
                            |   Victim Workstation     |
                            |     (PowerShell)         |
                            +--------------------------+


Channel A & B: Cloud Storage APIs (Google Drive & OneDrive)
Mechanism: Rather than communicating directly with a custom server IP/domain, the agent uses legitimate cloud provider APIs as a proxy layer (often termed data pipe leverage or covert storage channels).

Operation:

Commands: The C2 controller uploads tasking files (or metadata) to a dedicated directory in Google Drive or OneDrive using API tokens. The PowerShell agent periodically polls the API over standard HTTPS (Port 443) to read pending tasks.

Data Exfiltration: The agent writes execution output into local files and uploads them back to the cloud storage platform, where the controller retrieves them.

Security & Evasion Implications:

Since traffic is directed to legitimate domains (*.googleapis.com or *.microsoft.com), standard domain reputation filters and IP blocklists generally do not inspect or block the connections.

Traffic inspection relies heavily on SSL/TLS decryption (TLS Inspection) and behavior-based monitoring (e.g., detecting unusual API invocation patterns or high-frequency uploads by non-standard binaries/scripts).

Channel C: DNS Tunneling (UDP Port 53)
Mechanism: DNS is a foundational network protocol that is rarely blocked outbound, as host systems require it to resolve domain names.

Operation:

Inbound Tasks (TXT Records): The agent sends a DNS query for a TXT record under an attacker-controlled authoritative domain (e.g., task.c2domain.com). The controller’s custom DNS server responds with encoded data payload strings inside the TXT record response.

Outbound Data (A/AAAA Records): The agent encodes system diagnostic data or command outputs (e.g., in Base64 or Hex) and prepends it as subdomains in rapid A record queries (e.g., 48656c6c6f.c2domain.com). The controller intercepts these incoming UDP 53 packets and reassembles the payload chunks into coherent data streams.

2. Server & Agent Component Analysis
   
Server Infrastructure (C2 Controller)
c2_core.h: Serves as the global contract. It defines data structures like Chunk (to handle fragmented incoming data buffers) and C2Agent (to track active sessions, target identifiers, and last-seen timestamps/heartbeats).

c2_modules.c: Houses the heavy processing logic:

Multithreaded socket handlers (pthread) for concurrent UDP/DNS packet listening and HTTP queue management.

Custom byte parsers to extract subdomains from raw DNS query bytes.

Integration with libcurl to handle cloud API authentication (OAuth2 token refresh headers, REST requests).

main.c: Provides the operator interface (CLI) to select active targets, issue shell commands, monitor heartbeats, and switch between operational modes.

Target Payload Generation
Modular Templating: To minimize payload size and avoid hardcoding environment variables, the C2 server uses template files (template_drive.ps1, template_onedrive.ps1, template_dns.ps1).

Header Injection: During payload dynamic generation, the server prepends script_discovery.ps1 (a system profiling script that captures hostnames, OS version, architecture, and installed hotfixes) to the selected transport script to produce the final executable script (generated_agent*.ps1).

3. Automated Post-Exploitation Pipeline 
   
Once an agent connects and executes the initial profiling logic, the server automates local vulnerability assessment:

[Agent Profiling] ──> [Data Chunking] ──> [Reassembled to received_file.txt]
                                                     │
                                                     v
                                      [WES-NG Python Parser]
                                                     │
                                                     v
                                     [Missing Hotfixes / CVE List]
                                     
Data Reassembly: Reconstructed profiling information from incoming DNS or HTTPS streams is written to a localized file (received_file.txt).

Process Invocation: The C2 controller spawns a Python subprocess executing WES-NG (wes.py), a tool that parses system systeminfo output against a local database of known Windows vulnerabilities.

-muc-lookup Parameter: Queries the Microsoft Update Catalog to cross-reference missing patches against known CVEs and privilege escalation vectors, giving the operator immediate insight into potential local privilege escalation (LPE) paths without relying on active network scans on the victim network.

4. Software Architecture & Refactoring Considerations
   
The document notes that the baseline server implementation is primarily procedural and monolithic due to the fast-paced nature of proof-of-concept development. Key areas typically addressed in code refactoring for production-grade or robust academic frameworks include:

Abstraction of Transport Layer: Implementing an abstract channel interface (or function pointers in C) so that adding new transport channels (e.g., Slack API, WebSockets, ICMP) requires zero modifications to the core CLI or execution loop.

State Management & Concurrency Safety: Replacing global array-based session tracking with thread-safe data structures (e.g., mutex-protected hash maps or linked lists) to prevent race conditions during heavy agent traffic.

Input Sanitization & Buffer Protection: Transitioning from fixed array allocations (BUF_SIZE) and direct file-stream operations to dynamic memory management with strict boundary checks to prevent memory corruption vulnerabilities within the listener binaries.
                            
