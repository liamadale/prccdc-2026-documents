---
title: "Windows Incident Response & Intrusion Detection Playbook"
format:
  pdf:
    toc: true
    toc-depth: 3
    pdf-engine: pdflatex
    geometry:
      - margin=0.75in
    fontsize: 9pt
    include-in-header:
      text: |
        \usepackage{fvextra}
        \DefineVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}
        \usepackage{xurl}
        \usepackage{enumitem}
        \setlist[description]{style=nextline,leftmargin=0pt,labelindent=0pt,itemsep=2pt,parsep=0pt}
        \usepackage{longtable}
        \usepackage{booktabs}
---

# Windows Incident Response & Intrusion Detection Playbook

This guide details methods for identifying and responding to unauthorized access on Windows systems. It covers detection of persistence mechanisms, log analysis, immediate containment, and post-incident recovery. It follows the NIST SP 800-61 incident handling lifecycle: Identification, Containment, Eradication, and Recovery.

---

## Phase 1: Identification

Detecting intrusion through logs, processes, and persistence mechanisms.

### 1. Service Hijacking & Persistence

Attackers often install services or scheduled tasks to maintain access even after a reboot.

**Key Event Log Entries (System Log):**

| Event ID | Meaning |
|---|---|
| 7045 | A new service was installed (critical indicator of service-based persistence) |
| 4698 | A scheduled task was created |

**Using Sysinternals Autoruns:**

Autoruns (https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns) is the most comprehensive tool for finding persistence. Run it as Administrator and check the following tabs:

1. Open Autoruns and select **Options > Scan Options > Check VirusTotal.com**.
2. Review the **Services** tab. Look for:
   - Entries with "File not found" in the Image Path column.
   - Unsigned binaries (no publisher, or publisher marked in red).
   - Services pointing to binaries in user-writable directories (`C:\Users\`, `C:\Temp\`, `C:\ProgramData\`).
3. Review the **Scheduled Tasks** tab for the same indicators.
4. Review the **Logon** tab for unexpected startup entries.

**PowerShell -- List Auto-Start Services:**

```powershell
# List all services configured for automatic start, showing name, path, and start mode
Get-WmiObject Win32_Service |
  Where-Object { $_.StartMode -eq 'Auto' } |
  Select-Object Name, DisplayName, PathName, StartMode |
  Format-Table -AutoSize
```

Review the `PathName` column. Legitimate services typically reside in `C:\Windows\System32\` or `C:\Program Files\`. Services running from user directories or `C:\Temp` are suspicious.

**PowerShell -- List Scheduled Tasks:**

```powershell
# List all scheduled tasks with their actions and triggers
Get-ScheduledTask | Where-Object { $_.State -ne 'Disabled' } |
  ForEach-Object {
    $actions = ($_ | Get-ScheduledTaskInfo -ErrorAction SilentlyContinue)
    [PSCustomObject]@{
      TaskName = $_.TaskName
      TaskPath = $_.TaskPath
      State    = $_.State
      Action   = ($_.Actions | ForEach-Object { $_.Execute + " " + $_.Arguments }) -join "; "
    }
  } | Format-Table -AutoSize
```

Look for tasks that execute PowerShell, cmd, or scripts from temporary or user directories.

---

### 2. Registry & WMI Persistence

Attackers frequently write to auto-run registry keys or create WMI event subscriptions that survive reboots.

**Check Common Auto-Run Registry Keys:**

```powershell
# Per-machine auto-run (applies to all users)
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -ErrorAction SilentlyContinue
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce" -ErrorAction SilentlyContinue

# Per-user auto-run (applies to the current user)
Get-ItemProperty -Path "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -ErrorAction SilentlyContinue
Get-ItemProperty -Path "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce" -ErrorAction SilentlyContinue
```

Any entry pointing to a script, encoded command, or binary in an unusual path is suspicious.

**Check for WMI Event Subscriptions:**

WMI subscriptions are a stealthy persistence mechanism that can trigger actions on system events (e.g., at logon or on a timer).

```powershell
# List WMI event filter bindings (the persistence trigger)
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding

# List the event filters (what triggers the action)
Get-WMIObject -Namespace root\Subscription -Class __EventFilter

# List the event consumers (what action is taken)
Get-WMIObject -Namespace root\Subscription -Class CommandLineEventConsumer
```

On a clean system, these queries typically return no results. Any output should be investigated.

---

### 3. RDP & Login Activity

RDP is one of the most common attack vectors. You must distinguish between local interactive logins (Type 2), network logins (Type 3), and remote desktop logins (Type 10).

**Key Event Log Entries (Security Log):**

| Event ID | Meaning |
|---|---|
| 4624 | Successful logon. Check the `Logon Type` field: Type 10 = Remote Desktop. |
| 4625 | Failed logon. High volume from a single source indicates brute force. |
| 4778 | Session reconnected (attacker returning to an existing session). |
| 4720 | New user account created (check if the attacker created a backdoor account). |

: {tbl-colwidths="[15,85]"}

**Terminal Services Logs:**

Navigate in Event Viewer to: `Applications and Services Logs > Microsoft > Windows > TerminalServices-LocalSessionManager > Operational`

| Event ID | Meaning |
|---|---|
| 21 | Session logon succeeded |
| 24 | Session disconnected |
| 25 | Session reconnection succeeded |

**View Active RDP/Login Sessions:**

```powershell
qwinsta
```

This displays all sessions, their state (Active, Disconnected), and the owning username. Any session you do not recognize requires investigation.

**Query RDP Logins (Last 3 Months):**

```powershell
$since = (Get-Date).AddMonths(-3)
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4624; StartTime=$since} -ErrorAction SilentlyContinue |
  Where-Object { $_.Properties[8].Value -eq 10 } |
  ForEach-Object {
    [PSCustomObject]@{
      Time     = $_.TimeCreated
      User     = $_.Properties[5].Value
      SourceIP = $_.Properties[18].Value
    }
  } | Sort-Object Time -Descending | Format-Table -AutoSize
