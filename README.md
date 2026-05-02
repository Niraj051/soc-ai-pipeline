# 🛡️ SOC AI Pipeline — Automated Alert Detection & Analysis

> An end-to-end Security Operations Center (SOC) lab that automatically detects brute force attacks, enriches alerts with AI (MITRE ATT&CK, IOCs, severity), and delivers analyst-ready reports to Slack — all in real time.




[![Platform](https://img.shields.io/badge/Platform-Ubuntu%2022.04-orange)](https://ubuntu.com/)
[![Splunk](https://img.shields.io/badge/SIEM-Splunk%209.x-green)](https://splunk.com)
[![n8n](https://img.shields.io/badge/Automation-n8n-blueviolet)](https://n8n.io)
[![MITRE ATT&CK](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-red)](https://attack.mitre.org/)

---

## 📌 Table of Contents

- [What This Project Does](#-what-this-project-does)
- [Architecture Overview](#-architecture-overview)
- [Lab Environment](#-lab-environment)
- [Prerequisites](#-prerequisites)
- [Installation & Setup](#-installation--setup)
  - [1. Splunk Server (Ubuntu)](#1-splunk-server-ubuntu)
  - [2. Universal Forwarder (Windows 10)](#2-universal-forwarder-windows-10)
  - [3. n8n Server (Ubuntu)](#3-n8n-server-ubuntu)
  - [4. Groq AI Agent (inside n8n)](#4-groq-ai-agent-inside-n8n)
  - [5. Slack Integration](#5-slack-integration)
- [Correlation Rule — Brute Force Detection](#-correlation-rule--brute-force-detection)
- [n8n Workflow](#-n8n-workflow)
- [AI Agent Prompt](#-ai-agent-prompt)
- [Testing the Pipeline (Kali Linux Attack)](#-testing-the-pipeline-kali-linux-attack)
- [Sample Alert Output](#-sample-alert-output)

---

## 🔍 What This Project Does

This project automates the full SOC alert lifecycle:

```
Attacker (Kali) → Brute Force → Windows 10 → Event Logs
→ Splunk Universal Forwarder → Splunk SIEM (Correlation Rule)
→ Webhook → n8n → Groq AI Agent
→ MITRE ATT&CK + IOC + Severity + Remediation
→ Slack #soc-alerts
```

**Key capabilities:**
- Detects brute force attacks (≥5 failed logins in 60 seconds)
- Maps to MITRE ATT&CK tactics and techniques automatically
- Extracts IOCs: source IP, targeted user, hostname, timestamps
- Assigns severity: Critical / High / Medium / Low
- Generates step-by-step remediation for analysts
- Delivers a structured, analyst-ready report to Slack

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      ATTACK PHASE                           │
│   Kali Linux  ──── Hydra/Medusa ────►  Windows 10 Pro       │
│                  (Brute Force RDP/SMB)   (Target Machine)   │
└──────────────────────────┬──────────────────────────────────┘
                           │  Event ID 4625 (Failed Logon)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     COLLECTION PHASE                        │
│         Splunk Universal Forwarder (on Windows 10)          │
│               Monitors Security Event Log                   │
│                  Forwards via TCP 9997                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     DETECTION PHASE                         │
│              Splunk SIEM (Ubuntu Server)                    │
│   Correlation Rule: count(4625) by src_ip ≥ 5 in 1 min     │
│              → Trigger Alert → Webhook POST                 │
└──────────────────────────┬──────────────────────────────────┘
                           │  HTTP Webhook (JSON)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     ANALYSIS PHASE                          │
│                  n8n (Ubuntu Server)                        │
│   [Webhook Node] → [Groq AI Agent] → [Format Node]         │
│   • MITRE ATT&CK Tactic + Technique                        │
│   • IOC Extraction (IP, user, host, port)                  │
│   • Severity Rating                                        │
│   • Remediation Steps                                      │
└──────────────────────────┬──────────────────────────────────┘
                           │  Slack API
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   NOTIFICATION PHASE                        │
│              Slack  #soc-alerts  Channel                    │
│        Structured analyst report delivered instantly        │
└─────────────────────────────────────────────────────────────┘
```

---

## 🖥️ Lab Environment

| Machine | Role | OS | IP (example) |
|---|---|---|---|
| Kali Linux | Attacker / Red Team | Kali Rolling | 192.168.1.50 |
| Windows 10 Pro | Target / Log Source | Windows 10 22H2 | 192.168.1.100 |
| Splunk Server | SIEM / Detection | Ubuntu 22.04 LTS | 192.168.1.10 |
| n8n Server | Automation / AI | Ubuntu 22.04 LTS | 192.168.1.20 |

> 💡 You can run Splunk and n8n on the same Ubuntu machine if resources are limited.

---

## ✅ Prerequisites

- [VirtualBox](https://www.virtualbox.org/) or VMware (for the lab VMs)
- [Splunk Enterprise](https://www.splunk.com/en_us/download/splunk-enterprise.html) (free trial or dev license)
- [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html) for Windows
- [n8n](https://n8n.io/) — self-hosted on Ubuntu
- [Groq API key](https://console.groq.com/) — free tier is sufficient
- Slack workspace with a bot token and `#soc-alerts` channel
- Basic knowledge of Splunk SPL, Linux CLI, and networking

---

## 🚀 Installation & Setup

### 1. Splunk Server (Ubuntu)

```bash
# Download and install Splunk Enterprise
wget -O splunk.tgz 'https://download.splunk.com/products/splunk/releases/9.2.0/linux/splunk-9.2.0-linux-2.6-amd64.tgz'
tar -xvzf splunk.tgz -C /opt
sudo /opt/splunk/bin/splunk start --accept-license

# Enable Splunk to start on boot
sudo /opt/splunk/bin/splunk enable boot-start

# Enable receiver port for Universal Forwarder data
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:changeme
```

**Configure inputs.conf on the Splunk server:**

```ini
# /opt/splunk/etc/system/local/inputs.conf
[splunktcp://9997]
connection_host = ip
```

**Open firewall ports:**
```bash
sudo ufw allow 9997/tcp    # Universal Forwarder data
sudo ufw allow 8000/tcp    # Splunk Web UI
sudo ufw allow 8089/tcp    # Splunk management
```

---

### 2. Universal Forwarder (Windows 10)

1. Download the [Splunk Universal Forwarder MSI](https://www.splunk.com/en_us/download/universal-forwarder.html) for Windows x64.
2. Run the installer — set the receiving indexer to your **Splunk Server IP:9997**.
3. After install, configure the forwarder to monitor Windows Security logs:

```ini
# C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
[WinEventLog://Security]
disabled = 0
index = windows_logs
sourcetype = WinEventLog:Security
renderXml = false
```

```ini
# C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
[tcpout]
defaultGroup = splunk_server

[tcpout:splunk_server]
server = 192.168.1.10:9997
```

Restart the forwarder:
```powershell
Restart-Service SplunkForwarder
```

---

### 3. n8n Server (Ubuntu)

```bash
 sudo apt install docker.io
 sudo nano docker-compose.yaml
```
```
# docker-compose.yaml
 version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST= N8N_ip
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_SECURE_COOKIE=false
      - GENERIC_TIMEZONE=America/Toronto
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```
```
sudo docker-compose pull
sudo docker-compose up -d
  
# n8n will be available at http://N8N_ip:5678
```

```

Open firewall:
sudo ufw allow 5678/tcp
```

---

### 4. Groq AI Agent (inside n8n)

1. Get a free API key from [console.groq.com](https://console.groq.com/)
2. In n8n → **Settings → Credentials → New Credential → Groq API**
3. Paste your API key and save
4. Import the workflow: `n8n/soc-alert-workflow.json` (see [n8n Workflow](#-n8n-workflow))

---

### 5. Slack Integration

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From Scratch**
2. Enable **Incoming Webhooks** and create one for your `#soc-alerts` channel
3. Copy the Webhook URL
4. In n8n → **Settings → Credentials → New Credential → Slack API**
5. Add your Bot Token (with `chat:write` scope)

---

## 🔎 Correlation Rule — Brute Force Detection

The Splunk alert uses this SPL search (see full file: `splunk/searches/brute_force_detection.spl`):

```spl
index=windows_logs sourcetype="WinEventLog:Security" EventCode=4625
| bucket _time span=1m
| stats count as failed_attempts, 
        values(Account_Name) as targeted_users,
        values(Workstation_Name) as target_host
  by _time, src_ip
| where failed_attempts >= 5
| eval severity=case(
    failed_attempts >= 20, "CRITICAL",
    failed_attempts >= 10, "HIGH",
    failed_attempts >= 5,  "MEDIUM",
    true(),                "LOW"
  )
| table _time, src_ip, failed_attempts, targeted_users, target_host, severity
```

**Alert configuration in Splunk:**
- Trigger: **Per-Result** (each matching row = one alert)
- Schedule: **Real-time** or every 1 minute
- Alert action: **Webhook** → `http://YOUR_N8N_IP:5678/webhook/soc-alert`

---

## ⚙️ n8n Workflow

Import the pre-built workflow from `n8n/soc-alert-workflow.json` into your n8n instance:

1. Open n8n → **Workflows → Import from File**
2. Select `n8n/soc-alert-workflow.json`
3. Update credentials (Groq API key, Slack token)
4. Activate the workflow

**Workflow nodes:**

```
[Webhook Trigger]
    │  Receives JSON from Splunk
    ▼
[Groq AI Agent]
    │  Analyzes log data
    │  Extracts MITRE + IOC + Severity + Remediation
    ▼
[Set / Format Node]
    │  Structures the Slack message
    ▼
[Slack Node]
    │  Posts to #soc-alerts
    ▼
[Done]
```

---

## 🤖 AI Agent Prompt

The following prompt is used inside the Groq AI Agent node (also in `n8n/groq-agent-prompt.txt`):

```
You are a senior SOC analyst. Analyze the following Splunk security alert and provide a structured incident report.

ALERT DATA:
{{ $json }}

Provide your analysis in this exact format:

## 🚨 INCIDENT ALERT

**Severity:** [CRITICAL / HIGH / MEDIUM / LOW]
**Incident Type:** [e.g. Brute Force Attack]
**Timestamp:** [from alert data]

---

### 🎯 MITRE ATT&CK Mapping
- **Tactic:** [e.g. TA0006 - Credential Access]
- **Technique:** [e.g. T1110 - Brute Force]
- **Sub-Technique:** [e.g. T1110.001 - Password Guessing]
- **Reference:** https://attack.mitre.org/techniques/T1110/

---

### 🔍 Indicators of Compromise (IOCs)
- **Source IP:** [attacker IP]
- **Target Host:** [victim hostname]
- **Targeted User:** [username]
- **Failed Attempts:** [count]
- **Timeframe:** [start - end]
- **Protocol/Port:** [e.g. RDP / 3389]

---

### ⚠️ Severity Justification
[2-3 sentences explaining why this severity was assigned]

---

### 🔬 Investigation Steps
1. [Step 1 — what to check in Splunk]
2. [Step 2 — check for successful logins after the brute force]
3. [Step 3 — look for lateral movement]
4. [Step 4 — check for persistence mechanisms]
5. [Step 5 — correlate with threat intel]

---

### 🛠️ Remediation Actions
1. [Immediate action — e.g. block source IP]
2. [Account action — e.g. lock compromised account]
3. [Policy action — e.g. enable account lockout policy]
4. [Detection action — e.g. update firewall rules]
5. [Long-term — e.g. enable MFA]

---

### 📋 Analyst Notes
[Any additional context, false positive assessment, recommended escalation]
```

---

## 🐉 Testing the Pipeline (Kali Linux Attack)

> ⚠️ **Only test on machines you own or have explicit permission to test.**

```bash
# On Kali Linux — run Hydra brute force against Windows RDP
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.1.100

# Or against SMB
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://192.168.1.100

# Alternatively, use a custom small wordlist to trigger the alert quickly
hydra -l admin -P scripts/test-passwords.txt rdp://192.168.1.100 -t 4 -W 1
```

**Expected result within ~60 seconds:**
1. ✅ Splunk detects ≥5 failed logins
2. ✅ Splunk fires alert → sends webhook to n8n
3. ✅ n8n receives JSON → Groq AI analyzes it
4. ✅ Structured report appears in Slack `#soc-alerts`

---
