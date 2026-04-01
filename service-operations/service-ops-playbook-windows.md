---
title: "Windows Service Operations Playbook"
subtitle: "Procedures for maintaining scored service uptime, troubleshooting failures, and recovering from misconfigurations on Windows systems."
date: today
format:
  pdf:
    toc: true
    toc-depth: 3
    number-sections: true
    colorlinks: true
    fontsize: 11pt
    geometry:
      - margin=1in
    include-in-header:
      text: |
        \usepackage{fvextra}
        \DefineVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}
        \usepackage{xurl}
        \usepackage{etoolbox}
        \renewcommand{\arraystretch}{1.3}
        \AtBeginEnvironment{longtable}{\footnotesize\setlength{\tabcolsep}{3pt}}
tbl-colwidths: auto
---

## Backup & Restore

### Configuration & Registry Backup

**Before editing any configuration or registry key, always make a backup first.**

Registry backup:

```powershell
# Export specific hives
reg export HKLM\SYSTEM C:\Backups\system-hive.reg
reg export HKLM\SOFTWARE C:\Backups\software-hive.reg
reg export "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion" C:\Backups\winnt-version.reg

# Export a specific key
reg export "HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>" C:\Backups\svc-backup.reg
```

Configuration file backup:

```powershell
# Create backup directory
New-Item -ItemType Directory -Force -Path C:\Backups

# Copy IIS config
Copy-Item C:\Windows\System32\inetsrv\config\applicationHost.config C:\Backups\applicationHost.config.bak

# Copy DNS zones
Copy-Item C:\Windows\System32\dns -Recurse -Destination C:\Backups\dns-backup

# Copy any service config directory
Copy-Item "C:\path\to\config" -Recurse -Destination "C:\Backups\config-backup-$(Get-Date -Format yyyyMMdd-HHmm)"
```

System state backup (includes AD, registry, boot files, COM+):

```powershell
wbadmin start systemstatebackup -backuptarget:C:\Backups\ -quiet
```

### Database Backup & Restore

**SQL Server:**

```powershell
# Backup a database (T-SQL via sqlcmd)
sqlcmd -S localhost -Q "BACKUP DATABASE [<dbname>] TO DISK = 'C:\Backups\<dbname>.bak' WITH FORMAT"

# Backup all databases
sqlcmd -S localhost -Q "EXEC sp_MSforeachdb 'IF ''?'' NOT IN (''tempdb'') BACKUP DATABASE [?] TO DISK = ''C:\Backups\?.bak'' WITH FORMAT'"

# Restore a database
sqlcmd -S localhost -Q "RESTORE DATABASE [<dbname>] FROM DISK = 'C:\Backups\<dbname>.bak' WITH REPLACE"
```

If sqlcmd is not available, use SQL Server Management Studio (SSMS) or PowerShell:

```powershell
# Using the SqlServer PowerShell module
Backup-SqlDatabase -ServerInstance localhost -Database "<dbname>" -BackupFile "C:\Backups\<dbname>.bak"
Restore-SqlDatabase -ServerInstance localhost -Database "<dbname>" -BackupFile "C:\Backups\<dbname>.bak" -ReplaceDatabase
```

### Rollback Procedures

Registry restore:

```powershell
reg import C:\Backups\system-hive.reg
# Restart the affected service or reboot if necessary
```

IIS config restore:

```powershell
Copy-Item C:\Backups\applicationHost.config.bak C:\Windows\System32\inetsrv\config\applicationHost.config -Force
iisreset
```

General file restore:

```powershell
Copy-Item "C:\Backups\config-backup-YYYYMMDD-HHMM\*" "C:\path\to\config\" -Recurse -Force
Restart-Service <ServiceName>
```

**Verification after any rollback:**

```powershell
Get-Service <ServiceName>
# Check scoring dashboard to confirm service passes
```

## Scored Service Verification & Troubleshooting

### RDP (Remote Desktop)

**How the scorer tests it:** TCP connection to port 3389. If the port accepts connections, the check passes.

