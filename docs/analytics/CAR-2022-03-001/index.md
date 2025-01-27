---
title: "CAR-2022-03-001: Disable Windows Event Logging"
layout: analytic
submission_date: 2022/03/14
information_domain: Host
subtypes: Process
analytic_type: TTP
contributors: Lucas Heiligenstein
applicable_platforms: Windows
---


Adversaries may disable Windows event logging to limit data that can be leveraged for detections and audits. Windows event logs record user and system activity such as login attempts, process creation, and much more. This data is used by security tools and analysts to generate detections. There are different ways to perform this attack.
1. The first one is to create the Registry Key `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\MiniNt`. This action will not generate Security EventLog 4657 or Sysmon EventLog 13 because the value of the key remains empty. However, if an attacker uses powershell to perform this attack (and not cmd), a Security EventLog 4663 will be generated (but 4663 generates a lot of noise).
2. The second way is to disable the service EventLog (display name Windows Event Log). After disabed, attacker must reboot the system. The action of disabling or put in manual the service will modify the Registry Key value `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\EventLog\start`, therefore Security EventLog 4657 or Sysmon EventLog 13 will be generated on the system.
3. The third way is linked with the second. By default, the EventLog service cannot be stopped. If an attacker tries to stop the service, this one will restart immediately. Why ? Because to stop completely, this service must stop others, one in particular called netprofm (display name Network List Service). This service remains running until it is disabled. So Attacker must either disable EventLog and after to stop it or disable netprofm and after stop EventLog. Only stopping the service (even as admin) will not have an effect on the EventLog service because of the link with netprofm. Security EventLog 1100 will log the stop of the EventLog service (but also generates a lot of noise because it will generate a log everytime the system shutdown).
4. The fourth way is to use auditpol.exe to modify the audit configuration and disable/modify important parameters that will lead to disable the creation of EventLog.
5. The last one is to modify the Registry Key value `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\EventLog\Security\file` (or other kind of log) to modify the path where the EventLog are stocked. Importantly, with this technique, the EventViewer will use the value of the Registry Key "file" to know where to find the Log. Thus, using the EventViewer will always show the current event logs, but the old one will be stocked in another evtx. Also, the path must be in a folder that the Eventlog process has access (like it doesn’t work if attacker set up the new path in the Desktop). Attacker can also decrease the maxsize value of the Log to force the system to rewrite on the older EventLog (but the minimum cannot be less than 1028 KB). As the Registry key is modified, Security EventLog 4657 or Sysmon EventLog 13 will be generated on the system. All of these attacks required administrative right. Attacks number three, four and five do not require a system reboot to be effective immediately.

#### References
https://ptylu.github.io/content/report/report.html?report=25


### ATT&CK Detections

