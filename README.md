# 📌Project Overview

This project demonstrates the implementation of a centralized log management and security monitoring architecture using Splunk Enterprise. The environment bridges an Azure-deployed Windows Server acting as an Active Directory Domain Controller and an Ubuntu Linux VM running a standalone Splunk Indexer instance.   

By installing and deploying the Splunk Universal Forwarder on the Windows target, critical event logs (Authentication failures, successful logons, and account lockouts) are securely piped, parsed, and translated into operational security intelligence.   

## Architecture Overview
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Architecture%20Diagram.png">

## 🛠️ Infrastructure & Technologies Used
SIEM Platform: Splunk Enterprise (v10.2.2)    

Log Ingestion Agent: Splunk Universal Forwarder    

Cloud Infrastructure: Microsoft Azure (Virtual Networks, Network Security Groups, Ubuntu 22.04 LTS & Windows Server VMs)    

Query Language: Splunk Processing Language (SPL)    

Diagnostic Tools: PowerShell Core, Test-NetConnection, PuTTY SSH    

## 📋 Step-by-Step Implementation
### Step 1: Deploying the Architecture in Azure
Provisioned a Linux Ubuntu Standard_B2s virtual machine to meet Splunk's baseline hardware footprint.   

Hardened network parameters by configuring explicit inbound Network Security Group (NSG) variables.   


<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Azure%20NSG.png">

### Step 2: Provisioning Splunk Enterprise on Linux
Managed the instance remotely over SSH via PuTTY.   

Unpacked the deployment binaries, completed initial daemon configuration, and bound the pipeline to receive on port 9997.   

<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Screenshot%202026-05-23%20152627.png">

## Extract and establish the local Debian distribution package
```bash
sudo dpkg -i splunk-10.2.2-linux-amd64.deb
```

## Fire up the application daemon and accept corporate licensing terms
```bash
sudo /opt/splunk/bin/splunk start --accept-license
```

### Step 3: Log Agent Deployment & inputs.conf Engineering
Installed the Universal Forwarder on the primary Active Directory machine.   

Engineered a custom programmatic inputs.conf file path within ```C:\Program Files\SplunkUniversalForwarder\etc\system\local\``` to pull granular Event IDs (4624, 4625, 4740) out of the OS subsystem.  

| Configuration Key | Value |
| :--- | :--- |
| **[WinEventLog://Security]** | |
| disabled | 0 |
| start_from | oldest |
| current_only | 0 |
| evt_resolve_ad_obj | 1 |


# 🔍 Threat Hunting & Essential SPL Queries
To confirm that the telemetry pipeline was successfully processing logs, specific security queries were written to hunt for suspicious indicators.   

## 1. Verification of Logging Pipeline
Verifies the end-to-end data processing loop into the target index.   

Code snippet
```bash
index=windows_logs | head 100
```
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Index%20head%20100.png">

## 2. Identifying Active Brute-Force Activity (EventCode 4625)
Aggregates failed login attempts grouped by target user account names and calling computer nodes.   

Code snippet
```bash
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625
| stats count by Account_Name, Workstation_Name
| sort -count
```
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Index%20Event%204625.png">

## 3. Tracking Account Lockout Events (EventCode 4740)
Maps a programmatic table establishing a forensic timeline of account lockouts, pointing back to the origin machine.   

Code snippet
```bash
index=windows_logs sourcetype=WinEventLog:Security EventCode=4740
| table _time, Account_Name, Caller_Computer_Name
| sort -_time
```
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Index%204740.png">

# 📊 Operational Metrics & Security Dashboards

An operational Windows Security Overview Dashboard was created to visually correlate incoming log volumes. This abstracts complex queries into an executive-level summary chart for security teams.   

## Failed Logins (Last 24h): Configured as a dynamic Bar Chart to isolate account spray velocity.
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Failed%20Login%20Panel.png">


## Login Activity Over Time: Rendered via a structured Line Chart to define normal operational baselines vs anomalies.
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Login%20Activity%20OT%20Panel.png">

## Account Lockouts (Last 7d):
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Account%20Lockout%20Panel.png">


# 🛠️ Specialized Section: Deep-Dive Troubleshooting
## Incident: Log Forwarder Connectivity Deadlock

### Problem Description 
During initial construction, the entry-level Windows Server Universal Forwarder completely failed to transmit authentication logs to the Ubuntu Splunk Indexer. Running Layer 4 diagnostics from an administrative PowerShell window yielded a critical network failure:   

### PowerShell
```bash
Test-NetConnection -ComputerName <Ubuntu_IP> -Port 9997
```
### Status Return: TcpTestSucceeded : False

# 🛠️ DIAGNOSTIC TRIAGE PIPELINE
  
  [Missing inputs.conf] ──► Host service was not binding/listening on Port 9997.
  
  [Azure VNet Mismatch] ──► VMs were provisioned on isolated routing paths.
  
  [NSG Firewall Block]  ──► Inbound security rules dropped arbitrary traffic.
  
Root Cause Analysis (RCA) & Multi-Layer Resolution

## 1. Software Layer Resolution (Missing Listeners)
The Error: The Ubuntu Splunk installation did not automatically initialize local background bindings for logging ports (splunkd was not actively listening).   

The Fix: Configured a native localized listener setup manually via the terminal shell:   

```Bash
sudo mkdir -p /opt/splunk/etc/system/local
sudo nano /opt/splunk/etc/system/local/inputs.conf
```
### Appended the explicit listening stanza
[splunktcp://9997]
disabled = 0

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/system/local
sudo /opt/splunk/bin/splunk restart --run-as-root
```

## 2. Cloud Architecture Layer Resolution (VNet Isolation)
The Error: The Windows Server and Ubuntu instances were deployed into entirely different Azure Virtual Networks (VNets), breaking internal private IP routing.   

The Fix: Redeployed a clean Ubuntu Linux instance, ensuring explicit placement inside the exact VNet container hosting the Active Windows Domain Controller.   

## 3. Network Security Layer Resolution (Firewall Blocks)
The Error: Although Layer 3 ICMP ping traffic routed successfully across endpoints, the Azure Network Security Group (NSG) lacked an inbound rules exception to pass port 9997 traffic safely.   

The Fix: Provisioned a highly restrictive firewall exception rule within the Azure platform interface:   

| Property | Setting |
| :--- | :--- |
| **Source:** | VirtualNetwork |
| **Protocol:** | TCP |
| **Port Range:** | 9997 |
| **Action:** | Allow |
| **Priority:** | 100 |

## Final Verification Results

Following structural network realignment, socket validation testing confirmed an open, unhindered data pipeline:   

PowerShell
```bash
Test-NetConnection -ComputerName 10.0.0.5 -Port 9997
```
PingSucceeded      : True  (Layer 3 Private Routing OK)
TcpTestSucceeded   : True  (Layer 4 Splunk Service Binding OK)
<img width="1000" alt="image" src="https://github.com/DavidPatrick92/Splunk_SIEM_-_Log_Analysis/blob/main/Screenshots/Succesful%20Ping.png">

# 📈 Key Technical Takeaways

## Automated Detection Mechanisms 
Engineered persistent scheduled query patterns inside the SIEM configuration to generate tracking alerts for brute-force patterns over a sliding 15-minute window.   

## Proactive Defenses 
Gained experience distinguishing casual user input mistakes from targeted account enumeration vectors by studying event density.   

## Enterprise Infrastructure Design 
Deepened knowledge regarding software configurations (inputs.conf / outputs.conf), cloud firewalls, and data transport layers in enterprise environments.