**Verify it yourself:**

```powershell
# Check RDP service is running
Get-Service TermService

# Check it's listening on 3389
netstat -ano | findstr :3389

# Check RDP is enabled in registry
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name fDenyTSConnections
# Should be 0 (not denied)

# Test from another host
Test-NetConnection -ComputerName <ip> -Port 3389
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Port 3389 not listening | RDP disabled | Set `fDenyTSConnections` to 0 at `HKLM:\...\Terminal Server`, then `Restart-Service TermService` |
| Port 3389 not listening | TermService stopped | `Start-Service TermService` and set startup to Automatic |
| Connection timeout | Firewall blocking | `netsh advfirewall firewall add rule name="RDP" protocol=TCP dir=in localport=3389 action=allow` |
| NLA authentication error | NLA required, client unsupported | Set `UserAuthentication` to 0 at `HKLM:\...\WinStations\RDP-Tcp` |
| Login fails | User not in RDP Users group | `net localgroup "Remote Desktop Users" <username> /add` |

### IIS (HTTP / HTTPS)

**How the scorer tests it:** HTTP GET to port 80 or 443, checks for specific content on the page.

**Verify it yourself:**

```powershell
# Check IIS service
Get-Service W3SVC

# Check it's listening
netstat -ano | findstr ":80 "
netstat -ano | findstr ":443 "

# Check site content
Invoke-WebRequest -Uri http://localhost/ -UseBasicParsing | Select-Object -ExpandProperty Content | Select-String "expected text"

# List sites and their status
Get-IISSite
# Or using appcmd:
C:\Windows\System32\inetsrv\appcmd list site
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | W3SVC not running | `Start-Service W3SVC` and set startup to Automatic |
| Connection refused | IIS not installed | `Install-WindowsFeature Web-Server -IncludeManagementTools` |
| 403 Forbidden | Default document missing | Add `index.html` to site root, or use `appcmd set config /section:defaultDocument` |
| 404 Not Found | Wrong physical path | `Get-IISSite` to check, fix physical path |
| 500 Error | App pool crashed | `appcmd start apppool /apppool.name:DefaultAppPool` |
| 500 Error | Missing ASP.NET | `Install-WindowsFeature Web-Asp-Net45` |
| Wrong content | Wrong site on port 80 | `Get-IISSite` to check bindings, fix conflicts |
| HTTPS not working | Missing SSL cert/binding | Check port 443 bindings, verify cert in Local Machine store |
| Firewall blocking | Windows Firewall | Allow ports 80 and 443: `netsh advfirewall firewall add rule name="HTTP" protocol=TCP dir=in localport=80 action=allow` |

**Site content location:** Check the physical path in IIS Manager or:

```powershell
(Get-IISSite -Name "Default Web Site").Applications["/"].VirtualDirectories["/"].PhysicalPath
```

Default: `C:\inetpub\wwwroot\`

**Restart IIS:** `iisreset` or `Restart-Service W3SVC`

### DNS (Windows DNS Server)

**How the scorer tests it:** Sends a DNS query to the server for a specific record. If the correct answer is returned, the check passes.

**Verify it yourself:**

```powershell
# Check DNS service
Get-Service DNS

# Test a query
nslookup <domain> 127.0.0.1
Resolve-DnsName -Name <domain> -Server 127.0.0.1

# List all zones
Get-DnsServerZone

# List records in a zone
Get-DnsServerResourceRecord -ZoneName <zone>
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | DNS service stopped | `Start-Service DNS` and set startup to Automatic |
| REFUSED response | Recursion disabled or zone missing | `Get-DnsServerRecursion` to check; enable if needed; verify zone exists |
| SERVFAIL | Forwarder unreachable or zone corrupt | `Set-DnsServerForwarder -IPAddress @()` to remove bad forwarders |
| Wrong answer | Incorrect DNS record | `Set-DnsServerResourceRecord` or remove and re-add the record |
| Timeout | Firewall blocking port 53 | Allow UDP and TCP 53: `netsh advfirewall firewall add rule name="DNS" protocol=UDP dir=in localport=53 action=allow` |

