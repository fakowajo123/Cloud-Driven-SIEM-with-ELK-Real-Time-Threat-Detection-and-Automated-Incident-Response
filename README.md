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

### 1. Update and Upgrade the System

Before installing ELK components, update your server to ensure all packages are current:

```bash
sudo apt update && sudo apt upgrade -y
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

- **Important:**  
  **After installing Elasticsearch, a password for the `elastic` user will be automatically generated and shown in your terminal.  
  **Copy and save this password**—you will need it to log in to Kibana and complete initial setup.**

- **Configure Elasticsearch to listen on all interfaces:**  
  Edit the config file and change the network host from `localhost` to `0.0.0.0`:
  ```bash
  sudo nano /etc/elasticsearch/elasticsearch.yml
  ```
  Find the line:
  ```
  #network.host: 192.168.0.1
  ```
  And set:
  ```
  network.host: 0.0.0.0
  ```
  Then restart Elasticsearch:
  ```bash
  sudo systemctl restart elasticsearch
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

- **Configure Kibana to listen on all interfaces:**  
  Edit the config file and change the server host from `localhost` to `0.0.0.0`:
  ```bash
  sudo nano /etc/kibana/kibana.yml
  ```
  Find or add:
  ```
  server.host: "0.0.0.0"
  ```
  Then restart Kibana:
  ```bash
  sudo systemctl restart kibana
  ```

### 4. Access Kibana

- Open your browser and go to:  
  `http://<your-elk-server-public-ip>:5601`
- Follow the setup instructions to complete the configuration and set passwords for built-in users.
- **When prompted to log in, use the `elastic` username and the password you copied after installing Elasticsearch.**

### 5. Key Enrollment for Kibana and Fleet/Agent

- **To generate an enrollment key for Kibana:**
  - SSH into your ELK server and go to the Elasticsearch directory:
    ```bash
    cd /usr/share/elasticsearch/bin
    ```
  - Run the following command to generate an enrollment token (key) for Kibana:
    ```bash
    ./elasticsearch-create-enrollment-token -s kibana
    ```
  - **Copy the generated enrollment key.**
  - When you start Kibana for the first time, it will prompt you to enter this enrollment key.  
    Paste the copied key into Kibana to complete the setup and establish the connection between Kibana and Elasticsearch.

---

### 6. Generate and Configure Kibana API Keys

- To display and manage Kibana API/encryption keys, go to the Kibana installation directory:
  ```bash
  cd /usr/share/kibana/bin
  ls
  ```
- Run the following command to display required encryption keys:
  ```bash
  ./kibana-enrollment-keys
  ```
  This will output keys like:
  ```
  xpack.encryptedSavedObjects.encryptionKey: 0813d62e0a7a1fb60fb90754e0a5c97b
  xpack.reporting.encryptionKey: 0b96ab68dbafd4238c8e1f99445d8351
  xpack.security.encryptionKey: 6cc41d650536f45a4b21ae49e6f375ea
  ```
- Copy each of these keys.

- Still in the same directory, add each key to the Kibana keystore:
  ```bash
  ./kibana-keystore add xpack.encryptedSavedObjects.encryptionKey
  ./kibana-keystore add xpack.reporting.encryptionKey
  ./kibana-keystore add xpack.security.encryptionKey
  ```
  For each command, you will be prompted to paste the corresponding key you copied above.

- After entering all three keys, restart the Kibana service:
  ```bash
  sudo systemctl restart kibana
  ```

---

## Agent & Fleet Server Configuration

### 1. Add a Fleet Server in Kibana

- Log in to **Kibana** using your browser.
- Navigate to **Management > Fleet > Fleet Servers**.
- Click **Add a Fleet Server**.
- You will be prompted to enter the **IP address** of your Fleet Server (the machine you intend to run the Fleet Server on).
- Configure the settings as needed and continue.

### 2. Copy the Enrollment Command

- After entering your Fleet Server’s IP, Kibana will generate a command that includes:
  - The enrollment token
  - The Fleet Server host (your Fleet Server’s IP address)
  - The correct Kibana URL
- **Copy the entire command** shown in Kibana.

### 3. Enroll the Fleet Server on Your Machine

- SSH into the machine you want to use as your Fleet Server.
- **Paste and run the copied command** in your terminal.
- This will install and enroll Elastic Agent, registering it as a Fleet Server in your ELK environment.

### 4. Verify Fleet Server Status

- Return to Kibana’s **Management > Fleet > Fleet Servers** page.
- Ensure your Fleet Server appears as healthy and online.

### 5. Enroll Endpoints (Windows and Ubuntu Targets)

- After your Fleet Server is online, you can enroll other endpoint agents (Windows, Ubuntu, etc.) by repeating the process:
  - In **Fleet > Agents**, click on the **Fleet Server** you set up (this server will be used for both Windows and Ubuntu targets).
  - Copy the generated enrollment command/link provided by Kibana.
  - On each target machine (Windows or Ubuntu), open the command line (CMD, PowerShell, or terminal) and **paste the enrollment command**.
  - Run the command to enroll the agent. The agent will connect to the Fleet Server, and logs from your target machines will begin streaming into your ELK Stack.

