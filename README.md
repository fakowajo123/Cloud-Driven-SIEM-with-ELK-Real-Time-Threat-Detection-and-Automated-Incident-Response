# Cloud-Driven SIEM with ELK: Real-Time Threat Detection and Automated Incident Response

## Introduction

This project is a hands-on cybersecurity lab that simulates real-world attack and defense scenarios. It integrates adversary emulation, SIEM pipeline development with the ELK stack, endpoint detection, real-time log analysis, automated response, firewall architecture, and incident response workflows. The purpose is to demonstrate practical skills in building, monitoring, and defending modern network environments using both offensive and defensive security tools and techniques.

**Key skills demonstrated:**

- SIEM pipeline implementation with the ELK stack
- Adversary emulation (Mythic, SSH/RDP brute-force)
- Endpoint detection (Sysmon, Windows Defender, Ubuntu logs)
- Real-time log analysis and automated response
- Custom dashboard development
- Granular firewall rule enforcement
- Incident response workflow integration with osTicket
- Network architecture, offensive/defensive security, and centralized monitoring
- Actionable security analytics

---

## Table of Contents

1. [Setup & Prerequisites](#setup--prerequisites)
2. [Architecture](#architecture)
   - [Firewall Architecture](#firewall-architecture)
   - [Setup Layout](#setup-layout)
   - [Attack Diagram Layout](#attack-diagram-layout)
3. [ELK Stack Setup & Installation](#elk-stack-setup--installation)
   - [Kibana Installation](#kibana-installation)
4. [Agent & Fleet Server Configuration](#agent--fleet-server-configuration)
5. [Sysmon Configuration & Setup](#sysmon-configuration--setup)
6. [Attack Simulation](#attack-simulation)
   - [SSH Bruteforce Attack](#ssh-bruteforce-attack)
   - [RDP Attack](#rdp-attack)
   - [Data Exfiltration Attack](#data-exfiltration-attack)
7. [Dashboards, Log Analysis & Alerts](#dashboards-log-analysis--alerts)
8. [OS Ticket Integration](#os-ticket-integration)
   - [OS Ticket Installation](#os-ticket-installation)
   - [Integration with SIEM](#integration-with-siem)
9. [Detection](#detection)
10. [Response: Automated Defense](#response-automated-defense)
11. [Contributing](#contributing)
12. [License](#license)

---

## Setup & Prerequisites

- List all hardware, software, and cloud requirements.
- Network setup details.
- User permissions and security best practices.

---

## Architecture

### Firewall Architecture

> _Describe and diagram your firewall rules and segmentation._

### Setup Layout

> _Provide an overview of the entire setup environment._

### Attack Diagram Layout

> _Visualize how attacks traverse your architecture._

---

## ELK Stack Setup & Installation

- Step-by-step installation for Elasticsearch and Logstash.

### Kibana Installation

- How to install and configure Kibana UI for visualization.

---

## Agent & Fleet Server Configuration

- How to deploy agents and set up the Elastic Fleet Server.
- Enrollment and communication setup.

---

## Sysmon Configuration & Setup

- Installing and configuring Sysmon for endpoint telemetry.
- Log forwarding to ELK.

---

## Attack Simulation

### SSH Bruteforce Attack

- Simulate and monitor brute-force attempts on SSH.

### RDP Attack

- Simulate unauthorized RDP access attempts.

### Data Exfiltration Attack

This scenario demonstrates a simulated data exfiltration attack using the Mythic Command & Control (C2) framework. The objective is to emulate an attacker leveraging stolen credentials to access and exfiltrate sensitive data from a target system.

**Steps performed:**

1. **Credential Theft Simulation**
   - Used Mythic C2 to deploy an agent on the target endpoint.
   - Executed credential dumping techniques (e.g., via Mimikatz or built-in Mythic payload modules) to extract credentials such as username and password hashes from the target machine.
   - Captured and stored the stolen credentials within the C2 operator console for further use.
   - _Sample Output:_
     ```plaintext
     [*] Retrieved credentials:
     Username: victimuser
     Password Hash: aad3b435b51404eeaad3b435b51404ee:5f4dcc3b5aa765d61d8327deb882cf99
     ```
2. **Establishing C2 Communication**
   - Maintained persistence on the victim system using the Mythic agent.
   - Ensured secure and covert communication back to the Mythic server for command execution and data transfer.

3. **Data Discovery & Exfiltration**
   - Enumerated directories containing sensitive files (e.g., documents, password files).
   - Used Mythic’s file browser and download modules to exfiltrate selected files back to the C2 server.
   - _Example command:_
     ```plaintext
     download C:\Users\victimuser\Documents\confidential.xlsx
     ```
   - Verified that the exfiltrated data appeared in the Mythic C2 file repository.

4. **Detection & Logging**
   - Monitored SIEM logs for unusual credential access and file transfer events.
   - Created detection rules and dashboards to visualize and alert on suspicious data exfiltration activity.

**Key Takeaways:**
- Demonstrated end-to-end workflow of credential theft and data exfiltration using a modern C2 platform.
- Showed how SIEM solutions can detect and alert on these activities in real time.
- Highlighted the importance of monitoring lateral movement and unauthorized data access within enterprise environments.

---

## Dashboards, Log Analysis & Alerts

- Building dashboards in Kibana.
- Querying and analyzing logs.
- Setting up real-time alerts.

---

## OS Ticket Integration

### OS Ticket Installation

- Guide to installing and configuring OS Ticket.

### Integration with SIEM

- Connecting alerts/incidents to OS Ticket for ticketing and response workflow.

---

## Detection

- How threats are detected using SIEM rules, correlation, and analytics.

---

## Response: Automated Defense

- Automated playbooks and scripts for incident response.
- Example: blocking IPs, disabling compromised accounts, sending notifications.

---

## Contributing

We welcome and encourage contributions from everyone, regardless of experience level!  
Our aim is to make this lab approachable and educational, even if you are new to cybersecurity or SIEM solutions.

**Contribution Guidelines:**

- **Notes & Instructions:**  
  Every rule, dashboard, alert, and detection logic should include clear, concise notes and instructions within the configuration or as accompanying documentation. Contributors must explain what each rule does, why it is important, and how it can be tested. This helps beginners understand the purpose and usage of each component.

- **Step-by-step Help:**  
  If your contribution involves a new detection rule, alert, or integration, provide a step-by-step usage and testing guide. Screenshots and examples are highly encouraged.

- **AI Integration & Innovation:**  
  We encourage use of AI-driven techniques for detection, alerting, or automated response. You may propose or contribute machine learning models, anomaly detection scripts, or AI-based analytics to enhance threat detection and response. Please document your AI approach and provide usage instructions.

- **Inclusivity:**  
  Please write your notes and instructions so that even those with no prior experience can follow along and contribute.

- **Pull Requests:**  
  - Fork the repository and create your branch from `main`.
  - Add your feature or fix, including documentation and instructions.
  - Submit a pull request with a clear description and testing steps.

- **Questions & Support:**  
  If you’re unsure about anything, open an issue or discussion, and our community will help you out!

---

## License

> _Your chosen license._
