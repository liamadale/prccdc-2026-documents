---
title: "Rsyslog Client Setup -- Windows"
subtitle: "Forwarding Windows Event Logs to Centralized Rsyslog Server over TLS"
format:
  pdf:
    toc: true
    toc-depth: 3
    pdf-engine: pdflatex
    include-in-header:
      text: |
        \usepackage{fvextra}
        \DefineVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}
        \usepackage{xurl}
---

# Overview

This playbook configures a Windows host to forward Event Log entries (Security, System, Application) to the centralized rsyslog server over TLS using NXLog Community Edition. The client only needs the CA certificate -- no client certificate is required.

**What gets forwarded:** Windows Event Log entries from the Security, System, and Application channels. This includes logon events, service start/stop, policy changes, application errors, and anything else Windows logs to these channels.

**Prerequisites:**

- The rsyslog server is deployed and listening on TCP 6514 (see Rsyslog Server Setup playbook)
- You have the CA certificate file (`ca.crt`) from the server at `/etc/rsyslog-server/tls/ca/ca.crt`
- You know the rsyslog server's IP address
- Administrator access on the Windows host

---

# Step 1: Install NXLog Community Edition

Download NXLog CE from: https://nxlog.co/products/nxlog-community-edition/download

Run the installer with default settings. NXLog installs to `C:\Program Files\nxlog\`.

**Verify installation:**

```powershell
Get-Service nxlog
Test-Path "C:\Program Files\nxlog\nxlog.exe"
```

---

# Step 2: Transfer CA Certificate

Create the certificate directory and copy the CA cert from the rsyslog server:

```powershell
New-Item -Path "C:\Program Files\nxlog\cert" -ItemType Directory -Force
```

Transfer `ca.crt` from the rsyslog server to `C:\Program Files\nxlog\cert\ca.crt`. Methods:

- **SCP from server:** `scp /etc/rsyslog-server/tls/ca/ca.crt user@windows-host:"C:\Program Files\nxlog\cert\ca.crt"`
- **WinSCP:** Connect to the rsyslog server, download `/etc/rsyslog-server/tls/ca/ca.crt`
- **Network share:** Place `ca.crt` on a shared drive and copy it
- **USB:** Copy to USB from server, transfer to Windows host

**Verify the file is in place:**

```powershell
Get-Item "C:\Program Files\nxlog\cert\ca.crt"
```

---

# Step 3: Configure NXLog

Edit `C:\Program Files\nxlog\conf\nxlog.conf` and replace its contents with:

```
## NXLog TLS Syslog Configuration

define ROOT C:\Program Files\nxlog
define CERTDIR %ROOT%\cert
define SYSLOG_SERVER SERVER_IP_HERE

Moduledir %ROOT%\modules
CacheDir %ROOT%\data
Pidfile %ROOT%\data\nxlog.pid
SpoolDir %ROOT%\data
LogFile %ROOT%\data\nxlog.log

<Extension _syslog>
    Module      xm_syslog
</Extension>

<Input eventlog>
    Module      im_msvistalog
    Query       <QueryList>\
                    <Query Id="0">\
                        <Select Path="Security">*</Select>\
                        <Select Path="System">*</Select>\
                        <Select Path="Application">*</Select>\
                    </Query>\
                </QueryList>
</Input>

<Output syslog_tls>
    Module      om_ssl
    Host        %SYSLOG_SERVER%
    Port        6514
    CAFile      %CERTDIR%\ca.crt
    AllowUntrusted FALSE
    Exec        to_syslog_bsd();
</Output>

<Route eventlog_to_syslog>
    Path        eventlog => syslog_tls
</Route>
```

**Replace `SERVER_IP_HERE`** on the `define SYSLOG_SERVER` line with your rsyslog server's IP address (e.g., `10.0.0.44`).

**Configuration details:**

| Parameter | Value | Purpose |
|---|---|---|
| `im_msvistalog` | Input module | Reads Windows Event Log (Vista+ API) |
| `om_ssl` | Output module | Sends logs over TLS |
| `CAFile` | `ca.crt` path | Verify server's TLS certificate |
| `AllowUntrusted` | FALSE | Reject connections if server cert is invalid |
| `to_syslog_bsd()` | Format function | Converts events to BSD syslog format for rsyslog |

## Adding More Event Log Channels

To forward additional channels (e.g., PowerShell, Sysmon), add more `<Select>` lines inside the query:

```
<Select Path="Microsoft-Windows-PowerShell/Operational">*</Select>
<Select Path="Microsoft-Windows-Sysmon/Operational">*</Select>
```

## Filtering Events

To reduce noise, you can filter by Event ID. For example, to only forward logon events (4624, 4625) from Security:

```
<Select Path="Security">*[System[(EventID=4624 or EventID=4625)]]</Select>
```

---

# Step 4: Start NXLog Service

```powershell
Restart-Service nxlog
Get-Service nxlog
```

**Expected output:** Status should be `Running`.

Set NXLog to start automatically on boot (installer usually does this, but verify):

```powershell
Set-Service nxlog -StartupType Automatic
```

---

# Testing

## Generate a Test Event

```powershell
Write-EventLog -LogName Application -Source "Application" `
    -EventId 9999 -Message "TLS syslog test from $env:COMPUTERNAME"
```

