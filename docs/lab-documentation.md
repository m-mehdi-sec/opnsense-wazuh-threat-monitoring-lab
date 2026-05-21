# Lab Documentation – OPNsense Wazuh Threat Monitoring Lab

## Objective

The objective of this lab was to install, configure, and validate Wazuh as a SIEM and endpoint monitoring platform in a controlled Hyper-V lab environment.

The lab focused on practical Blue Team work: deploying a Wazuh server, connecting Windows endpoints, forwarding OPNsense firewall logs through syslog, simulating authentication-related threats, validating process execution through Sysmon, testing File Integrity Monitoring, and troubleshooting log ingestion until the full log path could be verified.

All testing was performed inside an isolated local lab environment.

---

## Lab Scenario

The lab was based on a medium-sized company scenario with separate departments and network segments.

In this lab, the environment represented:

- A main LAN used by HR and the Wazuh server
- A separate Support network
- OPNsense as firewall and gateway
- Windows clients acting as monitored endpoints
- Wazuh as the central monitoring and analysis platform

The original assignment used a slightly different IP plan, but the environment was adapted to fit the existing Hyper-V and OPNsense lab. The core goal remained the same: centralized monitoring, endpoint visibility, firewall log collection, and threat analysis.

---

## Lab Environment

| System | Role | IP Address | Network |
|---|---|---:|---|
| OPNsense_Firewall | Firewall / gateway | 192.168.10.1 | LAN |
| Wazuh_Server | Wazuh Manager / Dashboard | 192.168.10.150 | LAN |
| HR_Client | Windows HR endpoint | 192.168.10.101 | LAN |
| Support_Client | Windows Support endpoint | 192.168.30.101 | Support_LAN |

> Note: Some Wazuh alerts show the HR agent name as `HR_Ansatlld` due to a lab naming typo. This refers to the HR client agent.

---

## Network Segments

| Network | Purpose | Gateway |
|---|---|---:|
| 192.168.10.0/24 | Main LAN, HR client, Wazuh server | 192.168.10.1 |
| 192.168.30.0/24 | Support client network | 192.168.30.1 |

---

## Hyper-V Preparation

The existing Hyper-V environment was reviewed before starting the Wazuh configuration.

The following checks were completed:

- OPNsense was running
- The LAN network was reachable
- Ubuntu Server had internet access
- Windows clients were available
- Virtual switches were mapped to the correct networks
- Support_LAN was configured and active in OPNsense

A Hyper-V checkpoint was created before the Wazuh installation.

Checkpoint name:

    Before Wazuh install

This provided a rollback point in case the Wazuh installation or configuration failed.

---

## VM Naming

The VM names were adjusted to make the lab easier to follow and document.

| VM Name | Purpose |
|---|---|
| Wazuh_Server | Wazuh SIEM server |
| HR_Client | HR Windows endpoint |
| Support_Client | Support Windows endpoint |
| OPNsense_Firewall | Firewall and gateway |

Clear names made screenshots, troubleshooting, and documentation easier to understand.

---

## OPNsense Interface Configuration

OPNsense was used as the firewall and gateway.

The main LAN interface was already configured as:

    LAN: 192.168.10.1/24

This network was used by:

- Wazuh_Server
- HR_Client
- OPNsense management access

A separate Support network was configured as:

    Support_LAN: 192.168.30.1/24

The Support_LAN interface was connected to:

    Support_LAN_Switch

The interface status was verified as:

    up

This created a segmented environment where the Support client was separated from the main LAN.

---

## Wazuh Server Static IP Configuration

Ubuntu Server initially received a dynamic IP address:

    192.168.10.146/24

Because Wazuh agents and OPNsense syslog forwarding require a stable destination, the Wazuh server was changed to a static IP address.

Final Wazuh server network configuration:

| Setting | Value |
|---|---|
| IP address | 192.168.10.150 |
| Subnet mask | 255.255.255.0 |
| CIDR | /24 |
| Gateway | 192.168.10.1 |
| DNS | 8.8.8.8 / 1.1.1.1 |

The Netplan file used was:

    /etc/netplan/50-cloud-init.yaml