|Technique|Subtechnique(s)|Tactic(s)|Level of Coverage|
|---|---|---|---|
|[Impair Defenses](https://attack.mitre.org/techniques/T1562/)|[Disable Windows Event Logging](https://attack.mitre.org/techniques/T1562/002/)|[Defense Evasion](https://attack.mitre.org/tactics/TA0005/)|Moderate|


### D3FEND Techniques

|ID|Name|
|---|---| 
|D3-PSA | [Process Spawn Analysis](https://d3fend.mitre.org/technique/d3f:ProcessSpawnAnalysis)| 



### Data Model References

|Object|Action|Field|
|---|---|---|
|[registry](/data_model/registry) | [value_edit](/data_model/registry#value_edit) | [value](/data_model/registry#value) |
|[process](/data_model/process) | [create](/data_model/process#create) | [command_line](/data_model/process#command_line) |



### Implementations

#### Detection of Disable Windows Event Logging (Pseudocode)


This detects the disabling of Windows Event Logging, via process command line or registry key value manipulation.


```
processes = search Process:create
susp_processes = filter processes where ((command_line CONTAINS("*New-Item*") OR command_line CONTAINS("*reg add*")) OR command_line CONTAINS("*MiniNt*")) OR (command_line CONTAINS("*Stop-Service*")AND command_line CONTAINS("*EventLog*")) OR (command_line CONTAINS("*EventLog*") AND (command_line CONTAINS("*Set-Service*") OR command_line CONTAINS("*reg add*") OR command_line CONTAINS("*Set-ItemProperty*") OR command_line CONTAINS("*New-ItemProperty*") OR command_line CONTAINS("*sc config*"))) OR (command_line CONTAINS("*auditpol*") AND (command_line CONTAINS("*/set*") OR command_line CONTAINS("*/clear*") OR command_line CONTAINS("*/revove*"))) OR ((command_line CONTAINS("*wevtutil*") AND (command_line CONTAINS("*sl*") OR command_line CONTAINS("*set-log*"))))

reg_keys = search Registry:value_edit
event_log_reg_keys = filter reg_keys where Key="*EventLog*" AND (value="Start" OR value="File" OR value="MaxSize")
output susp_processes, event_log_reg_keys
```


#### Detection of Disable Windows Event Logging (Splunk)


Splunk version of the CAR pseudocode.


```
((EventCode="4688" OR EventCode="1") ((CommandLine="*New-Item*" OR CommandLine="*reg add*") CommandLine="*MiniNt*")OR (CommandLine="*Stop-Service*" CommandLine="*EventLog*")OR (CommandLine="*EventLog*" (CommandLine="*Set-Service*" OR CommandLine="*reg add*" OR CommandLine="*Set-ItemProperty*" OR CommandLine="*New-ItemProperty*" OR CommandLine="*sc config*")) OR (CommandLine="*auditpol*" (CommandLine="*/set*" OR CommandLine="*/clear*" OR CommandLine="*/revove*")) OR ((CommandLine="*wevtutil*" (CommandLine="*sl*" OR CommandLine="*set-log*")))) OR (EventCode="4719") OR ((EventCode="4657" OR EventCode="13") (ObjectName="*EventLog*") (ObjectValueName="Start" OR ObjectValueName="File" OR ObjectValueName="MaxSize"))
```


#### Detection of Disable Windows Event Logging (Logpoint)


LogPoint version of the CAR pseudocode.


```
((((((EventCode IN ["4688", "1"] CommandLine="*New-Item*" CommandLine="*reg add*" CommandLine IN "*MiniNt*") OR (CommandLine="*Stop-Service*" CommandLine="*EventLog*")) OR (CommandLine IN ["*Set-Service*", "*reg add*", "*Set-ItemProperty*", "*New-ItemProperty*", "*sc config*"] CommandLine IN "*EventLog*")) OR (CommandLine IN "*auditpol*" CommandLine IN ["*/set*", "*/clear*", "*/revove*"])) OR (CommandLine IN "*wevtutil*" CommandLine IN ["*sl*", "*set-log*"]) OR EventCode IN "4719") OR (EventCode IN ["4657", "13"] ObjectName IN "*EventLog*" ObjectValueName IN ["Start", "File", "MaxSize"]))
```



### Unit Tests

#### Test Case 1

MiniNt Registry Key creation with cmd.

```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\MiniNt"
```

#### Test Case 2

MiniNt Registry Key creation with powershell.

```
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\MiniNt"
```

#### Test Case 3

Disable EvenLog Service with Set-Service.

```
Set-Service -Name EventLog -StartupType Disabled
```

#### Test Case 4

Registry Key modification to disable EventLog Service.

```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\EventLog" /v start /t REG_DWORD /d 0x00000004 /f
```

#### Test Case 5

Stop EventLog Service with Stop-Service.

```
Stop-Service -Name EventLog -Force
```

#### Test Case 6

Audit configuration modification to disable EventLog with auditpol.

```
auditpol.exe /set /subcategory:"Process Creation" /success:Disable /failure:Disable
```

#### Test Case 7

Modification of Security EventLog path with wevtutil.

```
wevtutil.exe sl Security /logfilename:"C:\Windows\System32\winevt\Not-Important-Log.evtx"
```