```

**Query All User Logins (Excludes System/Service Accounts):**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4624,4625} -MaxEvents 200 |
  Where-Object {
    $_.Properties[5].Value -notmatch 'SYSTEM|LOCAL SERVICE|NETWORK SERVICE|\$$' -and
    $_.Properties[8].Value -in @(2,7,10,11)
  } |
  ForEach-Object {
    [PSCustomObject]@{
      Time      = $_.TimeCreated
      Status    = if ($_.Id -eq 4625) { "FAILED" } else { "Success" }
      User      = $_.Properties[5].Value
      LogonType = switch ($_.Properties[8].Value) {
        2  { "Interactive" }
        7  { "Unlock" }
        10 { "RDP" }
        11 { "CachedInteractive" }
      }
      SourceIP  = $_.Properties[18].Value
    }
  } | Format-Table -AutoSize
```

**Query Failed Login Attempts (Brute Force Detection):**

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4625} -MaxEvents 50 |
  ForEach-Object {
    $props = $_.Properties
    [PSCustomObject]@{
      Time      = $_.TimeCreated
      User      = $props[5].Value
      SourceIP  = if ($props.Count -gt 19) { $props[19].Value } else { "N/A" }
      LogonType = if ($props.Count -gt 10 -and $null -ne $props[10]) {
        switch ($props[10].Value) {
          2  { "Interactive" }
          10 { "RDP" }
          default { $props[10].Value }
        }
      } else { "N/A" }
    }
  } | Format-Table -AutoSize
```

Note the source IPs with the most failed attempts. These are your brute force sources and should be blocked in Phase 2.

**Check and Increase Log Retention:**

```powershell
# Check the oldest event in the Security log to determine current retention
Get-WinEvent -LogName Security -Oldest -MaxEvents 1 | Select-Object TimeCreated