**Restart:** `Restart-Service DNS`

### SMTP (Exchange / hMailServer)

**How the scorer tests it:** Connects to port 25 and attempts to send a test email through the SMTP conversation.

**Verify it yourself:**

```powershell
# Check service
Get-Service MSExchangeTransport   # Exchange
Get-Service hMailServer            # hMailServer

# Check it's listening
netstat -ano | findstr :25

# Test SMTP conversation
telnet localhost 25
# EHLO test
# MAIL FROM:<test@test.com>
# RCPT TO:<user@domain.com>
# DATA
# Subject: test
#
# test body
# .
# QUIT
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | SMTP service stopped | Start the relevant service |
| 550 relay not permitted | Relay restrictions | Configure accepted domains and relay permissions |
| 550 user unknown | Mailbox doesn't exist | Create the mailbox/account in the mail server admin interface |
| Timeout | Firewall blocking port 25 | `netsh advfirewall firewall add rule name="SMTP" protocol=TCP dir=in localport=25 action=allow` |

### Active Directory / Domain Services

**How the scorer tests it:** May test domain authentication (Kerberos on port 88), LDAP queries (port 389/636), or DNS resolution of AD records.

**Verify it yourself:**

```powershell
# Check AD DS service
Get-Service NTDS

# Check critical AD ports are listening
netstat -ano | findstr "88 389 636 3268 445"

# AD health check
dcdiag /v
# Focus on: Connectivity, Advertising, FrsEvent, DFSREvent, SysVolCheck, KccEvent, Replications

# Replication status
repadmin /replsummary
repadmin /showrepl

# Check SYSVOL share is accessible
net share | findstr SYSVOL
dir \\localhost\SYSVOL

# Verify NETLOGON share
net share | findstr NETLOGON
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| NTDS service stopped | AD DS crashed | `Start-Service NTDS` -- if it won't start, boot into DSRM and check event logs |
| dcdiag fails Advertising | DC not advertising properly | `nltest /dsregdns` to re-register AD DNS records |
| Replication failing | Network issue or time skew | Check connectivity between DCs; sync time: `w32tm /resync /force` |
| SYSVOL not shared | DFS Replication issue | `Restart-Service DFSR`; check `dfsrdiag pollad` |
| Kerberos failures | Time skew > 5 minutes | Sync time: `w32tm /resync /force`; verify NTP source: `w32tm /query /status` |
| DNS records missing | AD DNS zones not loading | `Restart-Service DNS`; check AD-integrated zone in DNS Manager |
| Firewall blocking | Multiple ports needed | Allow ports 88, 389, 636, 445, 3268, 53 inbound |

**Critical:** AD is a dependency for all domain-joined Windows hosts. If AD is down, expect authentication failures across the environment. Prioritize AD restoration.

### SQL Server

**How the scorer tests it:** Connects to port 1433 with valid credentials, may query specific data.

**Verify it yourself:**

```powershell
# Check service
Get-Service MSSQLSERVER     # Default instance
Get-Service "MSSQL`$*"      # Named instances

# Check it's listening
netstat -ano | findstr :1433

# Test login
sqlcmd -S localhost -U <user> -P <password> -Q "SELECT name FROM sys.databases"
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | SQL Server stopped | `Start-Service MSSQLSERVER` |
| Connection refused | Not listening on TCP | Open SQL Server Config Manager, enable TCP/IP, restart |
| Login failed | Wrong credentials | `sqlcmd -S localhost -Q "ALTER LOGIN [<user>] WITH PASSWORD = '<newpass>'"` |
| Login failed | SQL auth disabled | Enable mixed mode via `xp_instance_regwrite` (set LoginMode to 2), then restart |
| Database missing | Dropped or detached | Restore from backup or reattach |
| Firewall blocking | Port 1433 not open | `netsh advfirewall firewall add rule name="SQL" protocol=TCP dir=in localport=1433 action=allow` |