Connectivity was verified with:

    ping 192.168.10.1
    ping 8.8.8.8

Both tests were successful.

Observation:

The Wazuh server had working local connectivity and internet access before installation.

---

## Wazuh Installation

Wazuh was installed on Ubuntu Server using the automated installation script.

Commands used:

    sudo apt update
    sudo apt install -y curl
    curl -sO https://packages.wazuh.com/4.10/wazuh-install.sh
    sudo bash wazuh-install.sh -a

The first installation attempt failed because the VM did not meet the recommended hardware requirements.

The VM was adjusted to:

| Resource | Value |
|---|---:|
| Memory | 4096 MB |
| Processor | 2 virtual CPUs |

After increasing memory and CPU, the installation completed successfully.

The Wazuh dashboard was accessed at:

    https://192.168.10.150

Observation:

Wazuh required more resources than a basic Ubuntu server. After increasing the VM resources, the installation completed successfully.

---

## Wazuh Dashboard Access

The Wazuh dashboard was accessed from a Windows client.

A browser certificate warning appeared because the lab used a self-signed certificate. This was expected in a local lab.

The generated admin password from the Wazuh installation was used.

The assignment suggested changing the admin password, but the password change failed because the built-in admin user was reserved in this Wazuh version.

Observed message:

    Resource 'admin' is reserved.

The generated installation password was kept for the lab.

Observation:

The dashboard was accessible and working, but the admin password change step was skipped because of a version-specific restriction.

---

## Wazuh Agent Deployment

Agents were deployed from the Wazuh dashboard using:

    Endpoints -> Deploy new agent

The selected operating system package was:

    Windows MSI 32/64 bits

The Wazuh server address used for both Windows agents was:

    192.168.10.150

The generated PowerShell command was executed as Administrator on each Windows client.

The Wazuh agent service was started with:

    NET START WazuhSvc

---

## HR Client Agent

The HR client was configured with:

| Setting | Value |
|---|---|
| VM name | HR_Client |
| IP address | 192.168.10.101 |
| Subnet mask | 255.255.255.0 |
| Gateway | 192.168.10.1 |
| DNS | 8.8.8.8 / 1.1.1.1 |
| Wazuh agent name | HR_Anstalld |

The agent appeared in Wazuh as:

    HR_Anstalld

Status:

    Active

Observation:

The HR endpoint successfully connected to Wazuh and began sending events.

---

## Support Client Agent

The Support client was connected to:

    Support_LAN_Switch

The static IP configuration was:

| Setting | Value |
|---|---|
| VM name | Support_Client |
| IP address | 192.168.30.101 |
| Subnet mask | 255.255.255.0 |
| Gateway | 192.168.30.1 |
| DNS | 8.8.8.8 / 1.1.1.1 |
| Wazuh agent name | Support_Anstalld |

During configuration, the gateway was first entered incorrectly as:

    192.168.10.1

Windows warned that the default gateway was not on the same network segment as the IP address.

The gateway was corrected to:

    192.168.30.1

Connectivity was verified with:

    ping 192.168.30.1
    ping 192.168.10.150

Both tests were successful.

The agent appeared in Wazuh as:

    Support_Anstalld

Status:

    Active

Observation:

The Support endpoint successfully reached the Wazuh server from a separate network segment.

---

## Agent Verification

After deployment, Wazuh showed both Windows agents as active.

| Agent | Status |
|---|---|
| HR_Anstalld | Active |
| Support_Anstalld | Active |

Observation:

Both endpoints were successfully monitored by Wazuh.

---

## Threat Hunting Verification

Wazuh Threat Hunting was used to verify endpoint events.

The interface showed:

- Agent names
- Event timestamps
- Rule descriptions
- Rule IDs
- Rule levels
- Windows authentication events

Observation:

Threat Hunting confirmed that Wazuh was collecting and displaying Windows endpoint events.

---

## Brute Force Simulation

The goal was to simulate repeated failed login attempts against the HR client.

The current Windows user was checked with:

    whoami

Observed result:

    desktop-th2mb3n\mehdi

Failed login attempts were generated with:

    runas /user:mehdi cmd

Several incorrect passwords were entered.