# Increase the Security log maximum size to 1 GB (approximately 3 months of retention)
wevtutil sl Security /ms:1073741824
```

---

### 4. Anomalous Processes

Malware often masquerades as legitimate system processes. Knowing what is normal helps you spot what is not.

**Visual Inspection with Task Manager or Process Explorer:**

Key things to look for:

- **Parent-Child Relationships:** `cmd.exe` or `powershell.exe` should not be spawned by `svchost.exe`, `w3wp.exe` (IIS), or `spoolsv.exe`. This pattern typically indicates exploitation.
- **Binary Paths:** System binaries run from `C:\Windows\System32\`. If you see `csrss.exe` or `svchost.exe` running from `C:\Users\`, `C:\Temp\`, or any non-system path, it is malicious.
- **Typosquatting:** Look for misspelled process names (e.g., `svhost.exe` instead of `svchost.exe`, `csrs.exe` instead of `csrss.exe`).

**Using Sysinternals Process Explorer:**

1. Download and run Process Explorer (https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer) as Administrator.
2. Select **Options > Verify Image Signatures**. This checks whether each running binary is signed by its claimed publisher.
3. Unsigned processes or those with invalid signatures are highlighted and should be investigated.
4. Right-click a suspect process and select **Properties > Image** tab to view the full binary path and command line.

**PowerShell -- List Processes with Binary Paths:**

```powershell
# List all running processes with their executable paths
Get-Process | Select-Object Id, ProcessName, Path | Sort-Object ProcessName | Format-Table -AutoSize
```

**PowerShell -- Find Processes Running from Unusual Locations:**

```powershell
# Flag processes running from user directories, temp folders, or downloads
Get-Process | Where-Object {
  $_.Path -and ($_.Path -match 'Users|Temp|Downloads|AppData|ProgramData')
} | Select-Object Id, ProcessName, Path | Format-Table -AutoSize
```

---

### 5. PowerShell Attack Detection

PowerShell is a primary tool for attackers due to its deep system access. Script block logging and module logging capture what was executed.

**Enable PowerShell Script Block Logging (if not already enabled):**

```powershell
# Create the registry key for script block logging
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Force
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name "EnableScriptBlockLogging" -Value 1
```

**Review PowerShell Script Block Logs:**

```powershell
# View the most recent PowerShell script block log entries
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" -MaxEvents 50 |
  Where-Object { $_.Id -eq 4104 } |
  ForEach-Object {
    [PSCustomObject]@{
      Time       = $_.TimeCreated
      ScriptBlock = $_.Properties[2].Value
    }
  } | Format-List
```

Look for:

- Base64-encoded commands (`-EncodedCommand`, `FromBase64String`)
- Download cradles (`Invoke-WebRequest`, `Net.WebClient`, `DownloadString`)
- Reflective loading (`[Reflection.Assembly]::Load`)
- Credential access (`Mimikatz`, `sekurlsa`, `Get-Credential`)
- AMSI bypass attempts (`AmsiUtils`, `amsiInitFailed`)

---

### 6. Additional Indicators of Compromise

**Check for Unauthorized Local Accounts:**

```powershell
# List all local user accounts
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordLastSet | Format-Table -AutoSize

# List members of the Administrators group
Get-LocalGroupMember -Group "Administrators" | Format-Table -AutoSize
```

Any account you do not recognize -- especially one in the Administrators group -- should be disabled immediately.

**Check for Unexpected Shared Folders:**

```powershell
# List all SMB shares
Get-SmbShare | Select-Object Name, Path, Description | Format-Table -AutoSize
```

Attackers may create hidden shares (names ending in `$`) to exfiltrate data or stage tools.

**Check the Hosts File for DNS Hijacking:**

```powershell
Get-Content C:\Windows\System32\drivers\etc\hosts
```

Attackers sometimes redirect security update domains or internal services to attacker-controlled IPs.

**Check for Proxy or Network Configuration Changes:**

```powershell
# Check system proxy settings
netsh winhttp show proxy

# Check Internet Explorer/Edge proxy settings (often inherited by applications)
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" |
  Select-Object ProxyServer, ProxyEnable