**Restart:** `Restart-Service MSSQLSERVER`

### DHCP

**How the scorer tests it:** A client requests an IP via DHCP. If a valid lease is offered, the check passes.

**Verify it yourself:**

```powershell
# Check service
Get-Service DHCPServer

# Check scopes and leases
Get-DhcpServerv4Scope
Get-DhcpServerv4Lease -ScopeId <scope>

# Verify DHCP is authorized in AD (required for domain-joined DHCP servers)
Get-DhcpServerInDC
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| No leases offered | DHCP service stopped | `Start-Service DHCPServer` |
| No leases offered | Scope not activated | `Set-DhcpServerv4Scope -ScopeId <scope> -State Active` |
| No leases offered | No available addresses | Check scope range and exclusions: `Get-DhcpServerv4Scope`, expand range or clear stale leases |
| DHCP not authorized | Not authorized in AD | `Add-DhcpServerInDC -DnsName <server-fqdn> -IPAddress <server-ip>` |
| Wrong options | Gateway/DNS wrong | `Set-DhcpServerv4OptionValue -ScopeId <scope> -Router <gateway> -DnsServer <dns-ip>` |
| Firewall blocking | UDP 67/68 blocked | `netsh advfirewall firewall add rule name="DHCP" protocol=UDP dir=in localport=67 action=allow` |

**Restart:** `Restart-Service DHCPServer`

### SMB / File Shares

**How the scorer tests it:** Connects to port 445, authenticates, and accesses a specific share.

**Verify it yourself:**

```powershell
# Check service
Get-Service LanmanServer

# List shares
Get-SmbShare
net share

# Test access from another host
net use \\<ip>\<sharename> /user:<domain\user> <password>
dir \\<ip>\<sharename>
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | Server service stopped | `Start-Service LanmanServer` |
| Access denied | Wrong permissions | `Grant-SmbShareAccess -Name <share> -AccountName <user> -AccessRight Full -Force` |
| Access denied | NTFS permissions too restrictive | `icacls <path> /grant <user>:(OI)(CI)F` |
| Share not found | Share deleted or not created | `New-SmbShare -Name <sharename> -Path <path> -FullAccess Everyone` |
| Firewall blocking | Port 445 blocked | `netsh advfirewall firewall add rule name="SMB" protocol=TCP dir=in localport=445 action=allow` |

**Restart:** `Restart-Service LanmanServer`

## Windows Server Role Management

### Identifying Installed Roles & Features

```powershell
# List all installed roles and features
Get-WindowsFeature | Where-Object {$_.Installed}

# Short format -- just names
Get-WindowsFeature | Where-Object {$_.Installed} | Select-Object -ExpandProperty Name

# Check if a specific role is installed
Get-WindowsFeature -Name Web-Server
Get-WindowsFeature -Name AD-Domain-Services
Get-WindowsFeature -Name DNS
Get-WindowsFeature -Name DHCP
```

### Role-Specific Troubleshooting

**AD DS (Active Directory Domain Services):**

```powershell
dcdiag /v                           # Comprehensive AD health check
repadmin /replsummary               # Replication health
nltest /dsgetdc:<domain>            # Verify DC is locatable
Get-ADDomainController -Filter *    # List all DCs
```

**DNS:**

```powershell
Get-DnsServerZone                   # List all zones
Get-DnsServerDiagnostics            # Check DNS diagnostics
Clear-DnsServerCache                # Clear resolver cache if serving stale records
```

**DHCP:**

```powershell
Get-DhcpServerv4Scope              # List scopes
Get-DhcpServerv4ScopeStatistics    # Usage stats (how many leases available)
Get-DhcpServerv4Failover           # Check failover relationships
```

**IIS:**

```powershell
Get-IISSite                         # List sites
Get-IISAppPool                      # List app pools
C:\Windows\System32\inetsrv\appcmd list wp    # List running worker processes
```