Wazuh detected the failed login attempts in Threat Hunting.

| Field | Value |
|---|---|
| Agent | HR_Anstalld |
| Rule description | Logon Failure - Unknown user or bad password |
| Rule ID | 60122 |
| Rule level | 5 |

Observation:

Wazuh successfully detected failed authentication activity from the HR endpoint. This validated the brute force simulation.

---

## Successful Login Monitoring

The Support client was used to verify successful login monitoring.

After logging out and logging in again, Wazuh showed:

| Field | Value |
|---|---|
| Agent | Support_Anstalld |
| Rule description | Windows Logon Success |
| Rule ID | 60106 |
| Rule level | 3 |

Observation:

Successful login events were visible in Wazuh. These events are useful for timeline building during investigations.

---

## Suspicious Login Simulation

The assignment suggested simulating a suspicious login by changing the time zone on the Support client.

The time zone was changed to:

    (UTC+08:00) Beijing, Chongqing, Hong Kong, Urumqi

After the change, the user logged out and logged in again.

Wazuh detected the login event:

| Field | Value |
|---|---|
| Agent | Support_Anstalld |
| Rule description | Windows Logon Success |
| Rule ID | 60106 |

No country or geolocation alert was generated.

Explanation:

The lab used private internal IP addresses, such as:

    192.168.30.101

Private IP addresses do not provide public geolocation data. Changing the local Windows time zone does not make the login originate from another country.

Observation:

Wazuh detected the login event, but no geolocation-based anomaly was triggered. This showed the limitation of this simulation in a private lab environment.

---

## High Traffic / DDoS Simulation

High ICMP traffic was simulated from HR_Client against the OPNsense gateway.

Command used:

    ping 192.168.10.1 -t -l 65500

The goal was to simulate abnormal network activity and check whether Wazuh would generate a DDoS-style alert.

Result:

A dedicated DDoS alert was not generated automatically by the default Wazuh rules.

Observation:

The test generated traffic, but default Wazuh rules did not classify it as a DDoS alert. This showed that reliable DDoS detection usually requires additional telemetry, such as IDS/IPS logs, NetFlow, firewall counters, or custom Wazuh rules.

---

## Extended Endpoint Detection Validation

After the initial Wazuh lab was completed, additional validation was performed to test whether Wazuh could detect more detailed endpoint activity.

The focus was on:

- Process execution visibility
- Sysmon-based detections
- Command execution from Windows
- File Integrity Monitoring
- Custom folder monitoring through syscheck
- Troubleshooting why some events were not visible at first

This extended validation made the lab more realistic from a SOC and Blue Team perspective because endpoint detection often requires tuning, additional telemetry, and verification.

---

## Wazuh Agent Troubleshooting on HR Client

During the extended validation, some expected Windows events were not visible in Wazuh.

The Wazuh agent service was checked on the HR client:

    Get-Service wazuhsvc

Initial result:

    Stopped

The service was started again:

    Start-Service wazuhsvc

The Wazuh agent log was reviewed with:

    Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50

Important log entries showed that the agent connected successfully to the Wazuh server:

    Connected to the server ([192.168.10.150]:1514/tcp)
    Agent is now online

Observation:

The missing events were partly caused by the Wazuh agent being stopped. After starting the service, new endpoint events were sent to Wazuh again.

---

## Sysmon Process Execution Validation

The next step was to verify whether command execution could be detected in Wazuh.

The following commands were executed on the HR client through `cmd.exe`:

    cmd.exe /c whoami
    cmd.exe /c ipconfig

These commands were used because executing them through `cmd.exe` generated clearer Sysmon process creation events in Wazuh.

---

## ipconfig Detection in Wazuh

Wazuh detected the `ipconfig` command as a Sysmon process creation event.

Observed fields:

| Field | Value |
|---|---|
| Agent | HR_Anstalld |
| Event source | Microsoft-Windows-Sysmon/Operational |
| Sysmon Event ID | 1 |
| Image | C:\Windows\System32\ipconfig.exe |
| Command line | ipconfig |
| Parent image | C:\Windows\System32\cmd.exe |
| Parent command line | C:\WINDOWS\system32\cmd.exe /c ipconfig |
| Rule description | Suspicious Windows cmd shell execution |
| Rule ID | 92032 |
| Rule level | 3 |
| MITRE tactic | Discovery, Execution |
| MITRE technique | Account Discovery, Windows Command Shell |
| MITRE ID | T1087, T1059.003 |