```

---

### 7. CVE Identification & Vulnerability Assessment

Knowing whether your Windows Server has known, exploitable vulnerabilities is critical. An unpatched system with a public CVE is not a hypothetical risk -- it is an active liability.

**Step 1: Determine Your Exact OS Build**

```powershell
# Get the OS name, version, and build number
[System.Environment]::OSVersion
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber
```

Example output: `Windows Server 2019`, Version `1809`, Build `17763.5328`. The build number is essential for determining which patches have been applied.

**Step 2: List All Installed Patches (Hotfixes)**

```powershell
# List installed updates sorted by installation date (newest first)
Get-HotFix | Sort-Object InstalledOn -Descending | Format-Table -AutoSize
```

Note the most recent KB number and its install date. If the latest patch is months old, the system is likely vulnerable to any CVEs disclosed since that date.

**Step 3: Check for Missing Updates**

```powershell
# Query Windows Update for available but not-yet-installed updates
$session = New-Object -ComObject Microsoft.Update.Session
$searcher = $session.CreateUpdateSearcher()
$results = $searcher.Search("IsInstalled=0")
$results.Updates | ForEach-Object {
  [PSCustomObject]@{
    Title    = $_.Title
    KB       = ($_.KBArticleIDs -join ", ")
    Severity = $_.MsrcSeverity
  }
} | Format-Table -AutoSize
```

Any update with a Severity of **Critical** or **Important** should be treated as a priority.

**Step 4: Cross-Reference Against Known Exploited CVEs**

CISA maintains the Known Exploited Vulnerabilities (KEV) catalog -- a list of CVEs that are actively being used in attacks. Cross-reference your OS version and missing KBs against this list.

- CISA KEV Catalog.\
  <https://www.cisa.gov/known-exploited-vulnerabilities-catalog>
- Microsoft Security Update Guide (search by product and date).\
  <https://msrc.microsoft.com/update-guide>

**Step 5: Check for Common High-Risk Windows Server CVEs**

The following are frequently targeted in competition and real-world environments. Verify your patch level covers them:

| CVE | Name / Description | Affected |
|---|---|---|
| CVE-2017-0144 | EternalBlue (SMBv1 remote code execution) | Server 2008, 2008 R2, 2012, 2012 R2, 2016 |
| CVE-2020-1472 | Zerologon (Netlogon privilege escalation) | All domain controllers |
| CVE-2021-34527 | PrintNightmare (Print Spooler RCE) | All Windows Server versions |
| CVE-2021-36942 | PetitPotam (NTLM relay via LSARPC) | All Windows Server versions |
| CVE-2022-26923 | Certifried (AD CS privilege escalation) | Domain controllers with AD CS |
| CVE-2023-23397 | Outlook NTLM relay (no user interaction) | Exchange environments |

: {tbl-colwidths="[20,40,40]"}

**Quick check -- is SMBv1 enabled (EternalBlue attack surface)?**

```powershell
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol
```

If this returns `True`, disable it immediately:

```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

**Quick check -- is the Print Spooler running (PrintNightmare attack surface)?**

```powershell
Get-Service -Name Spooler | Select-Object Name, Status, StartType
```

If the server is not actively used as a print server, disable it:

```powershell
Stop-Service -Name Spooler -Force
Set-Service -Name Spooler -StartupType Disabled
```

**Best Course of Action When Dealing with a Compromisable Host**

If you determine that a server has unpatched critical CVEs and may already be compromised:

1. **Do not patch and assume the problem is solved.** A patch closes the door but does not evict an attacker who already entered. Treat the system as potentially compromised and run through Phase 1 identification checks first.

2. **Prioritize by exploitability, not just severity.** A CVE with a public exploit or entry in the CISA KEV catalog is a higher priority than a theoretical vulnerability. Focus on those first.

3. **If patching immediately is not possible** (e.g., during a competition window or change freeze), apply mitigations:
   - Disable the vulnerable service if it is not business-critical (e.g., Print Spooler, SMBv1).
   - Add firewall rules to restrict access to the vulnerable port/service to only trusted IPs.
   - Increase monitoring on the affected service's event logs.

4. **If exploitation has already occurred**, proceed directly to Phase 2 (Containment) and Phase 3 (Eradication). Patch the vulnerability as part of the eradication process, not as a standalone fix.

5. **Document everything.** Record the CVE, the patch status, the mitigation applied, and the time. This is critical for inject responses in CCDC and for incident reports in production.

---

## Phase 2: Containment & Ejection

Immediate actions to remove the attacker and cut off access. Speed is critical -- an attacker who realizes they have been detected may attempt to escalate privileges, exfiltrate data, or destroy evidence.

### 1. Terminating Active RDP Sessions

If an attacker is currently logged in via RDP, forcibly terminate their session.

1. **List all sessions** to find the attacker's session ID:

    ```cmd
    qwinsta
    ```

    Example output:
    ```
    SESSIONNAME    USERNAME    ID  STATE   TYPE
    rdp-tcp#0      attacker    3   Active
    ```

2. **Kill the session** using the ID from the output above:

    ```cmd
    rwinsta 3
    ```

    Alternatively, use `logoff`:

    ```cmd
    logoff 3
    ```

3. **Verify** the session is terminated:

    ```cmd
    qwinsta
    ```

---

### 2. Terminating Remote Desktop Services Sessions

On Windows Server machines running Remote Desktop Services (Terminal Services), use the Terminal Services Manager for additional session management.

1. Open `tsadmin.msc` from the Run dialog (Windows Server 2008/2008 R2) or use **Remote Desktop Services Manager** on newer versions.
2. Alternatively, use PowerShell to query and manage RD sessions:

    ```powershell
    # List all RD sessions on the local server
    query session

    # Disconnect a specific session
    tsdiscon <SessionID>

    # Log off a specific session
    logoff <SessionID>
    ```