---

## Sysmon Configuration & Setup

### 1. Download Sysmon

- Download Sysmon from the official Microsoft Sysinternals page:  
  [https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- Save the Sysmon zip file and extract its contents to a preferred folder (e.g., `C:\Sysmon`).

### 2. Download a Sysmon Configuration File

- Get a robust Sysmon configuration from the [sysmon-modular project on GitHub](https://github.com/olafhartong/sysmon-modular).
- Download the latest `sysmonconfig.xml` from the repository:
  - Go to: [https://github.com/olafhartong/sysmon-modular](https://github.com/olafhartong/sysmon-modular)
  - Download and save `sysmonconfig.xml` into the same folder where you extracted Sysmon (e.g., `C:\Sysmon`).

### 3. Install & Configure Sysmon

- Open **Command Prompt as Administrator**.
- Change directory to your Sysmon folder:
  ```cmd
  cd C:\Sysmon
  ```
- Install and configure Sysmon using the configuration file:
  ```cmd
  .\Sysmon64.exe -i .\sysmonconfig.xml
  ```
- You should see a message confirming successful installation.

### 4. Verify Sysmon Operation

- Open **Services** (search for "services.msc") and check that the **Sysmon64** service is running.
- Open **Event Viewer** (`eventvwr.msc`):
  - Navigate to **Applications and Services Logs > Microsoft > Windows > Sysmon > Operational**.
  - Confirm that Sysmon events are being logged.

---

### 5. Ingesting Sysmon and Defender Logs into ELK

To have your Windows logs (from Sysmon and Microsoft Defender) ingested into ELK, follow these steps:

1. **Get the Operational Channel Names:**
   - Open **Event Viewer**.
   - For **Sysmon**:  
     Go to `Applications and Services Logs > Microsoft > Windows > Sysmon > Operational`, then right-click **Operational** and select **Properties**.  
     Copy the **Full Name** shown (e.g., `Microsoft-Windows-Sysmon/Operational`).
   - For **Windows Defender**:  
     Similarly, find the Defender log (e.g., `Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational`), right-click **Operational**, select **Properties**, and copy the **Full Name** (e.g., `Microsoft-Windows-Windows Defender/Operational`).

2. **Add Custom Windows Event Logs in Elastic:**
   - In Kibana, go to **Integrations**.
   - Search for **Custom Windows Event Log**.
   - Click **Add Custom Windows Event Log**.
   - Create a name for the policy (e.g., `Win-Detection-Logs`).
   - In the integration settings, paste the **channel name** you copied earlier (e.g., `Microsoft-Windows-Sysmon/Operational`) into the required field.
   - You can add multiple custom event logs by adding more channel names (for both Sysmon and Windows Defender).
   - Assign this integration to your existing **Windows agent** (the Elastic Agent running on your Windows endpoint).
   - Save and apply the changes.

3. **Verify Log Ingestion:**
   - Go to **Kibana > Discover**.
   - Check for incoming Sysmon and Defender logs from your Windows machine.

---

## Attack Simulation

### SSH Bruteforce Attack

- **Tool:** We will use `hydra` to simulate a brute-force attack against an SSH service.
- **Platform:** Use your **Kali Linux VM** for running these attack simulations.

#### 1. Install Hydra

On Kali/Ubuntu:
```bash
sudo apt update && sudo apt install hydra -y
```

#### 2. Example SSH Brute Force Command

Replace `<target-ip>`, `<userlist.txt>`, and `<passlist.txt>` with your actual target IP, user list, and password list file paths.

```bash
hydra -L <userlist.txt> -P <passlist.txt> ssh://<target-ip>
```

**Example:**
```bash
hydra -L users.txt -P passwords.txt ssh://192.168.1.10
```

- `-L users.txt` : File containing username list
- `-P passwords.txt` : File containing password list
- `ssh://192.168.1.10` : Target SSH service

#### 3. Monitor in ELK

- During the brute-force attempt, monitor your ELK Stack dashboards to observe and analyze the attack events as they are ingested from your endpoints.

---

### RDP Bruteforce Attack

- **Tools:** `hydra` and `xrdp` (for RDP service).
- **Platform:** Use your **Kali Linux VM** for running these attack simulations.

#### 1. Install xrdp (if targeting a Linux machine with RDP) and Hydra

On Ubuntu/Kali:
```bash
sudo apt update && sudo apt install hydra xrdp -y
```

#### 2. Example RDP Brute Force Command

Replace `<target-ip>`, `<userlist.txt>`, and `<passlist.txt>` with your actual target IP, user list, and password list file paths.

```bash
hydra -L <userlist.txt> -P <passlist.txt> rdp://<target-ip>
```

**Example:**
```bash
hydra -L users.txt -P passwords.txt rdp://192.168.1.20
```

- `-L users.txt` : File containing username list
- `-P passwords.txt` : File containing password list
- `rdp://192.168.1.20` : Target RDP service

#### 3. Monitor in ELK

- During the brute-force attempt, monitor your ELK Stack dashboards to observe and analyze the attack events as they are ingested from your endpoints.

---

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