Observation:

Wazuh successfully detected `ipconfig.exe` execution through Sysmon. This demonstrated process-level visibility beyond basic Windows login events.

---

## whoami Detection in Wazuh

Wazuh also detected the `whoami` command as a Sysmon process creation event.

Observed fields:

| Field | Value |
|---|---|
| Agent | HR_Anstalld |
| Event source | Microsoft-Windows-Sysmon/Operational |
| Sysmon Event ID | 1 |
| Image | C:\Windows\System32\whoami.exe |
| Command line | whoami |
| Parent image | C:\Windows\System32\cmd.exe |
| Parent command line | C:\WINDOWS\system32\cmd.exe /c whoami |
| Rule description | Suspicious Windows cmd shell execution |
| Rule ID | 92032 |
| Rule level | 3 |
| MITRE tactic | Discovery, Execution |
| MITRE technique | Account Discovery, Windows Command Shell |
| MITRE ID | T1087, T1059.003 |

Observation:

Wazuh successfully detected `whoami.exe` execution through Sysmon. This is useful from a threat hunting perspective because commands like `whoami` are commonly used during discovery after initial access.

---

## File Integrity Monitoring Test Preparation

A custom test folder and file were created on the HR client.

Commands used:

    New-Item -ItemType Directory -Path C:\WazuhLab -Force
    Set-Content -Path C:\WazuhLab\test.txt -Value "test från Wazuh lab"
    Add-Content -Path C:\WazuhLab\test.txt -Value "ny ändring"

The file was verified with:

    Test-Path C:\WazuhLab
    Test-Path C:\WazuhLab\test.txt
    dir C:\WazuhLab
    Get-Content C:\WazuhLab\test.txt

Observed result:

    True
    True

The folder and file existed on the HR client.

Observation:

Creating the file was not enough by itself. Wazuh did not detect the custom folder until it was added to the File Integrity Monitoring configuration.

---

## File Integrity Monitoring Configuration

The Wazuh agent configuration file was opened on the HR client:

    C:\Program Files (x86)\ossec-agent\ossec.conf

The following line was added inside the `<syscheck>` block:

    <directories check_all="yes" realtime="yes" report_changes="yes">C:\WazuhLab</directories>

The Wazuh agent was restarted:

    Stop-Service wazuhsvc -Force
    Start-Service wazuhsvc

The agent log was checked:

    Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 100 | Select-String "WazuhLab"

Observed result:

    Monitoring path: 'c:\wazuhlab', with options 'size | permissions | owner | group | mtime | inode | hash_md5 | hash_sha1 | hash_sha256 | attributes | report_changes | realtime'

Observation:

This confirmed that Wazuh File Integrity Monitoring was actively monitoring `C:\WazuhLab`.

---

## File Integrity Monitoring Alert

After the folder was added to syscheck, the test file was modified again.

Command used:

    Add-Content C:\WazuhLab\test.txt "FIM test efter att WazuhLab övervakas"

Wazuh generated a File Integrity Monitoring alert.

Observed fields:

| Field | Value |
|---|---|
| Agent | HR_Anstalld |
| Location | syscheck |
| Decoder | syscheck_integrity_changed |
| File path | c:\wazuhlab\test.txt |
| Syscheck event | modified |
| Mode | realtime |
| Rule description | Integrity checksum changed |
| Rule ID | 550 |
| Rule level | 7 |
| Changed attributes | size, mtime, md5, sha1, sha256 |
| MITRE tactic | Impact |
| MITRE technique | Stored Data Manipulation |
| MITRE ID | T1565.001 |

The alert showed that the file changed from size `33` to `72`, and that the MD5, SHA1, and SHA256 hashes changed.

Observation:

Wazuh successfully detected that `test.txt` was modified. This confirmed that File Integrity Monitoring worked after the custom folder was added to syscheck.

---