3. On Windows Server 2012 and later, use Server Manager:
   - Navigate to **Remote Desktop Services > Collections**.
   - Right-click the suspicious session and select **Log Off**.

---

### 3. Blocking an Attacker's IP

If you identified the attacker's IP from Event ID 4624 source network address, block it immediately.

**PowerShell (Windows Firewall with Advanced Security):**

```powershell
# Create an inbound block rule for the attacker's IP
New-NetFirewallRule -DisplayName "Block Attacker IP" -Direction Inbound -LocalPort Any -Protocol TCP -Action Block -RemoteAddress 192.168.1.50

# Verify the rule was created
Get-NetFirewallRule -DisplayName "Block Attacker IP" | Get-NetFirewallAddressFilter
```

**CMD (netsh -- Legacy Method):**

```cmd
netsh advfirewall firewall add rule name="Block Attacker IP" dir=in interface=any action=block remoteip=192.168.1.50/32
```

**Block Multiple IPs:**

```powershell
# Block a list of attacker IPs in a single rule
New-NetFirewallRule -DisplayName "Block Attacker IPs" -Direction Inbound -Action Block -RemoteAddress @("192.168.1.50", "10.0.0.25", "172.16.5.100")
```

::: {.callout-tip}
Also create outbound block rules if the attacker's IP is being used as a command-and-control callback destination:

```powershell
New-NetFirewallRule -DisplayName "Block C2 Outbound" -Direction Outbound -Action Block -RemoteAddress 192.168.1.50
```
:::

---

### 4. Account Remediation

**Disable the Compromised Account:**

```powershell
Disable-LocalUser -Name "compromised_user"
```

**Verify the Account is Disabled:**

```powershell
Get-LocalUser -Name "compromised_user" | Select-Object Name, Enabled
```

**Remove the Account from Privileged Groups:**

```powershell
Remove-LocalGroupMember -Group "Administrators" -Member "compromised_user"
Remove-LocalGroupMember -Group "Remote Desktop Users" -Member "compromised_user"
```

**Force a Password Reset (If Keeping the Account):**

```powershell
$newPassword = Read-Host -AsSecureString "Enter new password"
Set-LocalUser -Name "compromised_user" -Password $newPassword
```

---

### 5. Network Isolation (Nuclear Option)

If you cannot identify the scope of the compromise or suspect lateral movement, disconnect the machine from the network entirely.

**Software Method (Disable the Network Adapter):**

```powershell
# List all network adapters to identify the correct one
Get-NetAdapter | Select-Object Name, Status, InterfaceDescription | Format-Table -AutoSize

# Disable the adapter
Disable-NetAdapter -Name "Ethernet" -Confirm:$false
```

::: {.callout-warning}
This will sever your own remote session if you are connected via RDP. Only do this if you have out-of-band access (console, iLO/iDRAC, or physical access to the machine).
:::

**Physical/Virtual Method:**

- **Physical machine:** Unplug the Ethernet cable.
- **Virtual machine:** Disconnect the vNIC in the hypervisor management console (vSphere, Hyper-V Manager, etc.).

---

## Phase 3: Eradication & Recovery

After containment, remove all attacker artifacts and restore the system to a known-good state.

### 1. Removing Persistence Mechanisms

Systematically remove every persistence mechanism identified in Phase 1.

**Remove Malicious Services:**

```powershell
# Stop the service
Stop-Service -Name "MaliciousService" -Force

# Delete the service (PowerShell 6+ or use sc.exe on older systems)
Remove-Service -Name "MaliciousService"

# Legacy method (CMD)
sc.exe delete "MaliciousService"
```

**Remove Malicious Scheduled Tasks:**

```powershell
# Disable and then remove the task
Disable-ScheduledTask -TaskName "MaliciousTask" -TaskPath "\"
Unregister-ScheduledTask -TaskName "MaliciousTask" -TaskPath "\" -Confirm:$false
```

**Remove Malicious Registry Run Keys:**

```powershell
# Remove a specific value from the Run key
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "MaliciousEntry"
Remove-ItemProperty -Path "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "MaliciousEntry"
```

**Remove WMI Persistence:**