## Service Dependency Mapping

Windows services have deep interdependencies, especially in AD environments.

### AD Dependency Chain

```
Active Directory Domain Services (NTDS)
  +-- DNS Server (AD-integrated zones)
  |     +-- All domain members (name resolution)
  +-- Kerberos (port 88)
  |     +-- All domain authentication
  +-- Group Policy
  |     +-- All domain member configurations
  +-- SYSVOL / NETLOGON shares
        +-- Login scripts, GPO files
```

### Service Dependencies Quick Reference

| Service | Depends On | Impact if Dependency Fails |
|------------|----------------------|---------------------------------------------|
| All domain-joined hosts | AD DS (DC) | Cannot authenticate, GPO won't apply |
| AD DS | DNS | AD replication fails, clients can't find DCs |
| IIS web apps | SQL Server | Application errors, no data |
| IIS web apps | DNS | Name resolution for backend connections fails |
| DHCP clients | DHCP Server | No IP = no connectivity |
| Exchange/mail | AD DS, DNS | Cannot authenticate users, cannot resolve MX records |
| File share clients | SMB (LanmanServer) | Cannot access shared files |

### Check Service Dependencies

```powershell
# Show what a service depends on
Get-Service <ServiceName> | Select-Object -ExpandProperty DependentServices
Get-Service <ServiceName> | Select-Object -ExpandProperty ServicesDependedOn

# Show dependency tree for a service
sc qc <ServiceName>
```

### Priority Order for Restoration

When multiple services are down, restore in this order:

1. **Network connectivity** (NICs, routing, firewall rules)
2. **Active Directory / Domain Controller** (NTDS, then DNS on the DC)
3. **DNS** (all name resolution depends on it)
4. **DHCP** (if clients use dynamic IPs)
5. **Database servers** (SQL Server -- web apps depend on them)
6. **Individual scored services** (IIS, RDP, SMTP, file shares)

## Change Management Discipline

### Copy Before Edit

**Before any configuration change:**

```powershell
# Registry backup
reg export "HKLM\SYSTEM\CurrentControlSet\Services\<ServiceName>" C:\Backups\svc-pre-change.reg

# Config file backup
Copy-Item <config-file> "<config-file>.bak.$(Get-Date -Format yyyyMMdd-HHmm)"

# IIS config backup
Copy-Item C:\Windows\System32\inetsrv\config\applicationHost.config "C:\Backups\applicationHost-$(Get-Date -Format yyyyMMdd-HHmm).config"
```

### Test Before Commit

| Service | Config Test Method |
|------------|---------------------------------------------|
| IIS | `C:\Windows\System32\inetsrv\appcmd verify config` (limited), test with browser/curl after restart |
| DNS | `Get-DnsServerZone` to verify zones load, `nslookup` to test resolution |
| AD | `dcdiag` after changes |
| Group Policy | `gpresult /r` on a client to verify policy applies |
| SQL Server | Test login after changes: `sqlcmd -S localhost -Q "SELECT 1"` |

### Verify After Every Change

After modifying any service configuration:

```powershell
# 1. Confirm the service is running
Get-Service <ServiceName>

# 2. Confirm it's listening on the expected port
netstat -ano | findstr :<port>

# 3. Perform the same check the scorer does
#    RDP: Test-NetConnection -ComputerName <ip> -Port 3389
#    HTTP: Invoke-WebRequest -Uri http://<ip>/ -UseBasicParsing
#    DNS: Resolve-DnsName -Name <record> -Server <ip>
#    SMTP: telnet <ip> 25
#    SQL: sqlcmd -S <ip> -U <user> -P <pass> -Q "SELECT 1"
#    SMB: net use \\<ip>\<share> /user:<user> <pass>

# 4. Check the scoring dashboard
```

### One Change at a Time

- Make one change, restart the affected service, verify scoring.
- If scoring breaks, revert using your backup and investigate.
- Do not stack multiple changes before testing.
