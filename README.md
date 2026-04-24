# Case Study: PacketMaze - Network Forensics & Incident Response

**Analyst:** Ditikrushna Routray
**Platform:** CyberDefenders (Lab #68 - PacketMaze)
**Discipline:** Network Forensics, Protocol Dissection, Incident Response

---

## ⚠️ Threat Context
**Scenario:** A persistent threat actor performed reconnaissance against the organization's external perimeter, discovering a legacy, public-facing FTP service that lacked basic access controls. By sniffing the unencrypted network traffic, the attacker successfully intercepted administrative credentials to gain initial access. To securely exfiltrate the staged data while evading corporate Data Loss Prevention (DLP) and IDS/IPS sensors, the adversary routed their communications through ProtonMail, leveraging its robust end-to-end encryption to blend malicious activity with legitimate web traffic.

---

## 🎯 Executive Summary
This case study details the triage and deep-packet inspection of a 37MB network capture (`.pcapng`). The investigation successfully identified a compromised user account via unencrypted protocol sniffing, tracked the attacker's post-exploitation file system staging, and extracted session-specific identifiers from encrypted TLS 1.3 traffic to map advanced adversary movements.

The entire analysis was conducted strictly via the Linux Command Line Interface (CLI), bypassing GUI tools in favor of targeted script-based dissection.

---

## 🛠️ Investigation Environment & Tooling
* **Primary Tool:** `tshark` (Terminal-based Wireshark)
* **Environment:** Linux OS (Bash)
* **Techniques:** Traffic filtering, SNI isolation, unencrypted payload extraction, hex manipulation.

> **OpSec Note:** To mitigate the risk of protocol dissector vulnerabilities inherent in handling unverified PCAP files, future forensic analysis will be migrated to a dedicated, network-isolated virtual sandbox.

---

## 🔍 Technical Findings & Indicators of Compromise (IOCs)

### 1. Cleartext Credential Interception (FTP)
Attackers frequently exploit legacy or misconfigured protocols to harvest credentials. Traffic analysis revealed an active FTP session where authentication data was transmitted without encryption.

* **Target Protocol:** FTP (File Transfer Protocol)
* **Compromised Account:** `kali`
* **Exposed Credential:** `AfricaCTF2021`
* **Extraction Methodology:** Isolated the FTP command sequence to extract the authentication argument.
    ```bash
    tshark -r evidence.pcapng -Y "ftp.request.command == PASS" -T fields -e ftp.request.arg
    ```
* **Methodology Rationale:** The `ftp.request.command == PASS` filter specifically isolates the plaintext password transmission inherent to legacy FTP, allowing immediate credential extraction without complex payload reconstruction.
* **Business Impact:** Plaintext credential exposure grants adversaries immediate, frictionless access to internal file repositories, leading to severe data breaches and providing a persistent foothold for lateral movement.

### 2. Attacker Persistence & Lateral Movement Staging
Following successful authentication, the attacker initiated file system modifications on the compromised server. Identifying these actions is critical for determining the scope of the breach.

* **Action Observed:** Creation of a non-standard, obfuscated staging directory.
* **Directory Name:** `ftp`
* **Creation Timestamp:** `2021-04-20 17:53:00`
* **Extraction Methodology:** Reconstructed the FTP-DATA stream to audit server-side file operations.
    ```bash
    tshark -r evidence.pcapng -Y "ftp-data" -V | grep -i "Apr 20"
    ```
* **Methodology Rationale:** Filtering for `ftp-data` and reconstructing the stream allows analysts to bypass the command channel and directly audit the attacker's server-side file modifications and directory creation actions.
* **Business Impact:** The establishment of an obfuscated staging directory signifies that the attacker is preparing bulk data for exfiltration or staging secondary malware, escalating the breach from unauthorized access to an active data theft operation.

### 3. Encrypted Traffic Triage (TLS 1.3)
To bypass network monitoring, attackers often route command-and-control (C2) or exfiltration traffic through encrypted channels. By analyzing the initial TLS handshake before the session key is established, specific session metadata can be extracted for tracking.

* **Target Domain:** `protonmail.com`
* **Indicator Type:** TLS 1.3 Client Random (32-byte hex string)
* **Extracted Value:** `24e92513b97a0348f733d16996929a79be21b0b1400cd7e2862a732ce7775b70`
* **Extraction Methodology:** Filtered traffic utilizing Server Name Indication (SNI) to isolate the specific target domain's Client Hello packet.
    ```bash
    tshark -r evidence.pcapng -Y "tls.handshake.extensions_server_name == \"protonmail.com\"" -T fields -e tls.handshake.random | head -n 1
    ```
* **Methodology Rationale:** Because TLS 1.3 encrypts the server certificate, Server Name Indication (SNI) is the primary plaintext identifier remaining in the initial Client Hello. This filter isolates the exact session to extract the Client Random, which is required for potential future traffic decryption.
* **Business Impact:** Utilizing encrypted communication channels like ProtonMail enables attackers to bypass perimeter monitoring and stealthily exfiltrate sensitive corporate data, drastically increasing the risk of intellectual property loss and regulatory penalties.

---

## 🧠 Analyst Notes & Retrospective
This analysis reinforces the necessity of strict CLI proficiency in high-pressure SOC environments. GUI tools often fail or crash under massive data loads, whereas `tshark` filters execute with high precision. Furthermore, the investigation highlighted the critical importance of secure file handling and environment isolation when dealing with potentially weaponized network artifacts.

---

## 🛡️ Remediation & Mitigation
To neutralize the identified vulnerabilities and harden the network against future compromises, the following strategic actions must be prioritized:
* **Deprecate Plaintext Protocols:** Immediately disable legacy FTP services across the environment. Mandate the use of secure, encrypted alternatives such as SFTP or FTPS for all internal and external file transfers.
* **Implement Strict Network Segmentation:** Isolate vulnerable or legacy systems into dedicated, tightly controlled VLANs. Prevent direct communication between external-facing servers and the core corporate network to contain potential breaches.
* **Enforce Multi-Factor Authentication (MFA):** Deploy MFA across all externally accessible services and administrative accounts to invalidate compromised credentials harvested from network sniffing.
* **Deploy Egress Filtering & TLS Inspection:** Restrict outbound server traffic to explicitly approved destinations. Implement TLS decryption at the perimeter firewall to inspect outgoing encrypted streams for unauthorized data exfiltration.