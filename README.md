# Wazuh Proof of Concept Guide (Windows)

## Guide Version

This guide is compatible with Wazuh 4.2.0 and newer versions.
[Click here for older versions](https://github.com/wazuh/wazuh/wiki/Proof-of-concept-guide/b3681f865eb671a0b0d13a1b1012477fd0d932ba).

## Table of Contents

- [Guide Version](#guide-version)
- [Brute Force Attack Detection - RDP](#brute-force-attack-detection---rdp)
- [File Integrity Monitoring](#file-integrity-monitoring)
- [Malware Detection and Removal - VirusTotal Integration](#malware-detection-and-removal---virustotal-integration)
- [Vulnerability Detection](#vulnerability-detection)
- [Slack Integration](#slack-integration)

## Introduction

This document explains how to set up a Wazuh environment to test various product capabilities on Windows endpoints. It assumes the following components are already installed:

- Elasticsearch + Kibana + Wazuh Kibana Plugin
- Wazuh Manager + Filebeat (for Elasticsearch integration)
- Wazuh Agent (Windows)

A good guide on how to install these components can be found in (https://github.com/Rasheed1506993/wazuh_lab/blob/main/Wazuh_Server_&_Agent_Installation_Documentation.pdf)

The sections below explain the configurations required to set up different use cases on Windows endpoints.

## Brute Force Attack Detection - RDP

Brute force attacks on RDP (Remote Desktop Protocol) are common attack vectors on Windows systems. Wazuh provides out-of-the-box rules capable of identifying brute force attacks by correlating multiple authentication failure events.

#### *Configuration*

- Ensure the Remote Desktop service is enabled on the monitored Windows machine.

![Remote Desktop Service](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image1.png)

- Ensure the Wazuh agent is installed and running on the Windows machine.

```powershell
Get-Service WazuhSvc
```

- To perform an automated attack from another machine, you can use the `hydra` tool:

![Hydra Tool](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image3.png)

```bash
# From a Linux machine
hydra -l Target_user -P password.txt <windows-ip-address> rdp
```

#### *Steps to Generate Alerts*

```bash
hydra -l Me -P /home/kali/password.txt 192.168.201.1 rdp
```

![Hydra Attack](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image4.png)

#### *Alerts*

Related alerts can be found in Kibana using:

- `rule.id:(60122 OR 60137)`
- `rule.description:*brute*force*`
- `data.win.system.eventID:4625` (Login failure)

![Kibana Alerts 1](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image5.png)

![Kibana Alerts 2](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image6.png)

![Kibana Alerts 3](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image7.png)

#### *Affected Endpoint*

- Windows

## File Integrity Monitoring

File Integrity Monitoring (FIM) is a vital security feature that monitors changes to critical files and directories. Wazuh can detect file creation, modification, or deletion in real-time.

#### *Configuration*

On the monitored Windows machine, edit the Wazuh agent configuration file located at: `C:\Program Files (x86)\ossec-agent\ossec.conf`

**Screenshot: Shows the path to ossec.conf file in Windows Explorer**

![ossec.conf path](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image8.png)

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

#### *Parameter Explanation*:

- `check_all="yes"`: Monitor all changes (size, permissions, owner, etc.)
- `report_changes="yes"`: Report file content changes
- `realtime="yes"`: Real-time monitoring
- `frequency="300"`: Periodic scan every 300 seconds (5 minutes)

- After modifying the configuration file, restart the Wazuh Agent service:

![Restart Service](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image11.png)

- Verify that the service restarted successfully:

```powershell
Get-Service WazuhSvc
```

#### *Steps to Generate Alerts*

Make changes to files in the monitored directories:

**1. Create a new file:**

```powershell
New-Item -Path "C:\Users\<USERNAME>\Desktop\test_file.txt" -ItemType File -Value "This is a test file for FIM monitoring"
```

![Create File](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image13.png)

**2. Modify an existing file:**

```powershell
Add-Content -Path "C:\Users\RAOOF\Desktop\test_file.txt" -Value "Modified content - $(Get-Date)"
```

![Modify File](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image14.png)

**3. Delete a file:**

```powershell
Remove-Item -Path "C:\Users\RAOOF\Desktop\test_file.txt"
```

![Delete File](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image15.png)

**4. Create multiple files for comprehensive testing:**

```powershell
1..5 | ForEach-Object {
    New-Item -Path "C:\Users\RAOOF\Desktop\test_$_.txt" -ItemType File -Value "Test file number $_"
}
```

![Multiple Files](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image16.png)

#### *Alerts*

Related alerts can be found in Kibana using:

**Search Queries:**
- `rule.groups:"syscheck"` - All FIM alerts
- `syscheck.event:"added"` - Added files
- `syscheck.event:"modified"` - Modified files
- `syscheck.event:"deleted"` - Deleted files
- `syscheck.path:"*Desktop*"` - Desktop changes

**Screenshot: Shows Kibana Dashboard interface with FIM alerts - General view of all alerts**

![FIM Dashboard](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image17.png)

**Screenshot: Shows details of file addition alert (syscheck.event: added)**

![File Added](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image18.png)

**Screenshot: Shows file modification alert (syscheck.event: modified) with logged changes**

![File Modified 1](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image19.png)

![File Modified 2](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image20.png)

**Screenshot: Shows file deletion alert (syscheck.event: deleted)**

![File Deleted](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image21.png)

**Screenshot: Shows a Visualization in Kibana displaying the distribution of FIM events by type (added, modified, deleted)**

![FIM Visualization](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image22.png)

#### *Additional Information Available in Alerts*:

- Full file path
- Date and time of change
- Event type (added, modified, deleted)
- Permission changes
- Owner changes
- File size changes
- File hash (MD5, SHA1, SHA256)

**Screenshot: Shows complete JSON of an alert displaying all available fields including hashes**

![JSON Details](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image23.png)

#### *Affected Endpoint*

- Windows

## Malware Detection and Removal - VirusTotal Integration

Wazuh has the ability to integrate with VirusTotal API and run a query when a file change is detected. When a malicious file is detected, Wazuh can be configured to remove it automatically.

#### *Prerequisites*

- VirusTotal API Key (can be obtained for free from https://www.virustotal.com)
- Wazuh Agent installed on Windows
- Python installed on Wazuh Manager

**Screenshot: Shows VirusTotal page for creating an account and obtaining an API Key**

![VirusTotal Account](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image24.png)

**Screenshot: Shows API Key page in VirusTotal account**

![VirusTotal API Key](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image25.png)

### Configuration on Wazuh Manager

#### 1. *Enable VirusTotal Integration*

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

![VirusTotal Config](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image26.png)

#### 2. *Create Custom Rules*

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
    <!-- Rule for detected malicious files -->
    <rule id="100300" level="12">
        <if_sid>87105</if_sid>
        <description>VirusTotal: Malicious file detected - $(data.virustotal.source.file)</description>
    </rule>
</group>
```

**Screenshot: Shows local_rules.xml file content with custom rules**

![Custom Rules](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image27.png)

#### 3. *Create Decoder for Active Response*

Edit `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="ar_log_fields">
    <parent>ar_log</parent>
    <regex offset="after_parent">^(\S+) - Removed threat located at (\S+)</regex>
    <order>script_name, path</order>
</decoder>
```

**Screenshot: Shows local_decoder.xml file content**

![Local Decoder](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image28.png)

#### 4. *Create Automatic Removal Rules*

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

![Removal Rules](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image29.png)

#### 5. *Configure Active Response on Wazuh Manager*

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

**Screenshot: Shows active response configuration in ossec.conf on Wazuh Manager**

#### 6. *Restart Wazuh Manager*

```bash
systemctl restart wazuh-manager
```

**Screenshot: Shows successful restart of wazuh-manager**

### Configuration on Windows Agent

#### 1. *Configure File Integrity Monitoring*

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

**[Place screenshot: Shows syscheck configuration in ossec.conf on Windows]**

#### 2. *Create Active Response Script*

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

![Remove Threat CMD Path](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image30.png)

**Screenshot: Shows remove-threat.cmd file content in Notepad**

![Remove Threat CMD Content](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image31.png)

#### 3. *Create Enhanced PowerShell Script (Optional)*

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

![Remove Threat PS1](https://github.com/Rasheed1506993/wazuh_lab/blob/main/media/image32.png)

#### 4. *Restart Wazuh Agent*

```powershell
Restart-Service -Name wazuh
```

### Steps to Generate Alerts

#### 1. *Download EICAR Test File*

EICAR is a standard test file for antivirus software; it is harmless but detected as malware.

**Using PowerShell:**

```powershell
# Navigate to Desktop
cd C:\Users\RAOOF\Desktop

# Download EICAR file
Invoke-WebRequest -Uri "http://www.eicar.org/download/eicar.com" -OutFile "eicar.com"
```

**Screenshot: Shows executing download command in PowerShell**

**Or using browser:**
- Open browser and go to: `http://www.eicar.org/download/eicar.com`
- Save the file to Desktop

**Screenshot: Shows EICAR download page in browser**

**Screenshot: Shows eicar.com file on Desktop before automatic deletion**

#### 2. *Create EICAR File Manually (Alternative)*

```powershell
# Create EICAR file manually
$eicar = 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*'
Set-Content -Path "C:\Users\Administrator\Desktop\eicar_test.txt" -Value $eicar
```

**Screenshot: Shows creating EICAR file manually**

#### 3. *Monitor Logs*

Monitor the active response log file:

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\active-responses.log" -Wait
```

**Screenshot: Shows active-responses.log file content in PowerShell**

### Alerts

Related alerts can be found in Kibana:

**Search Queries:**

1. **General VirusTotal Alerts:**

```
rule.groups:"virustotal"
```

**[Place screenshot: Shows all VirusTotal alerts in Kibana Dashboard]**

2. **Detected Malicious Files:**

```
data.virustotal.positives:>0
```

**[Place screenshot: Shows malicious file alerts with the number of engines that detected them (positives)]**

3. **Successful Removal Alerts:**

```
rule.id:100092
```

**[Place screenshot: Shows successful removal alert for malicious file]**

4. **Failed Removal Alerts:**

```
rule.id:100093
```

5. **Search for Specific File:**

```
data.virustotal.source.file:"*eicar*"
```

**[Place screenshot: Shows VirusTotal alert details for EICAR file showing:]**
- Number of antivirus engines that detected the file (positives)
- Total number of engines that scanned the file (total)
- Full file path
- Scan date and time

**[Place screenshot: Shows complete JSON of VirusTotal alert showing all technical details]**

**[Place screenshot: Shows a Visualization in Kibana displaying alert distribution by number of engines that detected the threat]**

#### *Additional Information in Alerts*:

- **Scan ID**: Unique scan identifier in VirusTotal
- **Positives**: Number of antivirus engines that detected the file as malicious
- **Total**: Total number of engines that scanned the file
- **Permalink**: Direct link to scan results on VirusTotal website
- **File hashes**: MD5, SHA1, SHA256 of the file
- **Scan date**: Date and time of scan

**[Place screenshot: Shows expanded alert details in Kibana with all fields mentioned above]**

### Verify Successful Automatic Removal

#### 1. *Check Active Response Log*

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\active-responses.log" | Select-Object -Last 10
```

**[Place screenshot: Shows last 10 entries in active response log confirming file deletion]**

#### 2. *Verify File Does Not Exist*

```powershell
Test-Path "C:\Users\Administrator\Desktop\eicar.com"
# Should return False if deleted successfully
```

**[Place screenshot: Shows Test-Path command result confirming file no longer exists]**

#### 3. *View Desktop*

**[Place screenshot: Shows Desktop after automatic deletion - malicious file no longer present]**

### Troubleshooting

#### *Issue*: *Files Not Being Sent to VirusTotal*

**Solutions:**

1. **Verify API Key Validity:**

```bash
# On Wazuh Manager
grep -i "virustotal" /var/ossec/logs/ossec.log
```

**[Place screenshot: Shows ossec.log to check for VirusTotal API errors]**

2. **Check API Limit:**

- Free key is limited to 4 requests per minute
- Ensure not exceeding the limit

**[Place screenshot: Shows error message if API limit is exceeded]**

#### *Issue*: *Active Response Not Working*

**Solutions:**

1. **Check File Permissions:**

```powershell
Get-Acl "C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.cmd" | Format-List
```

**[Place screenshot: Shows remove-threat.cmd file permissions]**

2. **Check Agent Log:**

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" | Select-Object -Last 20
```

**[Place screenshot: Shows last 20 lines from Wazuh agent log]**

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

**[Place screenshot: Shows manual script testing]**

### Affected Endpoint

- Windows

## Vulnerability Detection

Wazuh is capable of detecting whether installed applications contain unpatched security vulnerabilities (CVEs) on the monitored system. This feature is very important for proactive system security management.

### Configuration on Wazuh Manager

#### 1. *Enable Vulnerability Detector*

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

**[Place screenshot: Shows vulnerability-detector configuration in ossec.conf on Wazuh Manager]**

#### *Parameter Explanation*:

- `enabled`: Enable detector
- `interval`: Interval between scans (5 minutes)
- `ignore_time`: Ignore old vulnerabilities (6 hours)
- `run_on_start`: Run scan on startup
- `provider name="msu"`: Microsoft Security Updates vulnerability provider
- `provider name="nvd"`: National Vulnerability Database

#### 2. *Restart Wazuh Manager*

```bash
systemctl restart wazuh-manager
```

**[Place screenshot: Shows successful restart of wazuh-manager]**

#### 3. *Verify Database Update*

```bash
# Check CVE database
ls -lh /var/ossec/queue/vulnerabilities/cve.db

# Monitor update process in log
tail -f /var/ossec/logs/ossec.log | grep -i vulnerability
```

**[Place screenshot: Shows size and date of cve.db file]**

**[Place screenshot: Shows Wazuh log during vulnerability database update]**

### Configuration on Windows Agent

#### 1. *Enable syscollector to Collect Software Information*

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

**[Place screenshot: Shows syscollector configuration in ossec.conf on Windows]**

#### *Parameter Explanation*:

- `packages`: Collect list of installed software
- `hotfixes`: Collect list of installed Windows updates
- `interval`: Scan frequency (every hour)
- `scan_on_start`: Run scan on agent startup

#### 2. *Restart Wazuh Agent*

```powershell
Restart-Service -Name wazuh
```

**[Place screenshot: Shows restarting Wazuh service on Windows]**

#### 3. *Verify syscollector is Working*

```powershell
# Display last events in Wazuh log
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" | Select-Object -Last 30 | Select-String -Pattern "syscollector"
```

**[Place screenshot: Shows agent log with syscollector messages]**

### Steps to Generate Alerts

Security vulnerabilities will be detected automatically based on:
1. Software installed on the system
2. Missing Windows updates
3. Operating system version

#### 1. *View Installed Software*

```powershell
# Display list of installed software
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | 
    Select-Object DisplayName, DisplayVersion, Publisher | 
    Format-Table -AutoSize
```

**[Place screenshot: Shows list of installed software in PowerShell]**

#### 2. *Check Installed Updates*

```powershell
# Display list of installed Hotfixes
Get-HotFix | Sort-Object -Property InstalledOn -Descending | 
    Format-Table -AutoSize
```

**[Place screenshot: Shows list of installed Windows Updates]**

#### 3. *View Operating System Information*

```powershell
# Operating system information
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer
```

**[Place screenshot: Shows Windows operating system information]**

#### 4. *Simulate System with Vulnerabilities (Optional - For Testing Only)*

To test the feature better, you can:
- Use an old Windows system (like Windows Server 2012 or Windows 7)
- Install old software known to have vulnerabilities
- Don't install the latest security updates

**Important Note:** This is for testing only in an isolated environment!

### Alerts

Related alerts can be found in Kibana:

#### *Search Queries*:

1. **All Vulnerability Alerts:**

```
rule.groups:"vulnerability-detector"
```

**[Place screenshot: Shows all vulnerability detector alerts in Kibana Dashboard]**

2. **Vulnerabilities by severity level:**
```
data.vulnerability.severity:"High"
data.vulnerability.severity:"Critical"
data.vulnerability.severity:"Medium"
```

**[Place screenshot: Shows vulnerability alerts classified by severity level (High, Critical, Medium)]**

3. **Vulnerabilities in Specific Software:**

```
data.vulnerability.package.name:"<package_name>"
```

4. **Specific CVE Vulnerabilities:**

```
data.vulnerability.cve:"CVE-2021-*"
```

**[Place screenshot: Shows search for specific CVE]**

5. **Vulnerabilities with Available Patch:**

```
data.vulnerability.status:"Available"
```

### Alert Details

Each vulnerability alert contains the following information:

**[Place screenshot: Shows complete vulnerability alert details in Kibana displaying:]**

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

**[Place screenshot: Shows complete JSON of vulnerability alert]**

### Visualizations in Kibana

You can create useful dashboards to display vulnerabilities:

#### 1. *Vulnerability Distribution by Severity*

**[Place screenshot: Shows Pie Chart displaying vulnerability distribution by severity level]**

#### 2. *Top Affected Software*

**[Place screenshot: Shows Bar Chart displaying software with most security vulnerabilities]**

#### 3. *Vulnerability Trend Over Time*

**[Place screenshot: Shows Line Chart displaying vulnerability detection over time]**

#### 4. *CVSS Scores*

**[Place screenshot: Shows distribution of CVSS scores for detected vulnerabilities]**

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

**[Place screenshot: Shows searching for updates in Windows Update]**

3. **Update Affected Software:**

```powershell
# Example: Update specific software via Chocolatey
choco upgrade <package-name> -y
```

4. **Document Actions Taken:**
   - Log fixed vulnerabilities
   - Document accepted vulnerabilities (with justification)
   - Track remediation plans for pending vulnerabilities

**[Place screenshot: Shows Excel spreadsheet or document showing vulnerability tracking and actions taken]**

### Create Vulnerability Report

You can use Kibana Canvas or Reporting to create periodic reports:

**[Place screenshot: Shows PDF report generated from Kibana including vulnerability summary]**

### Troubleshooting

#### *Issue*: *No Security Vulnerabilities Detected*

**Solutions:**

1. **Check Database Update:**

```bash
# On Wazuh Manager
ls -lh /var/ossec/queue/vulnerabilities/cve.db
# File should exist and be several megabytes in size
```

**[Place screenshot: Shows verification of cve.db file existence and size]**

2. **Check syscollector Information Collection:**

```bash
# On Wazuh Manager
/var/ossec/bin/wazuh-db <agent-id> sql "select * from sys_programs limit 10"
```

**[Place screenshot: Shows database query result for installed software]**

3. **Check Manager Logs:**

```bash
grep -i "vulnerability" /var/ossec/logs/ossec.log | tail -20
```

**[Place screenshot: Shows ossec.log with vulnerability detector messages]**

### Affected Endpoint

- Windows

## Slack Integration

Wazuh can automatically send alerts to Slack channels, allowing the security team to receive immediate notifications about important security events.

### Prerequisites

#### 1. *Create Slack Workspace (If Not Already Existing)*

- Go to https://slack.com/create
- Create a new workspace or use an existing one

**[Place screenshot: Shows Slack Workspace creation page]**

#### 2. *Create Incoming Webhook*

1. Go to: https://api.slack.com/apps
2. Click "Create New App"
3. Choose "From scratch"
4. Enter app name (e.g., "Wazuh Security Alerts")
5. Choose Workspace

**[Place screenshot: Shows page for creating new Slack app]**

6. From the sidebar, select "Incoming Webhooks"
7. Enable "Activate Incoming Webhooks"
8. Click "Add New Webhook to Workspace"
9. Choose the channel you want to send alerts to
10. Copy Webhook URL

**[Place screenshot: Shows Incoming Webhooks page in Slack API]**

**[Place screenshot: Shows selecting channel to send alerts to]**

**[Place screenshot: Shows Webhook URL (with sensitive parts hidden)]**

Example Webhook URL format:

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX
```

### Configuration on Wazuh Manager

#### 1. *Add Slack Integration Configuration*

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

**[Place screenshot: Shows adding Slack configuration in ossec.conf]**

#### *Parameter Explanation*:

- `name`: Integration name (slack)
- `hook_url`: Webhook URL obtained from Slack
- `level`: Alert level to be sent (10 = High, 12 = Critical only)
- `alert_format`: Alert format (json)

#### 2. *Advanced Configuration - Customize Alerts*

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

**[Place screenshot: Shows multiple Slack configurations for different channels]**

#### 3. *Restart Wazuh Manager*

```bash
systemctl restart wazuh-manager
```

**[Place screenshot: Shows restarting wazuh-manager]**

#### 4. *Verify Integration Activation*

```bash
# Check Wazuh log
grep -i "slack\|integrator" /var/ossec/logs/ossec.log | tail -20
```

**[Place screenshot: Shows ossec.log with messages confirming Slack integration activation]**

### Steps to Generate and Test Alerts

#### 1. *Create Simple Test Alert*

On the monitored Windows machine, create a file in a monitored location:

```powershell
New-Item -Path "C:\Users\Administrator\Desktop\slack_test.txt" -ItemType File -Value "Testing Slack Integration - $(Get-Date)"
```

**[Place screenshot: Shows creating test file in PowerShell]**

#### 2. *Create High-Priority Alert - VirusTotal*

```powershell
# Download EICAR file
Invoke-WebRequest -Uri "http://www.eicar.org/download/eicar.com" -OutFile "C:\Users\Administrator\Desktop\eicar_slack_test.com"
```

**[Place screenshot: Shows downloading EICAR file for testing]**

#### 3. *Create Brute Force Attack Alert*

From another machine, make failed RDP login attempts:

```bash
hydra -l Administrator -p wrong_password <windows-ip> rdp
```

**[Place screenshot: Shows failed RDP attempts]**

### View Alerts in Slack

#### 1. *General Alerts*

**[Place screenshot: Shows Slack channel with multiple alerts from Wazuh]**

#### 2. *File Integrity Monitoring Alert Details*

**[Place screenshot: Shows FIM alert in Slack displaying:]**
- Affected file name
- Event type (added, modified, deleted)
- Date and time
- Agent name
- Alert level

#### 3. *VirusTotal Alert Details*

**[Place screenshot: Shows VirusTotal alert in Slack displaying:]**
- Malicious file name
- Number of engines that detected it
- VirusTotal link
- Removal status

#### 4. *Brute Force Alert Details*

**[Place screenshot: Shows brute force attack alert in Slack displaying:]**
- Attack type
- Source IP address
- Number of attempts
- Targeted username

### Customize Alert Format in Slack

#### 1. *Create Custom Script*

You can create a custom Python script to format alerts better:

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

# Format message
slack_msg = {
    "attachments": [
        {
            "color": color,
            "title": f"{severity_emoji} Wazuh Security Alert - Level {alert_level}",
            "text": rule_description,
            "fields": [
                {
                    "title": "Agent",
                    "value": agent_name,
                    "short": True
                },
                {
                    "title": "Timestamp",
                    "value": timestamp,
                    "short": True
                }
            ],
            "footer": "Wazuh SIEM",
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

**[Place screenshot: Shows custom-slack file content in text editor]**

#### 2. *Grant Required Permissions*

```bash
chmod 750 /var/ossec/integrations/custom-slack
chown root:ossec /var/ossec/integrations/custom-slack
```

**[Place screenshot: Shows executing chmod and chown commands]**

### Set Up Multiple Channels

You can send different types of alerts to different Slack channels:

#### 1. *Channel for Critical Alerts Only*

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/CRITICAL/WEBHOOK</hook_url>
    <level>12</level>
    <alert_format>json</alert_format>
</integration>
```

**[Place screenshot: Shows #security-critical channel in Slack with only critical alerts]**

#### 2. *Channel for Malware*

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/MALWARE/WEBHOOK</hook_url>
    <group>virustotal,yara,rootcheck</group>
    <alert_format>json</alert_format>
</integration>
```

**[Place screenshot: Shows #malware-alerts channel in Slack]**

#### 3. *Channel for Vulnerabilities*

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/VULNERABILITIES/WEBHOOK</hook_url>
    <group>vulnerability-detector</group>
    <alert_format>json</alert_format>
</integration>
```

**[Place screenshot: Shows #vulnerabilities channel in Slack]**

#### 4. *Channel for Network Attacks*

```xml
<integration>
    <name>slack</name>
    <hook_url>https://hooks.slack.com/services/YOUR/NETWORK/WEBHOOK</hook_url>
    <group>authentication_failed,attack,</group>
    <level>10</level>
    <alert_format>json</alert_format>
</integration>
```

**[Place screenshot: Shows #network-attacks channel in Slack]**

### Create Interactive Actions in Slack

You can add buttons for quick actions:

#### *Example*: *Button to Block IP*

```python
slack_msg = {
    "attachments": [
        {
            "color": color,
            "title": f"{severity_emoji} Brute Force Attack Detected",
            "text": f"Multiple failed login attempts from {source_ip}",
            "actions": [
                {
                    "type": "button",
                    "text": "Block IP",
                    "url": f"https://your-wazuh-api/security/block?ip={source_ip}"
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

**[Place screenshot: Shows Slack alert with interactive action buttons]**

### Monitor Integration Health

#### 1. *Check integratord Status*

```bash
# Verify integratord process is running
ps aux | grep integrator
```

**[Place screenshot: Shows ossec-integratord process running]**

#### 2. *Monitor Logs*

```bash
# Monitor integrator log in real-time
tail -f /var/ossec/logs/integrations.log
```

**[Place screenshot: Shows integrations.log with successful sending messages to Slack]**

#### 3. *Sending Statistics*

```bash
# Count number of alerts sent
grep -c "slack" /var/ossec/logs/integrations.log
```

**[Place screenshot: Shows statistics of number of alerts sent]**

### Troubleshooting

#### *Issue*: *Alerts Not Being Sent to Slack*

**Solutions:**

1. **Verify Webhook URL Validity:**

```bash
# Test Webhook manually
curl -X POST -H 'Content-type: application/json' \
--data '{"text":"Test message from Wazuh"}' \
YOUR_SLACK_WEBHOOK_URL
```

**[Place screenshot: Shows testing Webhook manually from command line]**

**[Place screenshot: Shows test message arriving in Slack successfully]**

2. **Check Alert Levels:**

```bash
# Ensure generated alerts exceed specified level
grep "level" /var/ossec/logs/alerts/alerts.json | tail -20
```

**[Place screenshot: Shows alert levels in alerts.json]**

3. **Check integrator Errors:**

```bash
grep -i "error\|fail" /var/ossec/logs/integrations.log | tail -20
```

**[Place screenshot: Shows error log in integrations.log]**

4. **Verify ossec.conf Configuration:**

```bash
# Verify XML format validity
/var/ossec/bin/verify-agent-conf
```

**[Place screenshot: Shows configuration validation result]**

#### *Issue*: *Duplicate or Too Many Alerts*

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

**[Place screenshot: Shows updated configuration in ossec.conf]**

3. **Exclude Certain Rules:**

In `/var/ossec/etc/rules/local_rules.xml`:

```xml
<rule id="100500" level="0">
    <if_sid>NOISY_RULE_ID</if_sid>
    <description>Excluding noisy rule from Slack</description>
</rule>
```

#### *Issue*: *Alerts Don't Contain Enough Information*

**Solution:** Use custom script as shown in the "Customize Alert Format" section

### Best Practices

#### 1. *Organize Channels*

- Channel for Critical alerts
- Channel for Malware and Vulnerabilities
- Channel for Network attacks
- Channel for Periodic reports

**[Place screenshot: Shows organized Slack channel list by alert type]**

#### 2. *Set Up Appropriate Alerts*

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

**[Place screenshot: Shows complete configuration with explanatory comments]**

#### 3. *Create Naming Convention*

- Use clear prefixes: `[CRITICAL]`, `[HIGH]`, `[MALWARE]`
- Add emojis for quick distinction
- Include affected machine name

**[Place screenshot: Shows Slack alerts with unified and clear naming]**

#### 4. *Set Up Workflows in Slack*

You can create automatic workflows in Slack:

- Create ticket in task management system
- Send notification to specific person
- Log event in spreadsheet

**[Place screenshot: Shows Slack Workflow Builder setup]**

#### 5. *Periodic Review*

```bash
# Create weekly report of alert count
cat /var/ossec/logs/integrations.log | \
grep "slack" | \
awk '{print $1}' | \
sort | uniq -c | \
tail -7
```

**[Place screenshot: Shows statistical report of number of alerts sent daily]**

### Create Daily Summary Report

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

# Create message
MESSAGE='{
  "text": "📊 *Wazuh Daily Security Report - '"$DATE"'*",
  "attachments": [
    {
      "color": "#36a64f",
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
    }
  ]
}'

# Send report
curl -X POST -H 'Content-type: application/json' --data "$MESSAGE" $WEBHOOK_URL
```

**[Place screenshot: Shows daily report script content]**

Add to crontab for daily execution:

```bash
# Run report daily at 9 AM
0 9 * * * /usr/local/bin/wazuh-daily-slack-report.sh
```

**[Place screenshot: Shows adding script to crontab]**

**[Place screenshot: Shows daily report in Slack with summarized statistics]**

### Alerts

Confirmation of sent alerts can be found:

```bash
# In Wazuh logs
grep "slack" /var/ossec/logs/integrations.log

# In Slack
# Check different channels
```

**[Place screenshot: Shows successful alert sending log in integrations.log]**

**[Place screenshot: Shows diverse range of alerts in a single Slack channel]**

### Affected Endpoint

- Wazuh Manager (for sending alerts)
- All Windows endpoints (as source of alerts)

## Conclusion

This guide covered the most important use cases for Wazuh on Windows endpoints:

1. ✅ **Brute Force Attack Detection on RDP** - Protection from unauthorized access attempts
2. ✅ **File Integrity Monitoring (FIM)** - Track changes to important files
3. ✅ **Malware Detection and Removal (VirusTotal)** - Proactive protection from malicious files
4. ✅ **Vulnerability Detection** - Manage security updates and patches
5. ✅ **Slack Integration** - Immediate notifications for important security events

**[Place screenshot: Shows comprehensive Kibana dashboard combining all alert types mentioned above]**

### Recommended Next Steps:

1. **Create Custom Security Policies:**
   - Define appropriate rules for your environment
   - Adjust alert levels
   - Create custom rules for special cases

2. **Automate Responses:**
   - Develop additional active responses
   - Integrate with automation tools (Ansible, PowerShell)
   - Create playbooks for common incidents

3. **Documentation and Compliance:**
   - Document all security procedures
   - Create periodic reports for management
   - Ensure compliance with standards (PCI-DSS, HIPAA, etc.)

4. **Training and Continuous Improvement:**
   - Train team on using Wazuh
   - Review and update rules regularly
   - Stay informed about latest threats

**[Place screenshot: Shows architectural diagram showing how all components work together (Wazuh Manager, Windows Agents, Elasticsearch, Kibana, Slack)]**

### Additional Resources:

- Official Documentation: https://documentation.wazuh.com
- Forum: https://groups.google.com/forum/#!forum/wazuh
- GitHub: https://github.com/wazuh/wazuh
- Slack Community: https://wazuh.com/community/join-us-on-slack/

**[Place screenshot: Shows additional resources page open in browser]**

## Important Final Notes:

⚠️ **Security Warnings:**
- Don't share API Keys or Webhook URLs publicly
- Use HTTPS for all communications
- Change default passwords
- Review permissions regularly

🔒 **Best Practices:**
- Test all configurations in a test environment first
- Keep backups of configuration files
- Monitor resource consumption (CPU, Memory, Disk)
- Update Wazuh and components regularly

📝 **Documentation:**
- Document all configuration changes
- Keep a log of incidents and actions taken
- Create a runbook for your team

**[Place screenshot: Shows final summary document of all steps and configurations]**

This guide was prepared to help you deploy and operate Wazuh efficiently in a Windows environment. Remember that cybersecurity is an ongoing process that requires continuous monitoring and updating.

**We wish you a safe and successful experience with Wazuh! 🛡️**