## Check NXLog Log

```powershell
Get-Content "C:\Program Files\nxlog\data\nxlog.log" -Tail 30
```

Look for connection success messages or errors related to TLS/SSL.

## Verify on Server

On the rsyslog server:

```bash
# Check for the Windows hostname directory
sudo ls -la /var/log/remote/

# Search for the test message
sudo grep -r "9999" /var/log/remote/
sudo grep -r "TLS syslog test" /var/log/remote/
```

---

# Troubleshooting

## Common Issues

| Symptom | Likely Cause | Solution |
|---|---|---|
| NXLog service won't start | Config syntax error | Check `nxlog.log`, run `nxlog.exe -v` to validate config |
| `SSL error` in nxlog.log | CA cert mismatch or missing | Verify `ca.crt` is at the path in `CAFile`, re-copy from server |
| `connection refused` in nxlog.log | Firewall blocking 6514 | Check Windows Firewall outbound and server-side firewall |
| No logs on server | NXLog not running or wrong server IP | Verify `Get-Service nxlog`, check `SYSLOG_SERVER` value |
| `AllowUntrusted` warning | Set to TRUE | Change to FALSE and ensure correct CA cert |
| Events missing from a channel | Channel not in Query | Add the missing `<Select Path="...">` line |

## Validate NXLog Configuration

```powershell
& "C:\Program Files\nxlog\nxlog.exe" -v
```

This checks config syntax without starting the service.

## Check Windows Firewall

Ensure outbound TCP 6514 is not blocked:

```powershell
# Check if there's a blocking rule
Get-NetFirewallRule -Direction Outbound | Where-Object {
    $_.Enabled -eq 'True' -and $_.Action -eq 'Block'
} | Get-NetFirewallPortFilter | Where-Object {
    $_.RemotePort -eq '6514'
}

# If needed, add an allow rule
New-NetFirewallRule -DisplayName "Allow Rsyslog TLS Out" `
    -Direction Outbound -Protocol TCP -RemotePort 6514 -Action Allow
```

## Test Network Connectivity

```powershell
# Basic TCP connection test
Test-NetConnection -ComputerName SERVER_IP_HERE -Port 6514
```

## Enable Verbose Logging

Edit `nxlog.conf` and add near the top:

```
LogLevel INFO
```

Restart the service and check `nxlog.log` for detailed connection information.

---

# Quick Reference

## Files on Windows Client

```
C:\Program Files\nxlog\
+-- conf\
|   +-- nxlog.conf          <-- Main configuration
+-- cert\
|   +-- ca.crt              <-- CA certificate (from server)
+-- data\
|   +-- nxlog.log           <-- NXLog service log (check for errors)
+-- modules\                <-- NXLog modules
+-- nxlog.exe               <-- NXLog binary
```

## Essential Commands

```powershell
# Service management
Restart-Service nxlog
Get-Service nxlog
Stop-Service nxlog
Start-Service nxlog

# Check NXLog logs
Get-Content "C:\Program Files\nxlog\data\nxlog.log" -Tail 30

# Validate config
& "C:\Program Files\nxlog\nxlog.exe" -v

# Generate test event
Write-EventLog -LogName Application -Source "Application" `
    -EventId 9999 -Message "Test event"

# Test network connectivity
Test-NetConnection -ComputerName SERVER_IP -Port 6514
```

## Event Log Channels Worth Forwarding

| Channel | What It Contains |
|---|---|
| Security | Logon/logoff, privilege use, policy changes, object access |
| System | Service start/stop, driver failures, time changes |
| Application | Application errors, warnings, informational events |
| Microsoft-Windows-PowerShell/Operational | PowerShell script execution (if enabled) |
| Microsoft-Windows-Sysmon/Operational | Process creation, network connections (if Sysmon installed) |
| Microsoft-Windows-Windows Defender/Operational | Defender detections and scan results |