## OPNsense Syslog Configuration

OPNsense was configured to forward logs to Wazuh.

OPNsense path:

    System -> Settings -> Logging -> Remote

Configuration:

| Setting | Value |
|---|---|
| Enabled | Yes |
| Transport | UDP |
| Destination | 192.168.10.150 |
| Port | 514 |
| Applications | All |
| Levels | All |
| Facilities | All |
| Description | Wazuh Syslog |

Observation:

OPNsense was configured to send firewall and system logs to Wazuh over UDP 514.

---

## Wazuh Syslog Receiver Configuration

The Wazuh configuration file was edited:

    /var/ossec/etc/ossec.conf

Final syslog receiver block:

    <remote>
      <connection>syslog</connection>
      <port>514</port>
      <protocol>udp</protocol>
      <allowed-ips>192.168.10.1</allowed-ips>
      <local_ip>0.0.0.0</local_ip>
    </remote>

After editing the file, Wazuh Manager was restarted:

    sudo systemctl restart wazuh-manager

The service status was checked with:

    sudo systemctl status wazuh-manager

Expected result:

    active (running)

Observation:

The final syslog receiver configuration was accepted and Wazuh Manager started successfully.

---

## Troubleshooting – Wazuh Syslog Integration

This was one of the most important troubleshooting parts of the lab.

The issue was that OPNsense was configured to send syslog traffic, but the logs were not immediately visible in Wazuh.

### Symptoms

- Windows agent events were visible
- OPNsense remote syslog was configured
- Wazuh Manager was running
- OPNsense logs were not initially visible in Wazuh
- Searches for ICMP or firewall events did not show the expected results

---

## Troubleshooting Step 1 – Fix ossec.conf Structure

The Wazuh configuration file was edited several times:

    /var/ossec/etc/ossec.conf

During early attempts, the remote syslog block was placed incorrectly outside the final ossec_config section.

This caused Wazuh Manager restart errors.

Observed error:

    Job for wazuh-manager.service failed because the control process exited with error code.

Cause:

The XML structure was invalid.

Issues included:

- remote block placed outside ossec_config
- duplicate closing ossec_config tags
- incorrectly typed closing tag
- invalid placement of the syslog receiver block

Fix:

The remote block was placed inside the final ossec_config section and the closing tag was corrected.

Verification:

    sudo systemctl restart wazuh-manager
    sudo systemctl status wazuh-manager

Result:

    active (running)

Observation:

Wazuh Manager only started successfully after the XML structure was corrected.

---

## Troubleshooting Step 2 – Verify UDP 514 with tcpdump

Even after Wazuh Manager accepted the configuration, it was necessary to verify whether OPNsense traffic actually reached the server.

tcpdump was not installed by default.

Initial result:

    sudo: tcpdump: command not found

tcpdump was installed with:

    sudo apt install -y tcpdump

The syslog traffic was checked with:

    sudo tcpdump -i any udp port 514

Observed traffic included:

    _gateway.45405 > ubuntu-server.syslog: SYSLOG auth.info
    _gateway.45405 > ubuntu-server.syslog: SYSLOG user.notice
    _gateway.45405 > ubuntu-server.syslog: SYSLOG local0.info

Observation:

tcpdump confirmed that OPNsense was sending syslog packets to the Wazuh server. The network path was working.

---

## Troubleshooting Step 3 – Check archives.log

Raw Wazuh logs were checked with:

    sudo tail -f /var/ossec/logs/archives/archives.log

Initial result:

The file stayed empty during testing.

Observation:

Syslog packets were reaching Ubuntu, but Wazuh was not yet writing them into archives.log.

---

## Troubleshooting Step 4 – Enable logall and logall_json

The Wazuh logall settings were checked with:

    sudo grep -i logall /var/ossec/etc/ossec.conf

Observed result:

    <logall>no</logall>
    <logall_json>no</logall_json>

The values were changed to:

    <logall>yes</logall>
    <logall_json>yes</logall_json>

Wazuh Manager was restarted:

    sudo systemctl restart wazuh-manager

archives.log was monitored again:

    sudo tail -f /var/ossec/logs/archives/archives.log

Result:

