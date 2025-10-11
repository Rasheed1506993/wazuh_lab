# Comprehensive Wazuh Proof of Concept Guide (Windows)

## Guide Version

This guide is compatible with Wazuh 4.2.0 and newer versions.
[Click here for older versions](https://github.com/wazuh/wazuh/wiki/Proof-of-concept-guide/b3681f865eb671a0b0d13a1b1012477fd0d932ba).

## Table of Contents

- [Guide Version](#guide-version)
- [Introduction](#introduction)
- [MITRE ATT&CK Mapping & Threat Simulation](#mitre-attck-mapping--threat-simulation)
- [Brute Force Attack Detection - RDP](#brute-force-attack-detection---rdp)
- [File Integrity Monitoring](#file-integrity-monitoring)
- [Malware Detection and Removal - VirusTotal Integration](#malware-detection-and-removal---virustotal-integration)
- [Vulnerability Detection](#vulnerability-detection)
- [Slack Integration](#slack-integration)
- [Conclusion](#conclusion)

## Introduction

This document explains how to set up a Wazuh environment to test various product capabilities on Windows endpoints. It assumes the following components are already installed:

- Elasticsearch + Kibana + Wazuh Kibana Plugin
- Wazuh Manager + Filebeat (for Elasticsearch integration)
- Wazuh Agent (Windows)

A good guide on how to install these components can be found in the [Wazuh Installation Documentation](https://github.com/Rasheed1506993/wazuh_lab/blob/main/Wazuh_Server_&_Agent_Installation_Documentation.pdf)

The sections below explain the configurations required to set up different use cases on Windows endpoints, with a special focus on mapping security events to the **MITRE ATT&CK framework** for enhanced threat intelligence and SOC analysis.

---

## MITRE ATT&CK Mapping & Threat Simulation

This section demonstrates how to **simulate adversary behaviors** on an Ubuntu server, generate **Wazuh alerts**, and map them to the **MITRE ATT&CK framework**. The goal is to showcase threat detection, analysis, and reporting skills relevant for SOC analyst roles.

### 🚀 Overview
- **Tooling**: Wazuh SIEM, Ubuntu agent, Wazuh dashboard
- **Focus**: Threat simulation, alert generation, MITRE ATT&CK mapping
- **SOC Relevance**: Demonstrates ability to connect logs → attacker behavior → defensive actions

### 🛠️ Simulated Adversary Behaviors

#### 1. SSH Brute Force Attack (T1110 - Credential Access)

**Objective**: Simulate credential brute force attacks to test detection capabilities.

**Commands Executed on Ubuntu Server**:
```bash
# Multiple failed SSH login attempts
ssh invaliduser@localhost
ssh invaliduser@localhost
ssh invaliduser@localhost
```

**Expected Wazuh Alert**:
- **Rule ID**: 5710
- **MITRE ATT&CK Technique**: T1110 - Brute Force
- **Tactic**: Credential Access
- **Description**: Multiple authentication failures detected

#### 2. Privilege Escalation (T1068 - Elevation of Privilege)

**Objective**: Simulate privilege escalation attempts using sudo.

**Commands Executed**:
```bash
# Attempt to execute privileged commands
sudo su
sudo cat /etc/shadow
sudo useradd testuser
```

**Expected Wazuh Alert**:
- **Rule ID**: 5402
- **MITRE ATT&CK Technique**: T1068 - Exploitation for Privilege Escalation
- **Tactic**: Privilege Escalation
- **Description**: Suspicious sudo command execution detected

#### 3. User Account Creation (T1136 - Create Account)

**Objective**: Simulate unauthorized user account creation.

**Commands Executed**:
```bash
# Create new user accounts
sudo useradd -m malicious_user
sudo useradd -m backdoor_account
```

**Expected Wazuh Alert**:
- **Rule ID**: 5901
- **MITRE ATT&CK Technique**: T1136 - Create Account
- **Tactic**: Persistence
- **Description**: New user account created on the system

### 📊 Analyzed Alerts in Wazuh Dashboard

#### Steps to Analyze Events:

1. **Navigate to Wazuh Dashboard**
   - Go to: Security Events → Events

**Screenshot: Wazuh Dashboard Report - Comprehensive Security Event Overview**

<img width="1710" height="1112" alt="Step 4 - report" src="https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/WazuhDashboardReport.png" />

*This screenshot shows: All three attack simulations detected with severity distribution, timeline, and complete event overview*

2. **Apply Filters for Each Event Type**:
   
   **SSH Brute Force Filter**:
   ```
   rule.id:5710
   rule.description:*authentication*failed*
   ```
   
   **Privilege Escalation Filter**:
   ```
   rule.id:5402
   rule.description:*sudo*
   ```
   
   **User Creation Filter**:
   ```
   rule.id:5901
   data.audit.command:*useradd*
   ```
**Screenshot: Example of New User Creation Alert**

<img width="1710" height="1112" alt="Step 4 - alerts" src="https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/UserCreationFilter.png" />

*This screenshot shows: Detailed view of T1136 detection with audit log showing useradd command and user creation metadata*

3. **Verify Event Timeline**
   - Confirm events appear in the correct time range
   - Document the sequence of attacker activities

4. **Extract Key Information**:
   - Rule IDs and severity levels
   - Source IP addresses
   - Affected usernames
   - Timestamps
   - MITRE ATT&CK mappings

### 📈 Created Visualizations

#### Visualization 1: SSH Brute Force Attempts Trend
- **Type**: Line Chart
- **X-Axis**: Time
- **Y-Axis**: Count of failed authentication attempts
- **Purpose**: Track brute force attack patterns over time

#### Visualization 2: Privilege Escalation by User
- **Type**: Bar Chart
- **X-Axis**: Username
- **Y-Axis**: Count of sudo commands
- **Purpose**: Identify users with suspicious privilege escalation activity

#### Visualization 3: User Creation Events
- **Type**: Data Table
- **Columns**: Timestamp, Username Created, Creator User, Source IP
- **Purpose**: Monitor unauthorized account creation attempts

### 🎯 Custom MITRE ATT&CK Threat Simulation Dashboard

#### Dashboard Components:

1. **Failed SSH Attempts Trend**
   - Real-time monitoring of brute force activities
   - Alert threshold indicators

2. **Privilege Escalation Attempts by User**
   - User behavior analytics
   - Anomaly detection visualization

3. **User Creation Events Timeline**
   - Persistence activity tracking
   - Account creation audit trail

#### Creating the Dashboard:

1. Navigate to Kibana → Dashboard → Create Dashboard
2. Add the three visualizations created above
3. Arrange them in a logical layout:
   - Top: SSH Brute Force Timeline (full width)
   - Middle Left: Privilege Escalation Chart
   - Middle Right: User Creation Table
4. Add filters panel for interactive analysis
5. Save as "MITRE ATT&CK Threat Simulation Dashboard"

**Screenshot: Custom MITRE ATT&CK Dashboard Overview**

<img width="1710" height="1112" alt="Step 4 - dashboard" src="https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/CustomMITREATT&CK.png" />

*This screenshot shows: Complete view of custom dashboard with all three visualizations, interactive filters, and time range selector*

### 🗺️ MITRE ATT&CK Mapping Table

| Event Type | Rule ID | MITRE ATT&CK Technique | Tactic | Severity |
|------------|---------|------------------------|--------|----------|
| SSH Brute Force | 5710 | T1110 - Brute Force | Credential Access | High |
| Privilege Escalation (sudo) | 5402 | T1068 - Exploitation for Privilege Escalation | Privilege Escalation | High |
| User Account Creation | 5901 | T1136 - Create Account | Persistence | Medium |

### 📸 MITRE ATT&CK Mapping Evidence

The following screenshots provide visual proof of MITRE ATT&CK technique mapping in Wazuh alerts:

#### 1. T1136 - Create Account (User Account Creation)

<img width="1710" height="1112" alt="Step 4 - mapping - new user account creation" src="https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/CreateAccount.png" />

*Shows: Alert details with MITRE ATT&CK T1136 mapping, including technique name, tactic (Persistence), and full audit trail*

---

#### 2. T1068 - Exploitation for Privilege Escalation

<img width="1710" height="1112" alt="Step 4 - mapping - privilege escalation" src="https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/Exploitation.png" />

*Shows: Privilege escalation alert with T1068 mapping, sudo command details, and elevated privilege context*

---

#### 3. T1110 - Brute Force (SSH Authentication)

<img width="1710" height="1112" alt="Step 4 - mapping - ssh brute force" src="https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/SSHAuthentication.png" />

*Shows: SSH brute force detection with T1110 mapping, failed authentication count, source IP, and targeted username*

---

**Key Information Visible in Screenshots:**
- ✅ MITRE ATT&CK Technique ID (T1110, T1068, T1136)
- ✅ Tactic Classification (Credential Access, Privilege Escalation, Persistence)
- ✅ Complete alert metadata (timestamp, severity, source)
- ✅ Command/action details from system logs
- ✅ Wazuh rule ID and description

1. **Wazuh Dashboard Report**
   - Shows comprehensive security event overview
   - Displays all three attack simulations detected
   - Includes severity distribution and timeline

2. **New User Creation Alert Example**
   - Detailed view of T1136 detection
   - Shows audit log with useradd command
   - Includes user creation metadata

3. **Custom Dashboard Overview**
   - Complete view of MITRE ATT&CK dashboard
   - All three visualizations working together
   - Interactive filters and time range selector

4. **MITRE ATT&CK Mapping Evidence**:
   - **New User Account Creation**: Shows T1136 mapping with technique details
   - **Privilege Escalation**: Shows T1068 mapping with sudo command details
   - **SSH Brute Force**: Shows T1110 mapping with failed authentication logs

### 🔍 SOC Analyst Workflow

#### Detection Phase:
1. Alert triggered in Wazuh
2. Analyst reviews dashboard for context
3. Identifies MITRE ATT&CK technique
4. Assesses severity and impact

#### Analysis Phase:
1. Investigates related events in timeline
2. Correlates multiple techniques (kill chain)
3. Identifies affected assets and users
4. Documents findings in incident report

#### Response Phase:
1. Contains affected systems
2. Blocks malicious IP addresses
3. Removes unauthorized accounts
4. Escalates to incident response team if needed

### 📝 Integration with Main Guide

The MITRE ATT&CK mapping demonstrated above applies to all security events throughout this guide:

- **Brute Force RDP Attacks** → Maps to T1110 (Brute Force)
- **File Integrity Changes** → Maps to T1565 (Data Manipulation)
- **Malware Detection** → Maps to T1204 (User Execution)
- **Vulnerability Exploitation** → Maps to T1203 (Exploitation for Client Execution)

Each subsequent section will include relevant MITRE ATT&CK technique mappings to provide comprehensive threat intelligence context.

---

## Brute Force Attack Detection - RDP

Brute force attacks on RDP (Remote Desktop Protocol) are common attack vectors on Windows systems. Wazuh provides out-of-the-box rules capable of identifying brute force attacks by correlating multiple authentication failure events.

### 🗺️ MITRE ATT&CK Mapping

- **Technique**: T1110 - Brute Force
- **Sub-technique**: T1110.001 - Password Guessing
- **Tactic**: Credential Access
- **Description**: Adversaries may use brute force techniques to gain access to accounts when passwords are unknown or obtained

### Configuration

- Ensure the Remote Desktop service is enabled on the monitored Windows machine.

![Remote Desktop Service](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image1.png)

- Ensure the Wazuh agent is installed and running on the Windows machine.

```powershell
Get-Service WazuhSvc
```

- To perform an automated attack from another machine, you can use the `hydra` tool:

![Hydra Tool](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image3.png)

```bash
# From a Linux machine
hydra -l Target_user -P password.txt <windows-ip-address> rdp
```

### Steps to Generate Alerts

```bash
hydra -l Me -P /home/kali/password.txt 192.168.201.1 rdp
```

![Hydra Attack](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image4.png)

### Alerts

Related alerts can be found in Kibana using:

- `rule.id:(60122 OR 60137)`
- `rule.description:*brute*force*`
- `data.win.system.eventID:4625` (Login failure)
- `data.mitre.technique:T1110*` (MITRE ATT&CK filter)

![Kibana Alerts 1](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image5.png)

![Kibana Alerts 2](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image6.png)

![Kibana Alerts 3](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image7.png)

### Alert Details

Each RDP brute force alert contains:
- Source IP address of attacker
- Target username
- Number of failed attempts
- Event ID 4625 (Windows login failure)
- **MITRE ATT&CK Technique**: T1110
- Alert severity level
- Timestamp of attack

### Affected Endpoint

- Windows

---

## File Integrity Monitoring

File Integrity Monitoring (FIM) is a vital security feature that monitors changes to critical files and directories. Wazuh can detect file creation, modification, or deletion in real-time.

### 🗺️ MITRE ATT&CK Mapping

- **Technique**: T1565 - Data Manipulation
- **Sub-technique**: T1565.001 - Stored Data Manipulation
- **Tactic**: Impact
- **Related Techniques**:
  - T1070.004 - File Deletion (Defense Evasion)
  - T1036 - Masquerading (Defense Evasion)

### Configuration

On the monitored Windows machine, edit the Wazuh agent configuration file located at: `C:\Program Files (x86)\ossec-agent\ossec.conf`

**Screenshot: Shows the path to ossec.conf file in Windows Explorer**

![ossec.conf path](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image8.png)

- Add or ensure the following directories are present in the `<syscheck>` section:

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>300</frequency>
  <scan_on_start>yes</scan_on_start>
  
  <!-- Monitor Administrator's Desktop -->
  <directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\Administrator\Desktop</directories>
  
  <!-- Monitor Downloads folder -->
  <directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\Administrator\Downloads</directories>
  
  <!-- Monitor Documents folder -->
  <directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\Administrator\Documents</directories>
  
  <!-- Monitor Wazuh installation directory -->
  <directories check_all="yes" report_changes="yes" realtime="yes">C:\Program Files (x86)\ossec-agent</directories>
  
  <!-- Monitor Startup files -->
  <directories check_all="yes" realtime="yes">C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup</directories>
</syscheck>
```

### Parameter Explanation:

- `check_all="yes"`: Monitor all changes (size, permissions, owner, etc.)
- `report_changes="yes"`: Report file content changes
- `realtime="yes"`: Real-time monitoring
- `frequency="300"`: Periodic scan every 300 seconds (5 minutes)

- After modifying the configuration file, restart the Wazuh Agent service:

![Restart Service](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image11.png)

- Verify that the service restarted successfully:

```powershell
Get-Service WazuhSvc
```

### Steps to Generate Alerts

Make changes to files in the monitored directories:

**1. Create a new file:**

```powershell
New-Item -Path "C:\Users\<USERNAME>\Desktop\test_file.txt" -ItemType File -Value "This is a test file for FIM monitoring"
```

![Create File](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image13.png)

**2. Modify an existing file:**

```powershell
Add-Content -Path "C:\Users\RAOOF\Desktop\test_file.txt" -Value "Modified content - $(Get-Date)"
```

![Modify File](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image14.png)

**3. Delete a file:**

```powershell
Remove-Item -Path "C:\Users\RAOOF\Desktop\test_file.txt"
```

![Delete File](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image15.png)

**4. Create multiple files for comprehensive testing:**

```powershell
1..5 | ForEach-Object {
    New-Item -Path "C:\Users\RAOOF\Desktop\test_$_.txt" -ItemType File -Value "Test file number $_"
}
```

![Multiple Files](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image16.png)

### Alerts

Related alerts can be found in Kibana using:

**Search Queries:**
- `rule.groups:"syscheck"` - All FIM alerts
- `syscheck.event:"added"` - Added files
- `syscheck.event:"modified"` - Modified files
- `syscheck.event:"deleted"` - Deleted files
- `syscheck.path:"*Desktop*"` - Desktop changes
- `data.mitre.technique:T1565*` - MITRE ATT&CK filter

**Screenshot: Shows Kibana Dashboard interface with FIM alerts - General view of all alerts**

![FIM Dashboard](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image17.png)

**Screenshot: Shows details of file addition alert (syscheck.event: added)**

![File Added](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image18.png)

**Screenshot: Shows file modification alert (syscheck.event: modified) with logged changes**

![File Modified 1](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image19.png)

![File Modified 2](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image20.png)

**Screenshot: Shows file deletion alert (syscheck.event: deleted)**

![File Deleted](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image21.png)

**Screenshot: Shows a Visualization in Kibana displaying the distribution of FIM events by type (added, modified, deleted)**

![FIM Visualization](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image22.png)

### Additional Information Available in Alerts:

- Full file path
- Date and time of change
- Event type (added, modified, deleted)
- Permission changes
- Owner changes
- File size changes
- File hash (MD5, SHA1, SHA256)
- **MITRE ATT&CK technique mapping**

**Screenshot: Shows complete JSON of an alert displaying all available fields including hashes**

![JSON Details](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image23.png)

### Affected Endpoint

- Windows

---

## Malware Detection and Removal - VirusTotal Integration

Wazuh has the ability to integrate with VirusTotal API and run a query when a file change is detected. When a malicious file is detected, Wazuh can be configured to remove it automatically.

### 🗺️ MITRE ATT&CK Mapping

- **Technique**: T1204 - User Execution
- **Sub-technique**: T1204.002 - Malicious File
- **Tactic**: Execution
- **Related Techniques**:
  - T1105 - Ingress Tool Transfer (Command and Control)
  - T1027 - Obfuscated Files or Information (Defense Evasion)

### Prerequisites

- VirusTotal API Key (can be obtained for free from https://www.virustotal.com)
- Wazuh Agent installed on Windows
- Python installed on Wazuh Manager

**Screenshot: Shows VirusTotal page for creating an account and obtaining an API Key**

![VirusTotal Account](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image24.png)

**Screenshot: Shows API Key page in VirusTotal account**

![VirusTotal API Key](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image25.png)

### Configuration on Wazuh Manager

#### 1. Enable VirusTotal Integration

Edit the `/var/ossec/etc/ossec.conf` file on the Wazuh Manager:

```xml
<ossec_config>
    <integration>
        <name>virustotal</name>
        <api_key>YOUR_VIRUSTOTAL_API_KEY_HERE</api_key>
        <rule_id>100200,100201</rule_id>
        <alert_format>json</alert_format>
    </integration>
</ossec_config>
```

**Screenshot: Shows editing ossec.conf file on Wazuh Manager with VirusTotal integration configuration**

![VirusTotal Config](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image26.png)

#### 2. Create Custom Rules

Add the following rules to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="syscheck,pci_dss_11.5,nist_800_53_SI.7,">
    <!-- Rules for Windows systems - Desktop monitoring -->
    <rule id="100200" level="7">
        <if_sid>550</if_sid>
        <field name="file">\\Users\\RAOOF\\Desktop</field>
        <description>File modified in RAOOF Desktop directory.</description>
    </rule>
    
    <rule id="100201" level="7">
        <if_sid>554</if_sid>
        <field name="file">\\Users\\RAOOF\\Desktop</field>
        <description>File added to RAOOF Desktop directory.</description>
    </rule>
    
    <!-- Rules for Downloads folder -->
    <rule id="100202" level="7">
        <if_sid>550</if_sid>
        <field name="file">\\Users\\RAOOF\\Downloads</field>
        <description>File modified in Downloads directory.</description>
    </rule>
    
    <rule id="100203" level="7">
        <if_sid>554</if_sid>
        <field name="file">\\Users\\RAOOF\\Downloads</field>
        <description>File added to Downloads directory.</description>
    </rule>
</group>

<group name="virustotal,">
    <!-- Rule for detected malicious files with MITRE mapping -->
    <rule id="100300" level="12">
        <if_sid>87105</if_sid>
        <description>VirusTotal: Malicious file detected - $(data.virustotal.source.file)</description>
        <mitre>
            <id>T1204.002</id>
        </mitre>
    </rule>
</group>
```

**Screenshot: Shows local_rules.xml file content with custom rules**

![Custom Rules](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image27.png)

#### 3. Create Decoder for Active Response

Edit `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="ar_log_fields">
    <parent>ar_log</parent>
    <regex offset="after_parent">^(\S+) - Removed threat located at (\S+)</regex>
    <order>script_name, path</order>
</decoder>
```

**Screenshot: Shows local_decoder.xml file content**

![Local Decoder](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image28.png)

#### 4. Create Automatic Removal Rules

Add to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="virustotal,">
    <rule id="100092" level="12">
        <if_sid>657</if_sid>
        <match>Successfully removed threat</match>
        <description>Successfully removed threat located at $(parameters.alert.data.virustotal.source.file)</description>
    </rule>

    <rule id="100093" level="12">
        <if_sid>657</if_sid>
        <match>Error removing threat</match>
        <description>Error removing threat located at $(parameters.alert.data.virustotal.source.file)</description>
    </rule>
</group>
```

**Screenshot: Shows automatic removal rules in local_rules.xml**

![Removal Rules](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image29.png)

#### 5. Configure Active Response on Wazuh Manager

Add to `/var/ossec/etc/ossec.conf`:

```xml
<ossec_config>
    <command>
        <name>remove-threat-windows</name>
        <executable>remove-threat.cmd</executable>
        <timeout_allowed>no</timeout_allowed>
    </command>

    <active-response>
        <disabled>no</disabled>
        <command>remove-threat-windows</command>
        <location>local</location>
        <rules_id>87105</rules_id>
    </active-response>
</ossec_config>
```

#### 6. Restart Wazuh Manager

```bash
systemctl restart wazuh-manager
```

### Configuration on Windows Agent

#### 1. Configure File Integrity Monitoring

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf`:

```xml
<syscheck>
    <disabled>no</disabled>
    <frequency>300</frequency>
    <scan_on_start>yes</scan_on_start>
    
    <!-- Real-time Desktop monitoring -->
    <directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\RAOOF\Desktop</directories>
    
    <!-- Monitor Downloads folder -->
    <directories check_all="yes" report_changes="yes" realtime="yes">C:\Users\RAOOF\Downloads</directories>
</syscheck>
```

#### 2. Create Active Response Script

Create the file `remove-threat.cmd` at: `C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.cmd`

```batch
@echo off
setlocal enableDelayedExpansion

:: Get JSON information from stdin
set "json="
for /f "delims=" %%i in ('more') do set "json=!json!%%i"

:: Extract file path from JSON (simplified method)
:: In production environment, use a tool like jq for Windows
echo %json% > C:\Program Files (x86)\ossec-agent\active-response.log

:: Extract path manually (can be improved)
for /f "tokens=2 delims=:" %%a in ('echo %json% ^| findstr /i "file"') do (
    set "filepath=%%a"
)

:: Remove quotes and commas
set "filepath=%filepath:"=%"
set "filepath=%filepath:,=%"
set "filepath=%filepath: =%"

:: Log the action
echo %date% %time% - Attempting to remove file: %filepath% >> "C:\Program Files (x86)\ossec-agent\active-responses.log"

:: Attempt to delete the file
if exist "%filepath%" (
    del /F /Q "%filepath%"
    if !errorlevel! equ 0 (
        echo %date% %time% - remove-threat.cmd - Successfully removed threat >> "C:\Program Files (x86)\ossec-agent\active-responses.log"
    ) else (
        echo %date% %time% - remove-threat.cmd - Error removing threat >> "C:\Program Files (x86)\ossec-agent\active-responses.log"
    )
) else (
    echo %date% %time% - remove-threat.cmd - File not found: %filepath% >> "C:\Program Files (x86)\ossec-agent\active-responses.log"
)

exit /b 0
```

**Screenshot: Shows creation of remove-threat.cmd file in the correct path**

![Remove Threat CMD Path](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image30.png)

**Screenshot: Shows remove-threat.cmd file content in Notepad**

![Remove Threat CMD Content](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image31.png)

#### 3. Create Enhanced PowerShell Script (Optional)

You can also create `remove-threat.ps1` for a better solution:

```powershell
# Read JSON from stdin
$input_json = [Console]::In.ReadToEnd()

# Convert JSON to PowerShell object
try {
    $alert = $input_json | ConvertFrom-Json
    $file_path = $alert.parameters.alert.data.virustotal.source.file
    
    # Log file
    $log_file = "C:\Program Files (x86)\ossec-agent\active-responses.log"
    
    # Log the action
    $timestamp = Get-Date -Format "yyyy/MM/dd HH:mm:ss"
    Add-Content -Path $log_file -Value "$timestamp - Attempting to remove: $file_path"
    
    # Delete the file if it exists
    if (Test-Path $file_path) {
        Remove-Item -Path $file_path -Force -ErrorAction Stop
        Add-Content -Path $log_file -Value "$timestamp - remove-threat.ps1 - Successfully removed threat"
        exit 0
    } else {
        Add-Content -Path $log_file -Value "$timestamp - remove-threat.ps1 - File not found"
        exit 1
    }
}
catch {
    $timestamp = Get-Date -Format "yyyy/MM/dd HH:mm:ss"
    $log_file = "C:\Program Files (x86)\ossec-agent\active-responses.log"
    Add-Content -Path $log_file -Value "$timestamp - remove-threat.ps1 - Error: $_"
    exit 1
}
```

**Screenshot: Shows remove-threat.ps1 file content**

![Remove Threat PS1](https://raw.githubusercontent.com/Rasheed1506993/wazuh_lab/main/media/image32.png)

#### 4. Restart Wazuh Agent

```powershell
Restart-Service -Name wazuh
```

### Steps to Generate Alerts

#### 1. Download EICAR Test File

EICAR is a standard test file for antivirus software; it is harmless but detected as malware.

**Using PowerShell:**

```powershell
# Navigate to Desktop
cd C:\Users\RAOOF\Desktop

# Download EICAR file
Invoke-WebRequest -Uri "http://www.eicar.org/download/eicar.com" -OutFile "eicar.com"
```

**Or using browser:**
- Open browser and go to: `http://www.eicar.org/download/eicar.com`
- Save the file to Desktop

#### 2. Create EICAR File Manually (Alternative)

```powershell
# Create EICAR file manually
$eicar = 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*'
Set-Content -Path "C:\Users\Administrator\Desktop\eicar_test.txt" -Value $eicar
```

#### 3. Monitor Logs

Monitor the active response log file:

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\active-responses.log" -Wait
```

### Alerts

Related alerts can be found in Kibana:

**Search Queries:**

1. **General VirusTotal Alerts:**
```
rule.groups:"virustotal"
```

2. **Detected Malicious Files:**
```
data.virustotal.positives:>0
```

3. **Successful Removal Alerts:**
```
rule.id:100092
```

4. **Failed Removal Alerts:**
```
rule.id:100093
```

5. **Search for Specific File:**
```
data.virustotal.source.file:"*eicar*"
```

6. **MITRE ATT&CK Filter:**
```
data.mitre.technique:T1204*
```

### Additional Information in Alerts:

- **Scan ID**: Unique scan identifier in VirusTotal
- **Positives**: Number of antivirus engines that detected the file as malicious
- **Total**: Total number of engines that scanned the file
- **Permalink**: Direct link to scan results on VirusTotal website
- **File hashes**: MD5, SHA1, SHA256 of the file
- **Scan date**: Date and time of scan
- **MITRE ATT&CK Technique**: T1204.002 - Malicious File

### Verify Successful Automatic Removal

#### 1. Check Active Response Log

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\active-responses.log" | Select-Object -Last 10
```

#### 2. Verify File Does Not Exist

```powershell
Test-Path "C:\Users\Administrator\Desktop\eicar.com"
# Should return False if deleted successfully
```

### Troubleshooting

#### Issue: Files Not Being Sent to VirusTotal

**Solutions:**

1. **Verify API Key Validity:**

```bash
# On Wazuh Manager
grep -i "virustotal" /var/ossec/logs/ossec.log
```

2. **Check API Limit:**
   - Free key is limited to 4 requests per minute
   - Ensure not exceeding the limit

#### Issue: Active Response Not Working

**Solutions:**

1. **Check File Permissions:**

```powershell
Get-Acl "C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.cmd" | Format-List
```

2. **Check Agent Log:**

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" | Select-Object -Last 20
```

3. **Test Script Manually:**

```powershell
# Create test JSON file
$test_json = @{
    command = "add"
    parameters = @{
        alert = @{
            data = @{
                virustotal = @{
                    source = @{
                        file = "C:\Users\Administrator\Desktop\test_file.txt"
                    }
                }
            }
        }
    }
} | ConvertTo-Json -Depth 10

# Create test file
New-Item -Path "C:\Users\Administrator\Desktop\test_file.txt" -ItemType File -Value "test"

# Run script
$test_json | & "C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.ps1"
```

### Affected Endpoint

- Windows

---

## Vulnerability Detection

Wazuh is capable of detecting whether installed applications contain unpatched security vulnerabilities (CVEs) on the monitored system. This feature is very important for proactive system security management.

### 🗺️ MITRE ATT&CK Mapping

- **Technique**: T1203 - Exploitation for Client Execution
- **Tactic**: Execution
- **Related Techniques**:
  - T1068 - Exploitation for Privilege Escalation
  - T1210 - Exploitation of Remote Services (Lateral Movement)
  - T1190 - Exploit Public-Facing Application (Initial Access)

### Configuration on Wazuh Manager

#### 1. Enable Vulnerability Detector

Edit `/var/ossec/etc/ossec.conf` on the Wazuh Manager:

```xml
<ossec_config>
  <vulnerability-detector>
    <enabled>yes</enabled>
    <interval>5m</interval>
    <ignore_time>6h</ignore_time>
    <run_on_start>yes</run_on_start>

    <!-- Windows OS vulnerabilities -->
    <provider name="msu">
      <enabled>yes</enabled>
      <update_interval>1h</update_interval>
    </provider>

    <!-- National Vulnerability Database -->
    <provider name="nvd">
      <enabled>yes</enabled>
      <update_from_year>2010</update_from_year>
      <update_interval>1h</update_interval>
    </provider>

  </vulnerability-detector>
</ossec_config>
```

#### Parameter Explanation:

- `enabled`: Enable detector
- `interval`: Interval between scans (5 minutes)
- `ignore_time`: Ignore old vulnerabilities (6 hours)
- `run_on_start`: Run scan on startup
- `provider name="msu"`: Microsoft Security Updates vulnerability provider
- `provider name="nvd"`: National Vulnerability Database

#### 2. Restart Wazuh Manager

```bash
systemctl restart wazuh-manager
```

#### 3. Verify Database Update

```bash
# Check CVE database
ls -lh /var/ossec/queue/vulnerabilities/cve.db

# Monitor update process in log
tail -f /var/ossec/logs/ossec.log | grep -i vulnerability
```

### Configuration on Windows Agent

#### 1. Enable syscollector to Collect Software Information

Edit `C:\Program Files (x86)\ossec-agent\ossec.conf`:

```xml
<wodle name="syscollector">
  <disabled>no</disabled>
  <interval>1h</interval>
  <scan_on_start>yes</scan_on_start>
  
  <!-- Collect hardware information -->
  <hardware>yes</hardware>
  
  <!-- Collect operating system information -->
  <os>yes</os>
  
  <!-- Collect network information -->
  <network>yes</network>
  
  <!-- Collect installed software list - important for vulnerability detection -->
  <packages>yes</packages>
  
  <!-- Collect installed updates information (Hotfixes) -->
  <hotfixes>yes</hotfixes>
  
  <!-- Collect open ports information -->
  <ports all="no">yes</ports>
  
  <!-- Collect running processes information -->
  <processes>yes</processes>
</wodle>
```

#### Parameter Explanation:

- `packages`: Collect list of installed software
- `hotfixes`: Collect list of installed Windows updates
- `interval`: Scan frequency (every hour)
- `scan_on_start`: Run scan on agent startup

#### 2. Restart Wazuh Agent

```powershell
Restart-Service -Name wazuh
```

#### 3. Verify syscollector is Working

```powershell
# Display last events in Wazuh log
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" | Select-Object -Last 30 | Select-String -Pattern "syscollector"
```

### Steps to Generate Alerts

Security vulnerabilities will be detected automatically based on:
1. Software installed on the system
2. Missing Windows updates
3. Operating system version

#### 1. View Installed Software

```powershell
# Display list of installed software
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | 
    Select-Object DisplayName, DisplayVersion, Publisher | 
    Format-Table -AutoSize
```

#### 2. Check Installed Updates

```powershell
# Display list of installed Hotfixes
Get-HotFix | Sort-Object -Property InstalledOn -Descending | 
    Format-Table -AutoSize
```

#### 3. View Operating System Information

```powershell
# Operating system information
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer
```

#### 4. Simulate System with Vulnerabilities (Optional - For Testing Only)

To test the feature better, you can:
- Use an old Windows system (like Windows Server 2012 or Windows 7)
- Install old software known to have vulnerabilities
- Don't install the latest security updates

**Important Note:** This is for testing only in an isolated environment!

### Alerts

Related alerts can be found in Kibana:

#### Search Queries:

1. **All Vulnerability Alerts:**
```
rule.groups:"vulnerability-detector"
```

2. **Vulnerabilities by severity level:**
```
data.vulnerability.severity:"High"
data.vulnerability.severity:"Critical"
data.vulnerability.severity:"Medium"
```

3. **Vulnerabilities in Specific Software:**
```
data.vulnerability.package.name:"<package_name>"
```

4. **Specific CVE Vulnerabilities:**
```
data.vulnerability.cve:"CVE-2021-*"
```

5. **Vulnerabilities with Available Patch:**
```
data.vulnerability.status:"Available"
```

6. **MITRE ATT&CK Filter:**
```
data.mitre.technique:T1203* OR data.mitre.technique:T1068*
```

### Alert Details

Each vulnerability alert contains the following information:

- **CVE ID**: Vulnerability number (e.g., CVE-2021-34527)
- **Title**: Vulnerability title
- **Severity**: Severity level (Critical, High, Medium, Low)
- **Package**: Name of affected software or component
- **Version**: Version of affected software
- **Architecture**: System architecture (x86, x64)
- **CVSS Score**: CVSS score (from 0 to 10)
- **Published Date**: Vulnerability publication date
- **Updated Date**: Last update date
- **Status**: Patch status (Pending, Available, Fixed)
- **References**: Links for more information
- **MITRE ATT&CK Technique**: Mapped exploitation technique

### Visualizations in Kibana

You can create useful dashboards to display vulnerabilities:

#### 1. Vulnerability Distribution by Severity
Create a Pie Chart displaying vulnerability distribution by severity level

#### 2. Top Affected Software
Create a Bar Chart displaying software with most security vulnerabilities

#### 3. Vulnerability Trend Over Time
Create a Line Chart displaying vulnerability detection over time

#### 4. CVSS Scores
Create a distribution of CVSS scores for detected vulnerabilities

#### 5. MITRE ATT&CK Technique Distribution
Create a visualization showing which exploitation techniques are most common in detected vulnerabilities

### Recommendations and Actions

When security vulnerabilities are detected:

1. **Review High-Priority Vulnerabilities:**
   - Focus on Critical and High-rated vulnerabilities
   - Review CVSS score (vulnerabilities with score 7 or higher need immediate attention)

2. **Check Patch Availability:**

```powershell
# Search for available updates
Get-WindowsUpdate -AcceptAll -Download -Install
```

3. **Update Affected Software:**

```powershell
# Example: Update specific software via Chocolatey
choco upgrade <package-name> -y
```

4. **Document Actions Taken:**
   - Log fixed vulnerabilities
   - Document accepted vulnerabilities (with justification)
   - Track remediation plans for pending vulnerabilities

### Create Vulnerability Report

You can use Kibana Canvas or Reporting to create periodic reports including:
- Total vulnerabilities by severity
- MITRE ATT&CK technique breakdown
- Affected systems and software
- Remediation status and timeline

### Troubleshooting

#### Issue: No Security Vulnerabilities Detected

**Solutions:**

1. **Check Database Update:**

```bash
# On Wazuh Manager
ls -lh /var/ossec/queue/vulnerabilities/cve.db
# File should exist and be several megabytes in size
```

2. **Check syscollector Information Collection:**

```bash
# On Wazuh Manager
/var/ossec/bin/wazuh-db <agent-id> sql "select * from sys_programs limit 10"
```

3. **Check Manager Logs:**

```bash
grep -i "vulnerability" /var/ossec/logs/ossec.log | tail -20
```

### Affected Endpoint

- Windows

---

## Slack Integration

Wazuh can automatically send alerts to Slack channels, allowing the security team to receive immediate notifications about important security events, including MITRE ATT&CK technique information.

### Prerequisites

#### 1. Create Slack Workspace (If Not Already Existing)

- Go to https://slack.com/create
- Create a new workspace or use an existing one

#### 2. Create Incoming Webhook

1. Go to: https://api.slack.com/apps
2. Click "Create New App"
3. Choose "From scratch"
4. Enter app name (e.g., "Wazuh Security Alerts")
5. Choose Workspace
6. From the sidebar, select "Incoming Webhooks"
7. Enable "Activate Incoming Webhooks"
8. Click "Add New Webhook to Workspace"
9. Choose the channel you want to send alerts to
10. Copy Webhook URL

Example Webhook URL format:
```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX
```

### Configuration on Wazuh Manager

#### 1. Add Slack Integration Configuration

Edit `/var/ossec/etc/ossec.conf` on the Wazuh Manager:

```xml
<ossec_config>
    <integration>
        <name>slack</name>
        <hook_url>https://hooks.slack.com/services/YOUR/WEBHOOK/URL</hook_url>
        <level>10</level>
        <alert_format>json</alert_format>
    </integration>
</ossec_config>
```

#### Parameter Explanation:

- `name`: Integration name (slack)
- `hook_url`: Webhook URL obtained from Slack
- `level`: Alert level to be sent (10 = High, 12 = Critical only)
- `alert_format`: Alert format (json)

#### 2. Advanced Configuration - Customize Alerts

You can customize the alerts sent:

```xml
<ossec_config>
    <!-- Send all alerts from level 10 and above -->
    <integration>
        <name>slack</name>
        <hook_url>https://hooks.slack.com/services/YOUR/WEBHOOK/URL</hook_url>
        <level>10</level>
        <alert_format>json</alert_format>
    </integration>

    <!-- Send specific alerts based on rules -->
    <integration>
        <name>slack</name>
        <hook_url>https://hooks.slack.com/services/YOUR/CRITICAL/WEBHOOK/URL</hook_url>
        <rule_id>100092,100093,87105</rule_id>
        <alert_format>json</alert_format>
    </integration>

    <!-- Send specific alerts based on group -->
    <integration>
        <name>slack</name>
        <hook_url>https://hooks.slack.com/services/YOUR/MALWARE/WEBHOOK/URL</hook_url>
        <group>virustotal,vulnerability-detector</group>
        <alert_format>json</alert_format>
    </integration>
</ossec_config>
```

#### 3. Restart Wazuh Manager

```bash
systemctl restart wazuh-manager
```

#### 4. Verify Integration Activation

```bash
# Check Wazuh log
grep -i "slack\|integrator" /var/ossec/logs/ossec.log | tail -20
```

### Steps to Generate and Test Alerts

#### 1. Create Simple Test Alert

On the monitored Windows machine, create a file in a monitored location:

```powershell
New-Item -Path "C:\Users\Administrator\Desktop\slack_test.txt" -ItemType File -Value "Testing Slack Integration - $(Get-Date)"
```

#### 2. Create High-Priority Alert - VirusTotal

```powershell
# Download EICAR file
Invoke-WebRequest -Uri "http://www.eicar.org/download/eicar.com" -OutFile "C:\Users\Administrator\Desktop\eicar_slack_test.com"
```

#### 3. Create Brute Force Attack Alert

From another machine, make failed RDP login attempts:

```bash
hydra -l Administrator -p wrong_password <windows-ip> rdp
```

### View Alerts in Slack

Alerts in Slack will include:
- Alert severity and level
- Rule description
- Agent name
- Timestamp
- **MITRE ATT&CK Technique** (if applicable)
- Source IP (for network attacks)
- File path (for FIM and malware alerts)

### Customize Alert Format with MITRE ATT&CK Information

#### Create Custom Script

On the Wazuh Manager, create `/var/ossec/integrations/custom-slack`:

```python
#!/usr/bin/env python3

import json
import sys
import requests
from datetime import datetime

# Read JSON from stdin
alert_json = json.loads(sys.stdin.read())

# Extract information
alert_level = alert_json.get('rule', {}).get('level', 0)
rule_description = alert_json.get('rule', {}).get('description', 'N/A')
agent_name = alert_json.get('agent', {}).get('name', 'N/A')
timestamp = alert_json.get('timestamp', 'N/A')

# Extract MITRE ATT&CK information
mitre_info = alert_json.get('rule', {}).get('mitre', {})
mitre_id = mitre_info.get('id', ['N/A'])[0] if isinstance(mitre_info.get('id'), list) else mitre_info.get('id', 'N/A')
mitre_tactic = mitre_info.get('tactic', ['N/A'])[0] if isinstance(mitre_info.get('tactic'), list) else mitre_info.get('tactic', 'N/A')
mitre_technique = mitre_info.get('technique', ['N/A'])[0] if isinstance(mitre_info.get('technique'), list) else mitre_info.get('technique', 'N/A')

# Determine color based on severity level
if alert_level >= 12:
    color = "#FF0000"  # Red - Critical
    severity_emoji = "🚨"
elif alert_level >= 10:
    color = "#FFA500"  # Orange - High
    severity_emoji = "⚠️"
elif alert_level >= 7:
    color = "#FFFF00"  # Yellow - Medium
    severity_emoji = "⚡"
else:
    color = "#00FF00"  # Green - Low
    severity_emoji = "ℹ️"

# Build fields list
fields = [
    {
        "title": "Agent",
        "value": agent_name,
        "short": True
    },
    {
        "title": "Alert Level",
        "value": str(alert_level),
        "short": True
    }
]

# Add MITRE ATT&CK information if available
if mitre_id != 'N/A':
    fields.append({
        "title": "MITRE ATT&CK Technique",
        "value": f"{mitre_id} - {mitre_technique}",
        "short": False
    })
    fields.append({
        "title": "MITRE ATT&CK Tactic",
        "value": mitre_tactic,
        "short": True
    })

# Format message
slack_msg = {
    "attachments": [
        {
            "color": color,
            "title": f"{severity_emoji} Wazuh Security Alert - Level {alert_level}",
            "text": rule_description,
            "fields": fields,
            "footer": "Wazuh SIEM | MITRE ATT&CK Framework",
            "footer_icon": "https://wazuh.com/favicon.ico",
            "ts": int(datetime.now().timestamp())
        }
    ]
}

# Send to Slack
webhook_url = "YOUR_SLACK_WEBHOOK_URL"
response = requests.post(webhook_url, json=slack_msg)

sys.exit(0)
```

#### Grant Required Permissions

```bash
chmod 750 /var/ossec/integrations/custom-slack
chown root:ossec /var/ossec/integrations/custom-slack
```

### Set Up Multiple Channels

You can send different types of alerts to different Slack channels:

#### 1. Channel for Critical Alerts Only

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/CRITICAL/WEBHOOK</hook_url>
    <level>12</level>
    <alert_format>json</alert_format>
</integration>
```

#### 2. Channel for Malware

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/MALWARE/WEBHOOK</hook_url>
    <group>virustotal,yara,rootcheck</group>
    <alert_format>json</alert_format>
</integration>
```

#### 3. Channel for Vulnerabilities

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/VULNERABILITIES/WEBHOOK</hook_url>
    <group>vulnerability-detector</group>
    <alert_format>json</alert_format>
</integration>
```

#### 4. Channel for Network Attacks

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/NETWORK/WEBHOOK</hook_url>
    <group>authentication_failed,attack,</group>
    <level>10</level>
    <alert_format>json</alert_format>
</integration>
```

### Create Interactive Actions in Slack

You can add buttons for quick actions:

#### Example: Button to View MITRE ATT&CK Details

```python
slack_msg = {
    "attachments": [
        {
            "color": color,
            "title": f"{severity_emoji} Brute Force Attack Detected",
            "text": f"Multiple failed login attempts from {source_ip}",
            "fields": [
                {
                    "title": "MITRE ATT&CK",
                    "value": f"T1110 - Brute Force",
                    "short": True
                }
            ],
            "actions": [
                {
                    "type": "button",
                    "text": "View MITRE Details",
                    "url": "https://attack.mitre.org/techniques/T1110/"
                },
                {
                    "type": "button",
                    "text": "View in Kibana",
                    "url": f"https://your-kibana/app/wazuh#/manager/?tab=ruleset&redirectRule={rule_id}"
                }
            ]
        }
    ]
}
```

### Monitor Integration Health

#### 1. Check integratord Status

```bash
# Verify integratord process is running
ps aux | grep integrator
```

#### 2. Monitor Logs

```bash
# Monitor integrator log in real-time
tail -f /var/ossec/logs/integrations.log
```

#### 3. Sending Statistics

```bash
# Count number of alerts sent
grep -c "slack" /var/ossec/logs/integrations.log
```

### Troubleshooting

#### Issue: Alerts Not Being Sent to Slack

**Solutions:**

1. **Verify Webhook URL Validity:**

```bash
# Test Webhook manually
curl -X POST -H 'Content-type: application/json' \
--data '{"text":"Test message from Wazuh"}' \
YOUR_SLACK_WEBHOOK_URL
```

2. **Check Alert Levels:**

```bash
# Ensure generated alerts exceed specified level
grep "level" /var/ossec/logs/alerts/alerts.json | tail -20
```

3. **Check integrator Errors:**

```bash
grep -i "error\|fail" /var/ossec/logs/integrations.log | tail -20
```

4. **Verify ossec.conf Configuration:**

```bash
# Verify XML format validity
/var/ossec/bin/verify-agent-conf
```

#### Issue: Duplicate or Too Many Alerts

**Solutions:**

1. **Increase Alert Level:**

```xml
<!-- Instead of level 7, use level 10 or higher -->
<integration>
    <name>slack</name>
    <hook_url>YOUR_WEBHOOK_URL</hook_url>
    <level>12</level>  <!-- Critical only -->
    <alert_format>json</alert_format>
</integration>
```

2. **Use Specific Rules Only:**

```xml
<integration>
    <name>slack</name>
    <hook_url>YOUR_WEBHOOK_URL</hook_url>
    <rule_id>100092,100093,87105,60122,60137</rule_id>
    <alert_format>json</alert_format>
</integration>
```

3. **Exclude Certain Rules:**

In `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100500" level="0">
    <if_sid>NOISY_RULE_ID</if_sid>
    <description>Excluding noisy rule from Slack</description>
</rule>
```

### Best Practices

#### 1. Organize Channels

- **#security-critical**: For critical alerts requiring immediate response
- **#security-malware**: For malware and VirusTotal alerts
- **#security-vulnerabilities**: For vulnerability detection alerts
- **#security-network**: For network attacks and brute force attempts
- **#security-reports**: For daily/weekly summary reports

#### 2. Set Up Appropriate Alerts

```xml
<!-- Example of balanced configuration -->
<ossec_config>
    <!-- Critical: For immediate alerts -->
    <integration>
        <name>slack</name>
        <hook_url>CRITICAL_WEBHOOK</hook_url>
        <level>12</level>
    </integration>
    
    <!-- High: For review within hours -->
    <integration>
        <name>slack</name>
        <hook_url>HIGH_WEBHOOK</hook_url>
        <level>10</level>
    </integration>
    
    <!-- Malware: Specialized -->
    <integration>
        <name>slack</name>
        <hook_url>MALWARE_WEBHOOK</hook_url>
        <group>virustotal,rootcheck</group>
    </integration>
</ossec_config>
```

#### 3. Create Naming Convention

- Use clear prefixes: `[CRITICAL]`, `[HIGH]`, `[MALWARE]`, `[MITRE: T1110]`
- Add emojis for quick distinction
- Include affected machine name
- Include MITRE ATT&CK technique ID

#### 4. Set Up Workflows in Slack

You can create automatic workflows in Slack:
- Create ticket in task management system
- Send notification to specific person
- Log event in spreadsheet
- Trigger incident response playbook based on MITRE technique

#### 5. Periodic Review

```bash
# Create weekly report of alert count
cat /var/ossec/logs/integrations.log | \
grep "slack" | \
awk '{print $1}' | \
sort | uniq -c | \
tail -7
```

### Create Daily Summary Report with MITRE ATT&CK Statistics

You can create a script to send a daily summary report:

```bash
#!/bin/bash
# /usr/local/bin/wazuh-daily-slack-report.sh

WEBHOOK_URL="YOUR_SLACK_WEBHOOK_URL"
DATE=$(date +%Y-%m-%d)

# Collect statistics
CRITICAL_COUNT=$(grep -c '"level":12' /var/ossec/logs/alerts/alerts.json)
HIGH_COUNT=$(grep -c '"level":10' /var/ossec/logs/alerts/alerts.json)
MALWARE_COUNT=$(grep -c 'virustotal' /var/ossec/logs/alerts/alerts.json)

# Count MITRE ATT&CK techniques
T1110_COUNT=$(grep -c 'T1110' /var/ossec/logs/alerts/alerts.json)  # Brute Force
T1204_COUNT=$(grep -c 'T1204' /var/ossec/logs/alerts/alerts.json)  # Malicious File
T1068_COUNT=$(grep -c 'T1068' /var/ossec/logs/alerts/alerts.json)  # Privilege Escalation

# Create message
MESSAGE='{
  "text": "📊 *Wazuh Daily Security Report - '"$DATE"'*",
  "attachments": [
    {
      "color": "#36a64f",
      "title": "Alert Statistics",
      "fields": [
        {
          "title": "Critical Alerts",
          "value": "'"$CRITICAL_COUNT"'",
          "short": true
        },
        {
          "title": "High Alerts",
          "value": "'"$HIGH_COUNT"'",
          "short": true
        },
        {
          "title": "Malware Detected",
          "value": "'"$MALWARE_COUNT"'",
          "short": true
        }
      ]
    },
    {
      "color": "#3AA3E3",
      "title": "MITRE ATT&CK Techniques Detected",
      "fields": [
        {
          "title": "T1110 - Brute Force",
          "value": "'"$T1110_COUNT"'",
          "short": true
        },
        {
          "title": "T1204 - Malicious File",
          "value": "'"$T1204_COUNT"'",
          "short": true
        },
        {
          "title": "T1068 - Privilege Escalation",
          "value": "'"$T1068_COUNT"'",
          "short": true
        }
      ]
    }
  ]
}'

# Send report
curl -X POST -H 'Content-type: application/json' --data "$MESSAGE" $WEBHOOK_URL
```

Add to crontab for daily execution:

```bash
# Run report daily at 9 AM
0 9 * * * /usr/local/bin/wazuh-daily-slack-report.sh
```

### Alerts

Confirmation of sent alerts can be found:

```bash
# In Wazuh logs
grep "slack" /var/ossec/logs/integrations.log

# In Slack
# Check different channels
```

### Affected Endpoint

- Wazuh Manager (for sending alerts)
- All Windows endpoints (as source of alerts)

---

## Conclusion

This comprehensive guide covered the most important use cases for Wazuh on Windows endpoints with integrated MITRE ATT&CK framework mapping:

### ✅ Core Security Capabilities

1. **MITRE ATT&CK Mapping & Threat Simulation**
   - Simulated real adversary behaviors (SSH brute force, privilege escalation, account creation)
   - Mapped security events to MITRE ATT&CK techniques
   - Created custom dashboards for threat intelligence
   - **SOC Analyst Skill**: Connecting logs → attacker behavior → defensive actions

2. **Brute Force Attack Detection on RDP** (T1110)
   - Protection from unauthorized access attempts
   - Real-time detection of credential stuffing attacks
   - Integration with MITRE ATT&CK Credential Access tactic

3. **File Integrity Monitoring (FIM)** (T1565, T1070.004)
   - Track changes to important files and directories
   - Detect data manipulation and defense evasion techniques
   - Real-time alerting with file hash tracking

4. **Malware Detection and Removal (VirusTotal)** (T1204)
   - Proactive protection from malicious files
   - Automatic threat removal with active response
   - Integration with VirusTotal for threat intelligence

5. **Vulnerability Detection** (T1203, T1068, T1210)
   - Manage security updates and patches
   - CVE tracking and CVSS scoring
   - Map vulnerabilities to potential exploitation techniques

6. **Slack Integration**
   - Immediate notifications for important security events
   - MITRE ATT&CK technique information in alerts
   - Multi-channel organization for different alert types

### 📊 MITRE ATT&CK Coverage Summary

| Tactic | Techniques Covered | Use Cases |
|--------|-------------------|-----------|
| Initial Access | T1190 | Vulnerability Detection |
| Execution | T1203, T1204 | Malware Detection, Vulnerability Exploitation |
| Persistence | T1136 | User Account Creation |
| Privilege Escalation | T1068 | Privilege Escalation, Vulnerability Detection |
| Defense Evasion | T1070.004, T1036, T1027 | File Integrity Monitoring, Malware Detection |
| Credential Access | T1110 | Brute Force Attack Detection |
| Lateral Movement | T1210 | Vulnerability Detection |
| Command and Control | T1105 | Malware Detection |
| Impact | T1565 | File Integrity Monitoring |

### 🎯 Recommended Next Steps

#### 1. Create Custom Security Policies

- **Define appropriate rules for your environment**
  - Create rules specific to your organization's threat landscape
  - Customize alert thresholds based on normal behavior baselines
  - Map custom rules to MITRE ATT&CK framework

- **Adjust alert levels**
  - Fine-tune severity levels to reduce false positives
  - Prioritize alerts based on business impact
  - Create tiered response procedures

- **Create custom rules for special cases**
  - Industry-specific compliance requirements
  - Critical asset monitoring
  - VIP user activity tracking

#### 2. Automate Responses

- **Develop additional active responses**
  - Automated IP blocking for brute force attacks (T1110)
  - Automated malware quarantine (T1204)
  - Account lockout for suspicious privilege escalation (T1068)
  - Network isolation for compromised systems

- **Integrate with automation tools**
  - **Ansible**: Automated remediation playbooks
  - **PowerShell**: Windows-specific response scripts
  - **Python**: Custom integration scripts
  - **SOAR platforms**: Security orchestration workflows

- **Create playbooks for common incidents**
  - Brute force attack response (T1110)
  - Malware infection response (T1204)
  - Privilege escalation response (T1068)
  - Data exfiltration prevention (T1565)

**Example Ansible Playbook for Brute Force Response:**

```yaml
---
- name: Respond to Brute Force Attack (T1110)
  hosts: firewall
  tasks:
    - name: Block attacker IP
      iptables:
        chain: INPUT
        source: "{{ attacker_ip }}"
        jump: DROP
        comment: "Blocked by Wazuh - T1110 Detection"
    
    - name: Send notification
      slack:
        token: "{{ slack_token }}"
        msg: "🚨 IP {{ attacker_ip }} blocked - MITRE T1110 Brute Force detected"
        channel: "#security-critical"
```

#### 3. Documentation and Compliance

- **Document all security procedures**
  - Incident response procedures for each MITRE technique
  - Escalation paths and contact information
  - System architecture and data flow diagrams
  - Configuration change management process

- **Create periodic reports for management**
  - Monthly security posture summary
  - MITRE ATT&CK coverage heatmap
  - Trend analysis of detected techniques
  - KPI dashboard (MTTD, MTTR, detection rate)

- **Ensure compliance with standards**
  - **PCI-DSS**: File integrity monitoring, access controls
  - **HIPAA**: Audit logging, encryption verification
  - **NIST CSF**: Detection and response capabilities
  - **ISO 27001**: Information security management
  - **GDPR**: Data breach detection and reporting

**Sample Compliance Mapping:**

| Compliance Requirement | Wazuh Capability | MITRE ATT&CK |
|------------------------|------------------|--------------|
| PCI-DSS 10.2.4, 10.2.5 | File Integrity Monitoring | T1565 |
| PCI-DSS 8.2.4 | Brute Force Detection | T1110 |
| HIPAA §164.312(b) | Audit Logging | All Techniques |
| NIST CSF DE.CM-7 | Malware Detection | T1204 |

#### 4. Training and Continuous Improvement

- **Train team on using Wazuh**
  - SOC analyst onboarding program
  - MITRE ATT&CK framework training
  - Hands-on alert triage exercises
  - Incident response simulations

- **Review and update rules regularly**
  - Quarterly rule effectiveness review
  - False positive analysis and tuning
  - New threat intelligence integration
  - MITRE ATT&CK framework updates

- **Stay informed about latest threats**
  - Subscribe to threat intelligence feeds
  - Attend security conferences and webinars
  - Participate in information sharing communities (ISACs)
  - Review MITRE ATT&CK updates and new techniques

- **Conduct purple team exercises**
  - Simulate attacks using MITRE ATT&CK techniques
  - Test detection and response capabilities
  - Document gaps and improvement areas
  - Measure detection coverage improvement over time

**Purple Team Exercise Example:**

```bash
# Exercise: Test T1110 Detection (Brute Force)
# Red Team Action:
hydra -l admin -P wordlist.txt <target-ip> rdp

# Blue Team Validation:
# 1. Verify alert generation in Wazuh
# 2. Confirm MITRE T1110 mapping
# 3. Test active response effectiveness
# 4. Document detection time (MTTD)
# 5. Validate incident response procedure
```

### 🔐 Security Architecture Overview

The integrated Wazuh deployment provides comprehensive security monitoring with MITRE ATT&CK mapping:

```
┌─────────────────────────────────────────────────────────────┐
│                     Wazuh Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐         ┌──────────────┐                │
│  │   Windows    │────────▶│    Wazuh     │                │
│  │   Agents     │  Logs   │   Manager    │                │
│  │              │◀────────│              │                │
│  └──────────────┘ Response└──────────────┘                │
│         │                        │                          │
│         │                        ├────▶ Elasticsearch       │
│         │                        │                          │
│         ▼                        ├────▶ VirusTotal API      │
│  ┌──────────────┐               │                          │
│  │File Integrity│               ├────▶ Vulnerability DB    │
│  │  Monitoring  │               │                          │
│  │   (T1565)    │               └────▶ MITRE ATT&CK        │
│  └──────────────┘                             │             │
│         │                                     │             │
│         ▼                                     ▼             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              Kibana Dashboard                        │ │
│  │  • Security Events                                   │ │
│  │  • MITRE ATT&CK Mapping                             │ │
│  │  • Custom Visualizations                            │ │
│  │  • Threat Intelligence                              │ │
│  └──────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           ▼                                 │
│                  ┌──────────────┐                          │
│                  │    Slack     │                          │
│                  │ Integration  │                          │
│                  │   (Alerts)   │                          │
│                  └──────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

### 📈 Key Performance Indicators (KPIs)

Track these metrics to measure security program effectiveness:

#### Detection Metrics
- **Mean Time to Detect (MTTD)**: Average time to detect security incidents
  - Target: < 15 minutes for critical alerts
  - Measure per MITRE technique

- **Detection Coverage**: Percentage of MITRE ATT&CK techniques covered
  - Current Coverage: 8 techniques across 8 tactics
  - Target: Expand to 20+ techniques

- **False Positive Rate**: Percentage of alerts that are false positives
  - Target: < 5% for critical alerts
  - Monitor and tune regularly

#### Response Metrics
- **Mean Time to Respond (MTTR)**: Average time to respond to incidents
  - Target: < 1 hour for critical alerts
  - Automated responses should be < 5 minutes

- **Automated Response Rate**: Percentage of incidents handled automatically
  - Current: Malware removal (T1204)
  - Target: Expand to brute force blocking (T1110)

- **Incident Resolution Rate**: Percentage of incidents fully resolved
  - Target: 100% for detected incidents
  - Track by MITRE technique

#### Coverage Metrics
- **Asset Coverage**: Percentage of endpoints with Wazuh agents
  - Target: 100% of critical assets
  - Monitor agent connectivity status

- **Rule Coverage**: Number of active detection rules
  - Current: Base rules + custom rules
  - Target: Continuous expansion based on threat landscape

### 🛡️ Additional Resources

#### Official Documentation
- **Wazuh Documentation**: https://documentation.wazuh.com
  - Installation guides
  - Configuration references
  - API documentation
  - Integration guides

- **MITRE ATT&CK**: https://attack.mitre.org
  - Technique descriptions
  - Detection strategies
  - Mitigation recommendations
  - Real-world examples

#### Community Resources
- **Wazuh Forum**: https://groups.google.com/forum/#!forum/wazuh
  - Community support
  - Configuration examples
  - Best practices discussions

- **GitHub**: https://github.com/wazuh/wazuh
  - Source code
  - Issue tracking
  - Custom integrations
  - Community contributions

- **Wazuh Slack Community**: https://wazuh.com/community/join-us-on-slack/
  - Real-time community chat
  - Expert assistance
  - Integration tips

#### Training and Certification
- **Wazuh Training**: Official training courses and materials
- **MITRE ATT&CK Training**: Free training and certification
- **SOC Analyst Certifications**: 
  - GCIA (GIAC Certified Intrusion Analyst)
  - GCIH (GIAC Certified Incident Handler)
  - Security Blue Team certifications

### ⚠️ Important Final Notes

#### Security Warnings
- **Don't share API Keys or Webhook URLs publicly**
  - Store credentials securely
  - Use environment variables
  - Implement key rotation policies
  - Monitor for unauthorized access

- **Use HTTPS for all communications**
  - Wazuh Manager to Agents
  - Wazuh to Elasticsearch
  - API integrations
  - Web dashboard access

- **Change default passwords**
  - Wazuh admin credentials
  - Elasticsearch passwords
  - Kibana access
  - API authentication tokens

- **Review permissions regularly**
  - User access levels
  - API key permissions
  - File system permissions
  - Network access controls

#### 🔒 Best Practices

##### Configuration Management
- **Test all configurations in a test environment first**
  - Separate development/staging/production environments
  - Validate rule changes before deployment
  - Test active responses thoroughly
  - Document all changes

- **Keep backups of configuration files**
  - Daily automated backups
  - Version control (Git) for configurations
  - Backup retention policy (30 days minimum)
  - Test restore procedures regularly

- **Monitor resource consumption (CPU, Memory, Disk)**
  - Set up resource monitoring alerts
  - Plan capacity for growth
  - Optimize database indexes
  - Archive old logs regularly

- **Update Wazuh and components regularly**
  - Subscribe to security advisories
  - Test updates in staging environment
  - Schedule maintenance windows
  - Maintain upgrade documentation

##### Operational Excellence
- **Document all configuration changes**
  - Change request process
  - Configuration version control
  - Rollback procedures
  - Change approval workflow

- **Keep a log of incidents and actions taken**
  - Incident tracking system
  - Lessons learned documentation
  - Root cause analysis
  - Corrective action tracking

- **Create a runbook for your team**
  - Alert triage procedures
  - Escalation paths
  - Common troubleshooting steps
  - Contact information

- **Conduct regular security assessments**
  - Quarterly rule effectiveness reviews
  - Annual penetration testing
  - Continuous vulnerability scanning
  - Gap analysis against MITRE ATT&CK

### 📊 MITRE ATT&CK Heat Map

Track your organization's detection coverage across MITRE ATT&CK tactics:

```
Detection Coverage by Tactic:
═══════════════════════════════════════════════════

Initial Access        ████░░░░░░ 40% (T1190)
Execution            ████████░░ 80% (T1203, T1204)
Persistence          ████░░░░░░ 40% (T1136)
Privilege Escalation ████░░░░░░ 40% (T1068)
Defense Evasion      ████████░░ 80% (T1070, T1036, T1027)
Credential Access    ██████████ 100% (T1110)
Discovery            ░░░░░░░░░░ 0%
Lateral Movement     ██░░░░░░░░ 20% (T1210)
Collection           ░░░░░░░░░░ 0%
Command & Control    ██░░░░░░░░ 20% (T1105)
Exfiltration         ░░░░░░░░░░ 0%
Impact               ████░░░░░░ 40% (T1565)

Overall Coverage: 42% (8 of 19 techniques)
```

**Improvement Recommendations:**
1. Expand coverage in Discovery tactic (process monitoring, network scanning)
2. Implement Collection tactic detection (clipboard data, screen capture)
3. Enhance Exfiltration detection (data transfer monitoring)
4. Add more Lateral Movement techniques (RDP monitoring, SMB abuse)

### 🎓 Learning Path for SOC Analysts

#### Beginner Level (Weeks 1-4)
- [ ] Understand Wazuh architecture and components
- [ ] Learn MITRE ATT&CK framework basics
- [ ] Practice alert triage in Kibana dashboard
- [ ] Complete basic rule customization
- [ ] Document your first 10 incident investigations

#### Intermediate Level (Weeks 5-12)
- [ ] Create custom detection rules
- [ ] Map detections to MITRE ATT&CK techniques
- [ ] Build custom Kibana visualizations
- [ ] Implement active response scripts
- [ ] Conduct threat hunting exercises
- [ ] Participate in purple team exercises

#### Advanced Level (Weeks 13-24)
- [ ] Develop complex correlation rules
- [ ] Integrate with SOAR platforms
- [ ] Design security architecture improvements
- [ ] Mentor junior analysts
- [ ] Contribute to threat intelligence feeds
- [ ] Present findings to management

### 📝 Sample Incident Report Template

```markdown
# Security Incident Report

## Incident Overview
- **Incident ID**: INC-2025-001
- **Date/Time**: 2025-10-11 14:30:00 UTC
- **Severity**: High
- **Status**: Resolved

## MITRE ATT&CK Mapping
- **Tactic**: Credential Access
- **Technique**: T1110 - Brute Force
- **Sub-Technique**: T1110.001 - Password Guessing

## Detection Details
- **Alert Rule ID**: 60122
- **Alert Description**: Multiple RDP authentication failures
- **Source IP**: 192.168.1.100
- **Target System**: WINSERVER01
- **Target Account**: Administrator

## Timeline
- 14:30:00 - First failed login attempt detected
- 14:30:15 - 20 failed attempts within 15 seconds
- 14:30:20 - Wazuh alert triggered (Level 10)
- 14:30:25 - Slack notification sent
- 14:31:00 - SOC analyst began investigation
- 14:35:00 - Source IP blocked via firewall
- 14:40:00 - Incident contained

## Response Actions Taken
1. Blocked attacker IP address (192.168.1.100)
2. Reviewed authentication logs for successful logins
3. Verified no account compromise occurred
4. Added IP to threat intelligence blocklist
5. Notified security team via Slack

## Root Cause
External brute force attack from known malicious IP range

## Lessons Learned
- Detection worked as expected (MTTD: < 1 minute)
- Response time: 5 minutes (within SLA)
- Recommendation: Implement geo-blocking for RDP access

## Follow-up Actions
- [ ] Review RDP access policies
- [ ] Implement MFA for administrative accounts
- [ ] Update firewall rules for RDP access
- [ ] Add attacker IP range to permanent blocklist
```

### 🚀 Deployment Checklist

Before going to production, ensure:

#### Infrastructure
- [ ] Wazuh Manager installed and configured
- [ ] Elasticsearch cluster deployed (3+ nodes for production)
- [ ] Kibana dashboard accessible via HTTPS
- [ ] Filebeat configured for log forwarding
- [ ] Backup and recovery procedures tested

#### Agent Deployment
- [ ] Wazuh agents deployed on all Windows endpoints
- [ ] Agent connectivity verified (all agents connected)
- [ ] Agent configuration synchronized
- [ ] Agent auto-enrollment configured
- [ ] Agent upgrade procedures documented

#### Detection Rules
- [ ] Base ruleset reviewed and customized
- [ ] Custom rules created for your environment
- [ ] MITRE ATT&CK mappings verified
- [ ] Alert thresholds tuned
- [ ] False positive rate < 5%

#### Integrations
- [ ] VirusTotal API integrated and tested
- [ ] Slack webhooks configured
- [ ] Vulnerability database updated
- [ ] Active response scripts tested
- [ ] Email notifications configured

#### Documentation
- [ ] Installation documentation complete
- [ ] Configuration management process defined
- [ ] Incident response playbooks created
- [ ] Escalation procedures documented
- [ ] Training materials prepared

#### Testing
- [ ] Purple team exercises conducted
- [ ] All MITRE techniques tested
- [ ] Active responses validated
- [ ] Backup/restore procedures verified
- [ ] Disaster recovery plan tested

#### Compliance
- [ ] Compliance requirements mapped
- [ ] Audit logging enabled
- [ ] Retention policies configured
- [ ] Access controls implemented
- [ ] Regular compliance reports scheduled

### 🎯 Success Metrics

After implementing this comprehensive Wazuh deployment, you should achieve:

#### Immediate Results (Week 1)
- ✅ Real-time visibility into security events
- ✅ Automated malware detection and removal
- ✅ Brute force attack detection
- ✅ File integrity monitoring operational
- ✅ Slack notifications functioning

#### Short-term Results (Month 1)
- ✅ < 15 minute MTTD for critical alerts
- ✅ < 5% false positive rate
- ✅ 100% agent deployment on critical assets
- ✅ 8+ MITRE ATT&CK techniques detected
- ✅ Complete incident documentation

#### Long-term Results (Quarter 1)
- ✅ 20+ MITRE ATT&CK techniques covered
- ✅ < 1 hour MTTR for critical incidents
- ✅ 50%+ automated response rate
- ✅ Compliance audit ready
- ✅ Mature SOC operations

### 🏆 Conclusion

This comprehensive guide has provided you with a production-ready Wazuh deployment integrated with MITRE ATT&CK framework for Windows environments. You now have:

1. **Complete Detection Coverage** across multiple attack vectors
2. **Automated Response Capabilities** for common threats
3. **Threat Intelligence Integration** with VirusTotal and MITRE ATT&CK
4. **Real-time Alerting** via Slack with technique mapping
5. **Vulnerability Management** for proactive security
6. **Compliance Readiness** for major security standards

**Remember**: Cybersecurity is an ongoing process that requires continuous monitoring, updating, and improvement. Stay vigilant, keep learning, and regularly test your defenses.

### 📞 Getting Help

If you need assistance:

1. **Community Support**: Wazuh Forum and Slack
2. **Professional Services**: Wazuh commercial support
3. **Security Community**: Share and learn from peers
4. **Documentation**: Always refer to official docs for latest information

---

**This guide was prepared to help you deploy and operate Wazuh efficiently in a Windows environment with comprehensive MITRE ATT&CK integration. The combination of Wazuh's powerful detection capabilities and MITRE ATT&CK's threat intelligence framework provides a robust foundation for modern security operations.**

**We wish you a safe and successful experience with Wazuh! 🛡️**

---

## Appendix: Quick Reference

### Common Wazuh Commands

```bash
# Wazuh Manager
systemctl status wazuh-manager
systemctl restart wazuh-manager
/var/ossec/bin/wazuh-control info

# Check agent status
/var/ossec/bin/agent_control -l
/var/ossec/bin/agent_control -i <agent-id>

# Test rules
/var/ossec/bin/wazuh-logtest

# Database queries
/var/ossec/bin/wazuh-db <agent-id> sql "SELECT * FROM sys_programs"
```

### Common PowerShell Commands

```powershell
# Wazuh Agent
Get-Service WazuhSvc
Restart-Service -Name wazuh
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20

# File Integrity Testing
New-Item -Path "C:\Users\Admin\Desktop\test.txt" -ItemType File
Add-Content -Path "C:\Users\Admin\Desktop\test.txt" -Value "Test"
Remove-Item -Path "C:\Users\Admin\Desktop\test.txt"
```

### Kibana Search Queries

```
# MITRE ATT&CK Technique Search
data.mitre.technique:T1110*

# High Severity Alerts
rule.level:>=10

# Specific Agent
agent.name:"WINSERVER01"

# Time Range
@timestamp:[now-1h TO now]

# Combined Query
rule.level:>=10 AND data.mitre.technique:* AND agent.name:"WINSERVER01"
```

### Alert Level Reference

| Level | Severity | Description | Example |
|-------|----------|-------------|---------|
| 0-3 | Informational | Normal events | Successful login |
| 4-6 | Low | Minor issues | Failed login attempt |
| 7-9 | Medium | Suspicious activity | Multiple failed logins |
| 10-12 | High | Security incident | Brute force detected |
| 13-15 | Critical | Severe breach | Malware execution |

---

**End of Comprehensive Wazuh Guide with MITRE ATT&CK Integration**