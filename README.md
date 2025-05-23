# Cloud-Driven SIEM with ELK: Real-Time Threat Detection and Automated Incident Response

## Introduction

This project is a hands-on cybersecurity lab that simulates real-world attack and defense scenarios. It integrates adversary emulation, SIEM pipeline development with the ELK stack, endpoint detection, real-time log analysis, automated response, firewall architecture, incident response workflows, and actionable security analytics.

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

### Basic Requirements

- **AWS Account:** An AWS account (with MFA enabled) and enough quota to launch EC2 instances for all lab components.
- **Networking & Firewalls:** Ability to configure AWS VPC, Security Groups, and open/close required ports for EC2, RDP/SSH, and cloud services.
- **Security & Automation:** Use AWS IAM roles and policies for access control, EC2 key pairs for instance login, Security Groups for isolation, and automate deployments with scripts or AWS CLI for consistency and speed.

---

### ☁️ Lab Infrastructure Overview

| Role                      | OS/Distribution         | Instance Type / Specs               | Description                                       |
|---------------------------|------------------------|-------------------------------------|---------------------------------------------------|
| **ELK Server**            | Ubuntu 22.04           | t2.xlarge (2 vCPUs, 8 GB RAM)       | Central SIEM for log collection, analysis, and dashboards |
| **Fleet Server**          | Ubuntu 22.04           | t2.medium (2 vCPUs, 4 GB RAM)       | Manages Elastic Agents for endpoint visibility    |
| **Windows Target**        | Windows Server 2022    | t2.medium (2 vCPUs, 4 GB RAM)       | RDP-exposed endpoint for red team testing; hosts Kali VM for internal attack simulation |
| **Kali (VM)**             | Kali Linux Rolling (VM)| 1 vCPU, 2 GB RAM (within Windows)   | Simulated attacker VM inside Windows target       |
| **Ubuntu Target**         | Ubuntu 22.04           | t2.medium (2 vCPUs, 4 GB RAM)       | SSH-exposed endpoint to simulate brute-force      |
| **Windows + osTicket**    | Windows Server 2022    | t2.medium (2 vCPUs, 4 GB RAM)       | Incident response management using osTicket       |
| **Mythic C2 Server**      | Ubuntu 22.04           | t2.medium (2 vCPUs, 4 GB RAM)       | Hosts the Mythic Command & Control (C2) framework |

---

## Architecture

### Firewall Architecture

> **Note:** The firewall in this lab is not only used to block or prevent unwanted access, but also to ensure that only the required and intended network connections are allowed for the proper functioning of each service and server. By explicitly defining which ports and protocols are permitted between components (such as endpoints, Fleet Server, ELK Server, osTicket, and administrative systems), the firewall enforces both security and connectivity. This approach allows necessary communication for log forwarding, agent orchestration, remote administration, and ticketing integration, while minimizing unnecessary exposure and reducing the attack surface.

#### Windows Endpoint

| Source       | Port(s)/Protocol | Action | Reason                                               |
| ------------ | ---------------- | ------ | ---------------------------------------------------- |
| 0.0.0.0/0    | TCP 3389         | Allow  | Simulate exposed RDP service for remote access       |
| Fleet Server | TCP 8220, 8200   | Allow  | Agent-to-Fleet communication                         |
| ELK Server   | TCP 5044, 9200   | Allow  | Send logs to Elasticsearch                           |
| Your IP      | TCP 1–65535      | Allow  | Remote admin config (from your machine via RDP only) |

#### Ubuntu Endpoint

| Source       | Port(s)/Protocol | Action | Reason                                               |
| ------------ | ---------------- | ------ | ---------------------------------------------------- |
| 0.0.0.0/0    | TCP 22           | Allow  | Simulate exposed SSH service for brute-force testing |
| Fleet Server | TCP 8220, 8200   | Allow  | Agent-to-Fleet communication                         |
| ELK Server   | TCP 5044, 9200   | Allow  | Send logs to Elasticsearch                           |
| Your IP      | TCP 1–65535      | Allow  | Remote admin/config                                  |

#### ELK Server

| Source            | Port(s)/Protocol | Action | Reason                                              |
| ----------------- | ---------------- | ------ | --------------------------------------------------- |
| Windows           | TCP 5044, 9200   | Allow  | Receive Sysmon and Defender logs via Beats or Agent |
| Ubuntu            | TCP 5044, 9200   | Allow  | Receive `/var/log` data from Filebeat/Elastic Agent |
| Fleet Server      | TCP 9200         | Allow  | Fleet querying and ingest coordination              |
| OS Ticket         | TCP 1–65535      | Allow  | Ticket system access for alert/case management      |
| Your IP           | TCP 1–65535      | Allow  | Admin/dashboard access for analytics and monitoring |
| OS Ticket Server  | TCP 9200, 5601   | Allow  | Allow bi-directional integration between ELK stack and OS Ticket server |

---

### Setup Layout

![Setup Layout](https://github.com/fakowajo123/Cloud-Driven-SIEM-with-ELK-Real-Time-Threat-Detection-and-Automated-Incident-Response/raw/main/Setup/Layout%20diagram%20.jpg)

### Attack Diagram Layout

![Attack Layout](https://github.com/fakowajo123/Cloud-Driven-SIEM-with-ELK-Real-Time-Threat-Detection-and-Automated-Incident-Response/raw/main/Setup/Attack%20Layout%20.jpg)

---

## ELK Stack Setup & Installation

### 1. Install Java

The ELK stack (Elasticsearch and Kibana) requires Java. Install OpenJDK 11:

```bash
sudo apt update
sudo apt install openjdk-11-jdk -y
```

### 2. Download and Install Elasticsearch

Official download page: https://www.elastic.co/downloads/elasticsearch  
Direct download (replace version if needed):

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.14.0-amd64.deb
sudo dpkg -i elasticsearch-8.14.0-amd64.deb
```

Or use apt repository:

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https -y
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update
sudo apt install elasticsearch -y
```

- Start and enable Elasticsearch:

```bash
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```

### 3. Download and Install Kibana

Official download page: https://www.elastic.co/downloads/kibana  
Direct download (replace version if needed):

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.14.0-amd64.deb
sudo dpkg -i kibana-8.14.0-amd64.deb
```

Or use apt repository (if not already added above):

```bash
sudo apt install kibana -y
```

- Start and enable Kibana:

```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

### 4. Access Kibana

- Open your browser and go to:  
  `http://<your-elk-server-public-ip>:5601`
- Follow the setup instructions to complete the configuration and set passwords for built-in users.

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
  Every rule, dashboard, alert, and detection logic should include clear, concise notes and instructions within the configuration or as accompanying documentation. Contributors must explain what and why, not just how.
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