OPNsense firewall logs appeared in archives.log.

Observed log types included:

- OPNsense filterlog
- pass in / pass out traffic
- UDP traffic
- DNS traffic
- firewall-related syslog entries

Observation:

Enabling logall and logall_json made the raw OPNsense logs visible in archives.log.

---

## Final Syslog Validation

The final verified log path was:

    OPNsense -> Syslog UDP 514 -> Wazuh_Server -> archives.log

Validation methods:

| Method | Result |
|---|---|
| tcpdump | Confirmed UDP 514 packets arriving |
| archives.log | Confirmed raw OPNsense logs |
| systemctl status | Wazuh Manager active |
| OPNsense remote logging | Configured and active |

Observation:

The syslog integration worked after troubleshooting. The main issue was not network connectivity, but Wazuh log visibility and archive logging.

---

## DDoS Alert Limitation

A dedicated DDoS alert was not triggered by default.

Possible reasons:

- The default Wazuh rules did not classify the ICMP test as DDoS
- OPNsense firewall logs did not provide enough structured DDoS context
- A single client ping test is a limited simulation
- Custom rules may be required
- More advanced telemetry would improve detection quality

Useful future sources could include:

- Suricata IDS/IPS logs
- NetFlow
- Firewall counters
- Custom Wazuh rules
- Better traffic volume telemetry

Observation:

This was an important learning point. A SIEM does not automatically detect every suspicious scenario. Detection quality depends on the log source, parser, rule logic, and context.

---

## Final Validation

| Component | Status |
|---|---|
| Wazuh Server | Working |
| Wazuh Dashboard | Working |
| HR Agent | Active |
| Support Agent | Active |
| Failed login detection | Working |
| Successful login monitoring | Working |
| Suspicious login simulation | Login event detected |
| Sysmon process monitoring | Working |
| whoami detection | Working through Sysmon Event ID 1 |
| ipconfig detection | Working through Sysmon Event ID 1 |
| File Integrity Monitoring | Working after adding C:\WazuhLab to syscheck |
| test.txt modification detection | Working, rule ID 550 |
| OPNsense syslog forwarding | Working |
| tcpdump verification | Working |
| archives.log verification | Working |
| Default DDoS alert | Not triggered |

---

## Key Findings

### Finding 1 – Windows Endpoint Monitoring Worked

Wazuh successfully collected authentication events from both Windows clients.

Events included:

- Windows Logon Success
- Logon Failure - Unknown user or bad password

---

### Finding 2 – Failed Login Attempts Were Detected

The brute force simulation generated failed login events.

Wazuh detected them with:

| Field | Value |
|---|---|
| Rule ID | 60122 |
| Rule level | 5 |
| Description | Logon Failure - Unknown user or bad password |

---

### Finding 3 – Suspicious Login Simulation Had Limits

Changing the Windows time zone generated a login event, but not a country-based alert.

This was expected because the lab used private internal IP addresses.

---

### Finding 4 – OPNsense Syslog Required Troubleshooting

OPNsense syslog forwarding required deeper troubleshooting before the logs became visible in archives.log.

The most useful tools were:

- systemctl
- tcpdump
- archives.log
- logall and logall_json settings

---

### Finding 5 – DDoS Detection Requires Better Telemetry

The ICMP traffic test did not trigger a default DDoS alert.

This showed that DDoS-style detection usually needs better telemetry, tuned rules, or IDS/IPS integration.

---

### Finding 6 – Sysmon Improved Endpoint Process Visibility

Wazuh detected command execution from the HR client through Sysmon Event ID 1.

Detected commands included:

- whoami.exe
- ipconfig.exe

Both were executed through `cmd.exe` and detected in Wazuh with rule ID `92032`.

This improved the endpoint visibility compared to relying only on basic Windows authentication events.

---

### Finding 7 – File Integrity Monitoring Worked After Tuning

The custom folder `C:\WazuhLab` was not detected automatically at first.

After adding it to the Wazuh agent syscheck configuration, Wazuh detected modifications to:

    c:\wazuhlab\test.txt

The alert was generated with:

