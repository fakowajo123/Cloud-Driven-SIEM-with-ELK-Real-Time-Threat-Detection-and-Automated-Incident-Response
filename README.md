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

### 🔎 Log Analysis & Threat Hunting: RDP and SSH Events

This section describes how to analyze logs and create threat hunting queries for RDP (Windows endpoint) and SSH (Ubuntu endpoint) attack scenarios. These queries help you visualize, detect, and alert on suspicious authentication activity, such as brute-force attempts, using Kibana's powerful search and visualization features.

#### Example ELK/Kibana Queries for Threat Hunting

You can use the following queries in Kibana's Discover or Alerts sections:

---

**SSH Brute-Force Detection (Failed Logins)**

```kql
event.dataset : "ssh" and event.action : "failed_login"
```
- Shows all failed SSH login attempts.

**SSH Multiple Failed Attempts from Single IP**

```kql
event.dataset : "ssh" and event.action : "failed_login"
| stats count() by source.ip, user.name
| where count() > 5
```
- Reveals possible brute-force sources by aggregating failed logins.

**[View SSH Dashboard (Ubuntu Endpoint)](https://github.com/fakowajo123/Cloud-Driven-SIEM-with-ELK-Real-Time-Threat-Detection-and-Automated-Incident-Response/blob/main/Dashboard/Ssh%20Brute%20force%20dashboard.jpg)**

---

**RDP Brute-Force Detection (Failed Logins on Windows)**

```kql
event.provider : "Microsoft-Windows-Security-Auditing" and event.code : "4625" and winlog.logon.type : "10"
```
- Shows failed RDP (Remote Desktop) logon attempts.

**RDP Multiple Failed Logons from Single IP**

```kql
event.provider : "Microsoft-Windows-Security-Auditing" and event.code : "4625" and winlog.logon.type : "10"
| stats count() by source.ip, user.name
| where count() > 5
```
- Aggregates failed RDP logons by source IP and username.

**[View RDP Dashboard (Windows Endpoint)](https://github.com/fakowajo123/Cloud-Driven-SIEM-with-ELK-Real-Time-Threat-Detection-and-Automated-Incident-Response/blob/main/Dashboard/RDP%20brute%20force%20dashboard.jpg)**

---

**Successful SSH Logins**

```kql
event.dataset : "ssh" and event.action : "success"
```

**Successful RDP Logins**

```kql
event.provider : "Microsoft-Windows-Security-Auditing" and event.code : "4624" and winlog.logon.type : "10"
```

---

### 🛎️ Creating Real-Time Alerts (Kibana)

You can create real-time alerts (rules) in ELK Stack that match the custom queries above, so you are notified immediately when suspicious activity is detected. Below are step-by-step instructions:

#### How to Create Custom Detection Rules in Kibana

1. **Go to Stack Management**  
   - In Kibana, navigate to **Stack Management > Rules and Connectors**.

2. **Create a New Rule**
   - Click **Create rule**.

3. **Select Rule Type**
   - Choose **Elasticsearch query** or **Threshold** rule type, depending on your use-case.

4. **Configure Rule Details**
   - **Name:** Give your rule a descriptive name (e.g., "SSH Brute Force Detection").
   - **Index pattern:** Select the index pattern that matches your logs (e.g., `logs-*` or `filebeat-*`).

5. **Add the KQL Query**  
   - For SSH brute force, use:
     ```kql
     event.dataset : "ssh" and event.action : "failed_login"
     ```
   - For multiple SSH failed attempts from a single IP (Threshold rule example):
     - **Field:** `source.ip`
     - **Threshold:** more than 5 occurrences in 1 minute
     - **Filter:** `event.dataset : "ssh" and event.action : "failed_login"`
   - For RDP brute force, use:
     ```kql
     event.provider : "Microsoft-Windows-Security-Auditing" and event.code : "4625" and winlog.logon.type : "10"
     ```
   - For multiple RDP failed logons from a single IP (Threshold rule example):
     - **Field:** `source.ip`
     - **Threshold:** more than 5 occurrences in 1 minute
     - **Filter:** `event.provider : "Microsoft-Windows-Security-Auditing" and event.code : "4625" and winlog.logon.type : "10"`

6. **Set Schedule and Actions**
   - Set the rule to check every minute (or your required interval).
   - Configure action(s): email, Slack notification, webhook, or integration with your ticketing system (e.g., osTicket).

7. **Save and Enable the Rule**
   - Save your rule and ensure it is enabled.

**Tip:** Use your dashboard images as references for visualizations and to validate your queries.

---

#### Example: Visual Alert Evidence

Below are example images showing how alerts look once triggered for different attack scenarios:

**SSH and RDP Brute-Force Alert Example:**  
![SSH and RDP Alert](https://github.com/fakowajo123/Cloud-Driven-SIEM-with-ELK-Real-Time-Threat-Detection-and-Automated-Incident-Response/raw/main/Alerts/RDP%20and%20ssh%20ALERT%20Triggered.jpg)

**C2 Command and Control Malware Alert Example:**  
![C2 Command and Control Alert](https://github.com/fakowajo123/Cloud-Driven-SIEM-with-ELK-Real-Time-Threat-Detection-and-Automated-Incident-Response/raw/main/Alerts/C2%20Command%20Alert%20Triggered.jpg)

---

## OS Ticket Integration

### OS Ticket Installation

#### Deploying osTicket using XAMPP on Windows

You can host osTicket locally using XAMPP, which provides Apache, MySQL (MariaDB), and PHP in a single package. Below are the setup and configuration steps:

1. **Download and Install XAMPP**
   - Download XAMPP from the [official website](https://www.apachefriends.org/index.html).
   - Run the installer and follow the prompts to install XAMPP (default options are fine).
   - Launch XAMPP Control Panel and **start** both the Apache and MySQL services.

2. **Download osTicket**
   - Download the latest osTicket release from [osTicket Download Page](https://osticket.com/download/).
   - Extract the osTicket archive.
   - Copy the extracted **upload** folder and rename it (e.g., `osticket`).
   - Move this folder to your XAMPP `htdocs` directory (usually: `C:\xampp\htdocs\osticket`).

3. **Create a MySQL Database for osTicket**
   - Open your browser and go to `http://localhost/phpmyadmin`.
   - Click **Databases** tab.
   - Enter a database name (e.g., `osticket`) and click **Create**.
   - Click **User Accounts** > **Add user account**.
     - Username: `osticketuser` (example)
     - Host: `localhost`
     - Password: choose a strong password
     - Check **Create database with same name and grant all privileges** (optional)
     - Or, grant all privileges on the `osticket` database
   - Save the user and note the credentials.

4. **Configure osTicket**
   - In `C:\xampp\htdocs\osticket\include`, rename `ost-sampleconfig.php` to `ost-config.php`.
   - Ensure `ost-config.php` is writable (`Right-click > Properties > Uncheck Read-only`).
   - **Important:** After completing the osTicket installer (step 5), download the generated `ost-config.php` file if prompted by the installer, and place it inside the `include` directory of your osTicket installation (`C:\xampp\htdocs\osticket\include\ost-config.php`), replacing the existing one if necessary.

5. **Run osTicket Web Installer**
   - Open your browser and go to: `http://localhost/osticket/`
   - Follow the installation steps:
     - Enter your helpdesk name and admin details.
     - For database settings:
       - MySQL Database: `osticket`
       - MySQL Username: `osticketuser`
       - MySQL Password: (your password)
       - MySQL Hostname: `localhost`
   - Complete the installer.
   - **When prompted by the installer, download the new `ost-config.php` and place it in the `include` directory as described above.**

6. **Switch to Using Server IP Address for Network Access**
   - After initial setup, log in to the osTicket admin panel at `http://localhost/osticket/scp/`.
   - To allow users to access osTicket from other devices on your network:
     - Edit the osTicket system settings and update any reference to "localhost" to your server's IP address (e.g., `192.168.1.100`).
     - You may also need to edit the `ost-config.php` file in `C:\xampp\htdocs\osticket\include` to update the `HTTP_SERVER` or site URL variable if present.
     - Restart Apache in XAMPP after making these changes.
   - Users can now access osTicket using `http://<your-server-ip>/osticket/` and the admin portal at `http://<your-server-ip>/osticket/scp/`.

---
### Integration with SIEM

You can connect your SIEM (ELK Stack) to osTicket to automatically create tickets from alerts. This is commonly done using email piping, webhooks, or osTicket's API.  
For advanced automation, you can set up a custom integration script that acts as a bridge between ELK and osTicket.

#### Step-by-Step: osTicket & SIEM Integration

1. **Generate an API Key in osTicket**
   - Open your osTicket admin panel.
   - Go to **Manage** > **API**.
   - Click **Add New API Key**.
   - In the **IP Address** field, enter the IP address of your ELK server (so only the ELK server can use this API key).
   - Fill in the other required details and save.
   - osTicket will generate an API key for you to use in integrations.

2. **Set Up the Webhook Connector in Kibana**
   - Go to **Stack Management** > **Connectors** in Kibana (make sure you are using an active trial license for 30 days to access all features).
   - Click **Create connector** and choose **Webhook**.
   - For the **URL**, enter your osTicket API endpoint (replace with your osTicket server's IP address):  
     ```
     https://<your-osticket-ip>/osticket/upload/api/tickets.xml
     ```
   - In the **Headers** section, add your API key:
     - Key: `X-API-Key`
     - Value: *your generated osTicket API key*
   - In the **Create action** section, go to GitHub and find an example XML body payload for osTicket's API [see example below], and paste it in the body field.
   - Save and test the connector to ensure it works.

   **Example XML Payload for osTicket API:**
   ```xml
   <ticket>
     <email>{{email}}</email>
     <name>{{name}}</name>
     <subject>{{subject}}</subject>
     <message>{{message}}</message>
     <ip>{{source.ip}}</ip>
   </ticket>
   ```
   - Replace the placeholders with variables or hardcoded values as needed, or use Kibana mustache variables to inject alert details.

3. **Automate Alert Ticket Creation**
   - Go to your detection rule or alert in Kibana.
   - For the **actions** section, select **Webhook** and choose your osTicket connector.
   - Fill out the payload as above with alert-specific variables.
   - Save the rule.

4. **Test the Integration**
   - Trigger a test alert in Kibana.
   - Check osTicket to confirm a ticket is created with the alert details in the body.

5. **Documentation**
   - Add the script location and usage information to your project’s README or documentation.
   - Example:  
     ```
     # osTicket SIEM Integration via Webhook
     - The webhook connector is configured in Kibana to send XML payloads to osTicket's API endpoint.
     - The API key is included as a header for authentication.
     - Alert details are sent in the XML body to automatically create tickets.
     ```

**Tip:**  
You can further customize the ticket body by including relevant log fields, alert metadata, and links back to your SIEM dashboards for faster incident response.

## Detection

- How threats are detected using SIEM rules, correlation, and analytics.

---

## Response: Automated Defense

Automated response in this lab leverages both EDR (Endpoint Detection and Response) and SIEM-to-osTicket integration to ensure incidents are rapidly contained and tracked for follow-up by your help desk or security team.

---

### Using EDR for Automated Response

1. **Deploy EDR Agents**
   - Install EDR agents (such as Elastic Agent, CrowdStrike Falcon, SentinelOne, or similar) on your endpoints (Windows & Linux).
   - Ensure agents are enrolled with your EDR/Fleet management server for centralized policy and event management.

2. **Integrate Elastic Defend**
   - In your ELK/Kibana interface, go to **Integrations**.
   - Search for and select **Elastic Defend** (Elastic Security integration).
   - Click **Add Elastic Defend** and then click **Complete EDR** when prompted.
   - Assign Elastic Defend to the agent policy that is applied to your endpoints (Windows, Linux, etc).
   - This ensures that EDR capabilities (such as malware prevention, host isolation, ransomware protection, etc.) are active on all covered endpoints.

3. **Configure Automated Response Rules**
   - In your EDR console, define policies for automated response actions, such as:
     - Isolating an endpoint upon detection of malware or ransomware.
     - Killing malicious processes automatically.
     - Blocking network connections to known bad IP addresses or domains.
     - Quarantining files or rolling back malicious changes.
   - Example: Elastic Agent with Elastic Security can be configured for "Host Isolation" when certain detection rules are triggered.

4. **Add Defend to Detection Rules**
   - To automate responses directly from SIEM rules:
     - Go to your detection rule in the SIEM/Kibana interface.
     - In the rule's **Actions** section, click **Add Action** and select **Defend**.
     - Set the **Response Action** to your desired automated response (e.g., isolate host, kill process, etc.).
     - Save the rule. Now, when this rule is triggered, the EDR will automatically take the specified action.

5. **Validate Response Actions**
   - Simulate an attack (e.g., run known malware or trigger a brute-force login).
   - Confirm that the EDR agent detects the threat and takes the defined automated response (isolation, process kill, etc.).
   - Check EDR/Fleet dashboards for response event logs and status.

---

### Ticket Generation for Help Desk (osTicket) from Alerts

1. **Alert Triggers in SIEM**
   - When a detection rule in your SIEM (ELK Stack) is triggered (such as malware detection, brute-force attempt, or suspicious activity), the SIEM sends an alert.

2. **Automated Ticket Creation via Webhook/API**
   - The SIEM is integrated with osTicket using a webhook connector (see [Integration with SIEM](#integration-with-siem)).
   - The webhook sends alert details (host, time, type, description, recommended actions, etc.) to the osTicket API endpoint as an XML payload.

3. **Help Desk Workflow**
   - osTicket receives the alert and creates a new ticket, assigning it to the appropriate help desk or SOC analyst queue.
   - The ticket contains all relevant alert details, including:
     - Alert type (e.g., "EDR: Host Isolated due to Ransomware Detection")
     - Impacted endpoint/user
     - Timestamp
     - SIEM/EDR findings and recommended next steps
     - Link to SIEM/EDR dashboard for additional context

4. **Investigation & Remediation**
   - Help desk or SOC staff review tickets in osTicket, investigate impacted endpoints, and follow playbooks for remediation.
   - Analysts can update, escalate, or close tickets as incidents are resolved.

---

**Summary Flow:**

1. **Threat detected by EDR/SIEM** ⟶
2. **EDR takes automated response (isolation, block, etc.)** ⟶
3. **SIEM generates alert** ⟶
4. **SIEM webhook creates osTicket ticket** ⟶
5. **Help desk/SOC notified for follow-up and documentation**

---

**Tip:**  
For even faster response, you can configure EDR and SIEM to include links in ticket bodies that take analysts directly to live dashboards or investigation tools.

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
