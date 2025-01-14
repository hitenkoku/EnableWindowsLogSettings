<div align="center">
 <h1>
    Yamato Security's ultimate guide to configuring and monitoring Windows event logs for Sigma users
 </h1>
 [ <b>English</b> ] | [<a href="README-Japanese.md">日本語</a>]
</div>
<p>

This is yet another guide on configuring and monitoring Windows event logs with an emphasis on making sure you have the proper logging enabled so that sigma rules have something to detect.

## Table of Contents

- [Author](#author)
- [Acknowledgements](#acknowledgements)
- [Problems with the default Windows log settings](#problems-with-the-default-windows-log-settings)
- [Warning: Use at your own risk!](#warning-use-at-your-own-risk)
- [Important Windows event logs](#important-windows-event-logs)
  - [Sigma's top log sources](#sigmas-top-log-sources)
    - [Top sigma log sources](#top-sigma-log-sources)
    - [Top Security Event IDs](#top-security-event-ids)
- [Increasing the maximum file size](#increasing-the-maximum-file-size)
  - [Option 1: Manually through Event Viewer](#option-1-manually-through-event-viewer)
  - [Option 2: Windows built-in tool](#option-2-windows-built-in-tool)
  - [Option 3: PowerShell](#option-3-powershell)
  - [Option 4: Group Policy](#option-4-group-policy)
- [Configuration script](#configuration-script)
- [Configuring log settings](#configuring-log-settings)
  - [Sysmon log (1382 sigma rules)](#sysmon-log-1382-sigma-rules)
  - [Security log (1045 sigma rules (903 process creation rules + 142 other rules))](#security-log-1045-sigma-rules-903-process-creation-rules--142-other-rules)
  - [Powershell logs (175 sigma rules)](#powershell-logs-175-sigma-rules)
    - [Module logging (30 sigma rules)](#module-logging-30-sigma-rules)
      - [Enabling module logging](#enabling-module-logging)
        - [Option 1: Enabling through group policy](#option-1-enabling-through-group-policy)
        - [Option 2: Enabling through the registry](#option-2-enabling-through-the-registry)
    - [Script Block Logging (134 sigma rules)](#script-block-logging-134-sigma-rules)
      - [Enabling Script Block logging](#enabling-script-block-logging)
      - [Option 1: Enabling through group policy](#option-1-enabling-through-group-policy-1)
      - [Option 2: Enabling through the registry](#option-2-enabling-through-the-registry-1)
    - [Transcription logging](#transcription-logging)
      - [Enabling Transcription logging](#enabling-transcription-logging)
        - [Option 1: Enabling through group policy](#option-1-enabling-through-group-policy-2)
        - [Option 2: Enabling through the registry](#option-2-enabling-through-the-registry-2)
    - [References](#references)
  - [System log (55 sigma rules)](#system-log-55-sigma-rules)
  - [Application log (16 sigma rules)](#application-log-16-sigma-rules)
  - [Windows Defender Operational log (10 sigma rules)](#windows-defender-operational-log-10-sigma-rules)
  - [Bits-Client Operational log (6 sigma rules)](#bits-client-operational-log-6-sigma-rules)
  - [Firewall log (6 sigma rules)](#firewall-log-6-sigma-rules)
  - [NTLM Operational log (3 sigma rules)](#ntlm-operational-log-3-sigma-rules)
  - [Security-Mitigations KernelMode and UserMode logs  (2 sigma rules)](#security-mitigations-kernelmode-and-usermode-logs--2-sigma-rules)
  - [PrintService logs (2 sigma rules)](#printservice-logs-2-sigma-rules)
    - [Admin (1 sigma rule)](#admin-1-sigma-rule)
    - [Operational (1 sigma rule)](#operational-1-sigma-rule)
  - [SMBClient Security log (2 sigma rules)](#smbclient-security-log-2-sigma-rules)
  - [AppLocker logs (1 sigma rule)](#applocker-logs-1-sigma-rule)
  - [CodeIntegrity Operational log (1 sigma rule)](#codeintegrity-operational-log-1-sigma-rule)
  - [Diagnosis-Scripted Operational log (1 sigma rule)](#diagnosis-scripted-operational-log-1-sigma-rule)
  - [DriverFrameworks-UserMode Operational log  (1 sigma rule)](#driverframeworks-usermode-operational-log--1-sigma-rule)
  - [WMI-Activity Operational log  (1 sigma rule)](#wmi-activity-operational-log--1-sigma-rule)
  - [TerminalServices-LocalSessionManager Operational log  (1 sigma rule)](#terminalservices-localsessionmanager-operational-log--1-sigma-rule)
  - [TaskScheduler Operational log  (1 sigma rule)](#taskscheduler-operational-log--1-sigma-rule)

# Author
 
Zach Mathis ([@yamatosecurity](https://twitter.com/yamatosecurity)). As I do more research and testing, I plan on periodically updating this as there is much room for improvement (both in the documentation as well as in creating more detection rules.) PRs are welcome and will gladly add you as a contributor.

If you find any of this useful, please give a star on GitHub as it will probably help motivate me to continue updating this.

# Acknowledgements

Most of the information comes from Microsoft's [Advanced security auditing FAQ](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/advanced-security-auditing-faq), [sigma](https://github.com/SigmaHQ/sigma) rules, the [ACSC guide](https://www.cyber.gov.au/acsc/view-all-content/publications/windows-event-logging-and-forwarding) and my own research/testing. I would like to thank the [sigma community](https://github.com/SigmaHQ/sigma/graphs/contributors) in particular for making threat detection open source and free for the benefit of all of the defenders out there.

# Problems with the default Windows log settings

By default, Windows will not log many events necessary for detecting malicious activity and performing forensics investigations.
Also, the default maxium size for event files is only 20 MB for the classic event logs (`Security`, `System`, `Application`), 15 MB for PowrShell and a mere 1 MB for almost all of the other logs so there is a good chance that evidence is overwritten over time.
Simple PowerShell and Batch scripts in this repository have been provided to let systems administrators easily configure their Windows machines so that they will have the logs that they need when an incident occurs. For large networks, you probably want to use this document as a reference and configure your endpoints with Group Policy and/or InTune.

# Warning: Use at your own risk!

I do not take any responsibility for any adverse effects of enabling too much logging or the accuracy of anything in this repository.
It is your responsibility to understand and test out any changes you make to your systems on test machines before rolling out to production.
I recommend turning on as much logging as possible on test machines that mimic your environment for at least a week and then confirm if there are any events that are generating too much noise or if there are events that you want but are not being generated.

You can view the total number and percent of Event IDs in an `evtx` file with [Hayabusa](https://github.com/Yamato-Security/hayabusa)'s event ID metrics command.

Example: `hayabusa.exe -M -f path/to/Security.evtx`

# Important Windows event logs

* The most important event log to turn on is probably `Process Creation` which tracks what processes are run on a system.
  Currently, about half of [Sigma](https://github.com/SigmaHQ/sigma)'s detection rules rely on this event.
  This can be accomplished by installing Sysmon (Event ID 1) or enabling built-in logs (Security Event ID 4688).
  `Sysmon 1` will provide detailed information such as hashes and metadata of the executable so is ideal but in the case that Sysmon cannot be installed it is possible to turn on with Windows built-in functionality. However, it is important that command line logging is also enabled as many detection rules rely on this. Unfortunately `Security 4688` does not provide as detailed information as the Sysmon process creation logs.
* The second most important event log is a properly tuned Security log.
* The third most important are probably PowerShell Module logging and ScriptBlock logging as attackers will often abuse PowerShell.
* The forth are probably all of the other Sysmon events.
* After these, there are many other logs under the "Application and Services Logs" folder that are also very important: AppLocker, Bits-Client, NTLM, PowerShell, PrintService, Security-Mitigations, Windows Defender, Windows Firewall With Advanced Security, WMI-Activity, etc...

## Sigma's top log sources

![WindowsEventsWithSigmaRules](WindowsEventsWithSigmaRules.png)

Approximately less than 20% of sigma rules can be used with the default Windows audit settings!

### Top sigma log sources

![SigmaTopLogSources](SigmaTopLogSources.png)

### Top Security Event IDs

![TopSecurityEventIDs](TopSecurityEventIDs.png)

# Increasing the maximum file size

## Option 1: Manually through Event Viewer

This is not practical to do at scale, but the easiest way to enable/disable logs and check and/or configure their maximum file size is by right-clicking on the log in Event Viewer and opening up `Properties`.

## Option 2: Windows built-in tool

You can use the built-in `wevtutil` command.

Example: `wevtutil sl Security /ms:1073741824` to increase the maximum file size for the Security log to 1 GB.

## Option 3: PowerShell

## Option 4: Group Policy

It is straightforward to increase the maximum file size for the classic event logs such as `Security`, `System`, and `Application`, however, unfortunately you need to install Administratvie Templates and/or directly modify the registry in order to change the maximum file size for the other logs. It may just be easier to increase the file size with a `.bat` script on startup.

# Configuration script

A script to increase the maximum file size and enable the proper logs has been provided here: [YamatoSecurityConfigureWinEventLogs.bat](YamatoSecurityConfigureWinEventLogs.bat)

# Configuring log settings

## Sysmon log (1382 sigma rules)

File: `Microsoft-Windows-Sysmon%4Operational.evtx`

Default settings: `Not installed`

Installing and configuring sysmon is the single best thing you can do to increase your visibility on Windows endpoints but it will require planning, testing and maintenance. This is a big topic in itself so it is out of scope of this document at the moment. Please check out the following resources:
* TrustedSec Sysmon Community Guide: [https://github.com/trustedsec/SysmonCommunityGuide](https://github.com/trustedsec/SysmonCommunityGuide)
* Sysmon Modular: [https://github.com/olafhartong/sysmon-modular](https://github.com/olafhartong/sysmon-modular)
* Florian Roth's updated fork of the Swift On Security's sysmon config file: [https://github.com/Neo23x0/sysmon-config](https://github.com/Neo23x0/sysmon-config)
* Ion-storms' updated fork of the Swift On Security's sysmon config file: [https://github.com/ion-storm/sysmon-config](https://github.com/ion-storm/sysmon-config)

## Security log (1045 sigma rules (903 process creation rules + 142 other rules))

File: `Security.evtx`

Default settings: `Partially enabled`

The Security log is the most complex to configure so I have created a seperate document for it: [ConfiguringSecurityLogAuditPolicies.md](ConfiguringSecurityLogAuditPolicies.md)

## Powershell logs (175 sigma rules)

File: `Microsoft-Windows-PowerShell%4Operational.evtx`

### Module logging (30 sigma rules)

Turning on module logging will enable event ID `4103`. 
Module logging has the advantage that it can run on older OSes and versions of PowerShell: PowerShell 3.0 (Win 7+).
Another benefit is that it logs both the PowerShell command executed as well as the results.
The disadvantage is that it will create an extremely high number of events.
For example, if an attacker runs Mimikatz, it will create 7 MB of logs with over 2000 events! 

#### Enabling module logging

Default settings: `No Auditing`

##### Option 1: Enabling through group policy
In the Group Policy editor, open `Computer Configuration\Administrative Templates\Windows Components\Windows PowerShell` and enable `Turn on Module Logging`.
In the `Options` pane, click the `Show...` button to configure what modules to log.
Enter `*` in the `Value` textbox to record all modules.

##### Option 2: Enabling through the registry
```
HKLM\SOFTWARE\Wow6432Node\Policies\Microsoft\Windows\PowerShell\ModuleLogging → EnableModuleLogging = 1
HKLM\SOFTWARE\Wow6432Node\Policies\Microsoft\Windows\PowerShell\ModuleLogging \ModuleNames → * = *
```

### Script Block Logging (134 sigma rules)

Default settings: `On Win 10+, if a PowerShell script is flagged as suspicious by AMSI, it will be logged with a level of Warning.`

Turning on Script Block logging will enable event ID `4104` as well as `4105` and `4106` if you enable `Log script block invocation start / stop events`, however, it is not recommended to enable the script block invocation start and stop events. 
It is supported by default in PowerShell 5.0+ (Win 10+), however you can enable this on older OSes (Win 7+) if you install .NET 4.5 and WMF 4.0+.
Unfortunately, the maximum size of a single Windows event log is 32 KB so any PowerShell scripts greater than this will be fragmented in 32 KB sized blocks.
If you have the original PowerShell Operational `.evtx` file, you can use the [block-parser](https://github.com/matthewdunwoody/block-parser) tool to un-fragment these logs into a single easily readable text file.
One good thing about Script Block logging is that even if a malicious script is obfuscated with XOR, Base 64, ROT13, etc... the decoded script will be logged making analysis much easier.
The logs are more reasonable to work with than module logging as if an attacker runs Mimikatz, only 5 MB and 100 events will generated compared to the 7 MB and over 2000 events.
However, the output of the commands are not recorded with Script Block logging.

#### Enabling Script Block logging

#### Option 1: Enabling through group policy
In the Group Policy editor, open `Computer Configuration\Administrative Templates\Windows Components\Windows PowerShell` and enable `Turn on PowerShell Script Block Logging`.

#### Option 2: Enabling through the registry
`HKLM\SOFTWARE\Wow6432Node\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging → EnableScriptBlockLogging = 1`

### Transcription logging

Default settings: `No Auditing`

It is possible to also save PowerShell logs to text files on the local computer with transcription logs.
While an attacker can usually easily delete the transcription logs for anti-forensics, there may be scenarios where the attacker clears all of the event logs but does not search for transcription logs to delete. 
Therefore, it is recommended to also enable transcription logs if possible.
Ideally, transcript logs should be saved to a write-only network file share, however, this may be difficult to implement in practice.
Another benefit of transcription logs is they include the timestamp and metadata for each command and are very stroage efficient with less than 6 KB for Mimikatz execution. By default, they are saved to the user's documents folder. The downside is that the transcription logs only record what appears in the PowerShell terminal.

#### Enabling Transcription logging

##### Option 1: Enabling through group policy
In the Group Policy editor, open `Computer Configuration\Administrative Templates\Windows Components\Windows PowerShell` and enable `Turn on PowerShell Transcription`.
Then, specify the output directory.

##### Option 2: Enabling through the registry
```
HKLM\SOFTWARE\Wow6432Node\Policies\Microsoft\Windows\PowerShell\Transcription → EnableTranscripting = 1
HKLM\SOFTWARE\Wow6432Node\Policies\Microsoft\Windows\PowerShell\Transcription → EnableInvocationHeader = 1
HKLM\SOFTWARE\Wow6432Node\Policies\Microsoft\Windows\PowerShell\Transcription → OutputDirectory = “” (Enter path. Empty = default)
```

### References

* [Mandiant Blog: Greater Visibility Through PowerShell Logging](https://www.mandiant.com/resources/blog/greater-visibilityt)

## System log (55 sigma rules)

File: `System.evtx`

Default settings: `Enabled. 20 MB`

Malware will often install services for persistence, local privilege esclation, etc... which can be found in this log.
It is also possible to detect various vulnerabilities being exploited here.

## Application log (16 sigma rules)

File: `Application.evtx`

Default settings: `Enabled. 20 MB`

This log is mostly noise but you may be able to find some important evidence here.
One thing to be careful about is that different vendors will use the same event IDs for different events so you should also filter on not just Event IDs but Provider Names as well.

## Windows Defender Operational log (10 sigma rules)
 
File: `Microsoft-Windows-Windows Defender%4Operational.evtx`

Default settings: `Enabled. 1 MB`

You can detect not only Windows Defender alerts (which are important to monitor), but also exclusions being added, tamper protection being disabled, history deleted, etc...

## Bits-Client Operational log (6 sigma rules)
 
File: `Microsoft-Windows-Bits-Client%4Operational.evtx`

Default settings: `Enabled. 1 MB`

Bitsadmin.exe is a popular [lolbin](https://lolbas-project.github.io/lolbas/Binaries/Bitsadmin/) that attackers will abuse for downloading and executing malware.
You may find evidence of that in this log, although there will be a lot of false positives to watch out for.

## Firewall log (6 sigma rules)

File: `Microsoft-Windows-Windows Firewall With Advanced Security%4Firewall.evtx`

Default settings: `Enabled? 1 MB`

You can find evidence of firewall rules being added/modified/deleted here.
Malware will often add firewall rules to make sure they can communicate with their C2 server, add proxy rules for lateral movement, etc...

## NTLM Operational log (3 sigma rules)

File: `Microsoft-Windows-NTLM%4Operational.evtx`

Default settings: `Enabled but Auditing is disabled. 1 MB`

This log is recommended to enable if you want to disable NTLM authentication. 
Disabling NTLM will most likely break some communication, so you can monitor this log on the DCs and other servers to see who is still using NTLM and disable NTLM gradually starting with those users before disabling it globally.
It is possible to detect NTLM being used for incoming connections in logon events such as 4624 but you need to enable this log if you want to monitor who is making outgoing NTLM connections.

To enable auditing, in Group Policy open `Computer Configuration\Policies\Windows Settings\Security Settings\Local Policies\Security Options` and configure the proper various `Network security: Restrict NTLM:` settings.

Reference: [Farewell NTLM](https://www.scip.ch/en/?labs.20210909)

## Security-Mitigations KernelMode and UserMode logs  (2 sigma rules) 

Files: `Microsoft-Windows-Security-Mitigations%4KernelMode.evtx`, `Microsoft-Windows-Security-Mitigations%4UserMode.evtx`

Default settings: `Enabled. 1 MB`

At the moment there are only 2 sigma rules for these logs but you should probably be collecting and monitoring all of the Exploit Protection, Network Protection, Controlled Folder Access and Attack Surface Reduction logs (About 40+ Event IDs). 

Unfortunately the Attack Surface Reduction logs (previously WDEG(Windows Defender Exploit Guard) and EMET) are spread across multiple logs and require complex XML queries to search them.

Details: [https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/overview-attack-surface-reduction?view=o365-worldwide](https://learn.microsoft.com/en-us/microsoft-365/security/defender-endpoint/overview-attack-surface-reduction?view=o365-worldwide)

## PrintService logs (2 sigma rules)

It is recommended to enable the Operational log as well to detect Print Spooler attackers. (Ex: PrintNightmare, etc...)

### Admin (1 sigma rule)

File: `Microsoft-Windows-PrintService%4Admin.evtx`

Default settings: `Enabled. 1 MB`

### Operational (1 sigma rule)

File: `Microsoft-Windows-PrintService%4Operational.evtx`

Default settings: `Disabled. 1 MB`

## SMBClient Security log (2 sigma rules) 

File: `Microsoft-Windows-SmbClient%4Security.evtx`

Default settings: `Enabled. 8 MB`

Used to attempt to detect PrintNightmare (Suspicious Rejected SMB Guest Logon From IP) and users mounting hidden shares.

## AppLocker logs (1 sigma rule) 

Files: `Microsoft-Windows-AppLocker%4MSI and Script.evtx`, `Microsoft-Windows-AppLocker%4EXE and DLL.evtx`, `Microsoft-Windows-AppLocker%4Packaged app-Deployment.evtx`, `Microsoft-Windows-AppLocker%4Packaged app-Execution.evtx`

Default settings: `Enabled if AppLocker is enabled? 1 MB`

This is important to make sure is enabled and monitored if you are using AppLocker.

## CodeIntegrity Operational log (1 sigma rule)

File: `Microsoft-Windows-CodeIntegrity%4Operational.evtx`

Default settings: `Enabled. 1 MB`

Check this log to detect driver load events that get blocked by Windows code integrity checks, which may indicate a malicious driver that faild to load.

## Diagnosis-Scripted Operational log (1 sigma rule) 

Files: `Microsoft-Windows-Diagnosis-Scripted%4Operational.evtx`

Default settings: `Enabled. 1 MB`

Evidence of diagcab packages being used for exploitation may be found here.

## DriverFrameworks-UserMode Operational log  (1 sigma rule) 

Files: `Microsoft-Windows-DriverFrameworks-UserMode%4Operational.evtx`

Default settings: `No Auditing. 1 MB`

Detects plugged in USB devices.

## WMI-Activity Operational log  (1 sigma rule) 

File: `Microsoft-Windows-WMI-Activity%4Operational.evtx`

Default settings: `Enabled on Win10+. 1 MB`

This is important to monitor as attackers will often exploit WMI for persistence and lateral movement.

## TerminalServices-LocalSessionManager Operational log  (1 sigma rule) 

File: `Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx`

Default settings: `Enabled. 1 MB`

Detects cases in which ngrok, a reverse proxy tool, forwards events to the local RDP port, which could be a sign of malicious behaviour

## TaskScheduler Operational log  (1 sigma rule) 

File: `Microsoft-Windows-TaskScheduler%4Operational.evtx`

Default settings: `Disabled. 1 MB`

Attackers will often abuse tasks for persistence and lateral movement so this should be enabled.