| Field | Value |
|---|---|
| Rule description | Integrity checksum changed |
| Rule ID | 550 |
| Rule level | 7 |
| Syscheck event | modified |
| Mode | realtime |

This confirmed that File Integrity Monitoring worked after the correct folder was added to the monitoring configuration.

---

## Lessons Learned

This lab showed that deploying a SIEM is not only about installing a tool. It also requires validation and troubleshooting.

Main lessons:

- Wazuh provides useful endpoint visibility
- Failed login attempts are easy to validate in Threat Hunting
- Successful login events are useful for investigation timelines
- OPNsense can forward logs to Wazuh using syslog
- tcpdump confirms whether logs reach the server
- archives.log confirms whether Wazuh stores raw logs
- XML structure in ossec.conf must be correct
- Wazuh Manager fails to start if the configuration is invalid
- logall and logall_json affect raw log visibility
- Default rules do not always generate the expected alert
- Custom rules and tuning are important in real SOC work
- Sysmon provides better process creation visibility than basic Windows logs alone
- Commands such as whoami and ipconfig are useful for threat hunting validation
- File Integrity Monitoring requires the correct monitored path
- A stopped agent can make events disappear from the SIEM even when they exist locally
- Troubleshooting is a central part of Blue Team methodology

---

## Security Reflection

Wazuh complements vulnerability scanning by providing continuous monitoring.

A vulnerability scanner can identify weaknesses at a specific point in time, while Wazuh can monitor ongoing activity such as:

- Failed login attempts
- Successful logins
- Endpoint events
- Process execution
- File modification
- Firewall logs
- Suspicious behavior patterns

This makes Wazuh useful for detecting activity that vulnerability scanning alone cannot identify.

The extended validation also showed an important SOC lesson: if an event is not visible in the SIEM, the issue may be caused by agent status, missing telemetry, missing configuration, or search/filter limitations. A security analyst must be able to troubleshoot the full chain from endpoint event generation to SIEM alert visibility.

---

## Relation to Zero Trust and Segmentation

The lab also supported Zero Trust and segmentation concepts.

The HR and Support clients were placed in separate networks. Wazuh provided visibility into endpoint behavior across these segments.

In a production environment, Wazuh could support Zero Trust by monitoring authentication events, detecting abnormal login behavior, correlating firewall and endpoint logs, monitoring endpoint process execution, detecting file changes on sensitive systems, and supporting investigations when access patterns look suspicious.

---

## What Could Be Improved

Future improvements:

- Create custom Wazuh rules for ICMP flood detection
- Add Suricata IDS/IPS logs from OPNsense
- Forward more detailed firewall logs
- Expand Sysmon configuration for broader endpoint telemetry
- Create dashboards for authentication monitoring
- Create dashboards for Sysmon process execution events
- Create dashboards for File Integrity Monitoring events
- Test Microsoft 365 or cloud identity logs
- Use NetFlow or traffic analysis for better DDoS visibility
- Create active response rules in Wazuh
- Add email or notification alerts
- Test command execution detection using PowerShell logging
- Add more monitored folders for FIM validation
- Create custom correlation rules between firewall logs and endpoint events

---

## Conclusion

The lab successfully demonstrated how Wazuh can be deployed and used for endpoint monitoring, authentication event analysis, process execution monitoring, File Integrity Monitoring, and firewall log collection in a segmented Hyper-V environment.

The strongest initial results were Windows authentication monitoring and failed login detection from the HR client. OPNsense syslog forwarding was also successfully validated after troubleshooting with tcpdump and archives.log.

The extended validation strengthened the lab by proving that Wazuh could also detect command execution through Sysmon and file modification through File Integrity Monitoring. Commands such as `whoami` and `ipconfig` were detected as Sysmon process creation events, and changes to `C:\WazuhLab\test.txt` were detected as an integrity checksum change.

The DDoS simulation did not trigger a default Wazuh DDoS alert, but this became an important learning point. It showed that reliable SIEM detection often requires the right log sources, correct parsing, custom rules, and proper tuning.

Overall, this lab provided practical experience with Wazuh, OPNsense, syslog, endpoint monitoring, Sysmon, File Integrity Monitoring, troubleshooting, and SOC-style analysis.