```powershell
# Remove WMI event subscriptions (filter, consumer, and binding)
Get-WMIObject -Namespace root\Subscription -Class __EventFilter | Where-Object { $_.Name -eq "MaliciousFilter" } | Remove-WMIObject
Get-WMIObject -Namespace root\Subscription -Class CommandLineEventConsumer | Where-Object { $_.Name -eq "MaliciousConsumer" } | Remove-WMIObject
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding | Where-Object { $_.Filter -match "MaliciousFilter" } | Remove-WMIObject
```

**Delete Attacker Tools and Payloads:**

```powershell
# Remove files identified during the investigation
Remove-Item -Path "C:\Temp\malware.exe" -Force
Remove-Item -Path "C:\Users\Public\payload.ps1" -Force
```

---

### 2. Credential Reset

Assume all credentials on the compromised system are exposed.

1. **Reset all local user passwords:**

    ```powershell
    $newPassword = Read-Host -AsSecureString "Enter new password"
    Set-LocalUser -Name "<username>" -Password $newPassword
    ```

2. **Reset the local Administrator password:**

    ```powershell
    $adminPassword = Read-Host -AsSecureString "Enter new Administrator password"
    Set-LocalUser -Name "Administrator" -Password $adminPassword
    ```

3. **If the machine is domain-joined**, coordinate with your domain administrator to:
   - Reset the compromised user's domain password.
   - Reset the computer account password (`Reset-ComputerMachinePassword`).
   - Review and reset any service account credentials that were stored on the machine.
   - Check for Golden Ticket / Silver Ticket attacks if domain controller compromise is suspected (reset the KRBTGT account twice).

4. **Rotate any API keys, tokens, or certificates** stored on the system.

---

### 3. System Hardening

After restoring the system, apply hardening measures to prevent recurrence.

**Restrict RDP Access:**

```powershell
# Allow RDP only from specific trusted subnets
New-NetFirewallRule -DisplayName "Allow RDP - Trusted Networks Only" -Direction Inbound -LocalPort 3389 -Protocol TCP -Action Allow -RemoteAddress @("10.0.0.0/24", "192.168.1.0/24")

# Block RDP from all other sources
New-NetFirewallRule -DisplayName "Block RDP - All Others" -Direction Inbound -LocalPort 3389 -Protocol TCP -Action Block
```

**Enable Network Level Authentication (NLA) for RDP:**

```powershell
# Require NLA before establishing an RDP connection
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "UserAuthentication" -Value 1
```

**Enable Audit Policies for Ongoing Monitoring:**

```powershell
# Enable auditing for logon events (success and failure)
auditpol /set /subcategory:"Logon" /success:enable /failure:enable

# Enable auditing for process creation
auditpol /set /subcategory:"Process Creation" /success:enable

# Enable auditing for account management events
auditpol /set /subcategory:"User Account Management" /success:enable /failure:enable
```

**Enable Command-Line Auditing in Process Creation Events:**

```powershell
# This causes Event ID 4688 to include the full command line of every new process
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" -Name "ProcessCreationIncludeCmdLine_Enabled" -Value 1 -Type DWord -Force
```

**Disable Unnecessary Services:**

```powershell
# Example: disable the Print Spooler if not needed (commonly exploited)
Stop-Service -Name "Spooler" -Force
Set-Service -Name "Spooler" -StartupType Disabled
```

---

## Resources & Documentation

**Event Log Analysis:**

- Microsoft Learn: Audit Logon Events -- Official documentation on interpreting 4624/4625 logs.\
  <https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/audit-logon-events>
- Ultimate Windows Security: Event ID Encyclopedia -- Comprehensive reference for all Windows Event IDs.\
  <https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/>

**Sysinternals Tools:**

- Sysinternals Suite -- Essential tools including Process Explorer, Autoruns, and Sysmon.\
  <https://learn.microsoft.com/en-us/sysinternals/>
- Autoruns -- Comprehensive persistence scanner.\
  <https://learn.microsoft.com/en-us/sysinternals/downloads/autoruns>
- Process Explorer -- Advanced process inspection with signature verification.\
  <https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer>

**Incident Response Frameworks:**

- NIST SP 800-61 Rev. 2 -- Computer Security Incident Handling Guide.\
  <https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final>
- SANS Incident Handler's Handbook.\
  <https://www.sans.org/white-papers/33901/>

**Video Walkthrough:**

- Windows Security Event Logs EXPLAINED! -- Practical walkthrough on navigating Event Viewer to find logon (4624) and failure (4625) events.\
  <https://www.youtube.com/watch?v=rmuQQoYsRO4>
