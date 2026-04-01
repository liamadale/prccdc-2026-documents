---
title: "First 15 Minutes Runbook"
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

# First 15 Minutes Runbook

Priority-ordered checklist for every team member the moment they sit down at their assigned system. This document tells you **what to do** and **which reference document to use**. Keep your printed binder open alongside this runbook.

## How Quotient Scoring Works

PRCCDC uses the **Quotient** scoring engine. Understanding how it works is critical to keeping your points.

- **Rounds:** The engine checks every scored service every ~60 seconds (with random jitter of 5-10 seconds, so you cannot predict exact timing).
- **Content checks, not just port checks.** Quotient does not just verify a port is open. It logs into SSH, queries DNS records and verifies the answers, fetches web pages and checks for specific content via regex, downloads files over FTP/SMB and verifies their contents, sends real emails through SMTP, and runs SQL queries against databases. If red team defaces your site or corrupts data, you lose points even if the service is running.
- **Credential rotation:** Each round, Quotient picks a **random credential** from its credlist for your team. If that credential is stale (you changed the password on the box but did not submit a PCR), the check fails.
- **SLA penalties:** If a service fails for 5 consecutive rounds (~5 minutes), Quotient applies a large penalty (typically 25 points) on top of the missed per-round points. Short outages are recoverable; long outages are devastating.
- **Scoring engine traffic:** Quotient runners connect from the scoring infrastructure and may rotate source IPs. Do **not** try to whitelist a single scoring IP -- whitelist the entire scoring subnet provided in the competition packet.

---

## Minutes 0-2: Identify Your System

**Goal:** Determine exactly what you are sitting in front of and record it on your inventory sheet.

### Tasks

1. **Identify the OS, hostname, IP addresses, gateway, and DNS.** Open the **Host Enumeration Cheatsheet** and run the commands under *System & OS* and *Network & Exposure* for your platform (Linux or Windows).
2. **Identify all listening services.** Use the *Network & Exposure* section of the **Host Enumeration Cheatsheet** to list listening ports and the processes behind them.
3. **Record everything immediately** on the **Host Inventory Sheet**. Fill in the *Core Identity* table and the *Network --- Interfaces & Addressing* table. Do not rely on memory.

::: {.callout-important}
Do not skip the inventory sheet. Injects will ask for this information and you will not have time to re-gather it later.
:::

---

## Minutes 2-5: Secure Access

**Goal:** Lock out attackers using default or known credentials. Red team will try default passwords within the first minutes.

### Tasks

1. **Change all passwords** -- root/Administrator first, then every human account. Use the team-agreed password scheme (16+ characters, mixed case/number/symbol). Refer to the **Linux Blue Team Cheatsheet** (*User Hardening* section) or the **Windows Blue Team Cheatsheet** (*User Hardening* section) for the commands.
2. **Submit Password Change Requests (PCRs) immediately.** After changing passwords on the box, log into the Quotient scoring dashboard and submit a PCR for every changed account. The engine randomly selects credentials from its credlist each round -- if your PCR is not submitted, checks will intermittently fail whenever it picks the stale password.

::: {.callout-warning}

## PCR Rules

- Only usernames that **already exist** in Quotient's credlist can be updated. Submitting a PCR for an unknown username is silently ignored.
- Change the password on the host **first**, then submit the PCR. If Quotient checks between those two steps, you lose one round -- but that is better than the reverse (PCR submitted, password not yet changed).
- Keep a written log of every password change and PCR submission on your **Host Inventory Sheet**.
:::

3. **Enumerate all accounts and privilege groups.** Use the *Users, Privilege, Auth* section of the **Host Enumeration Cheatsheet** to list users, check for UID-0 accounts (Linux), and review group memberships. Cross-reference against the credlist usernames shown in Quotient -- do **not** disable or delete accounts that appear in the credlist.
4. **Disable unknown or suspicious accounts** that are **not** in the Quotient credlist. If you find accounts you do not recognize, disable them immediately. Record what you found on the *Accounts* page of the **Host Inventory Sheet**.
5. **Lock down remote access (SSH/RDP).** Refer to the **Linux Blue Team Cheatsheet** (*User Hardening / SSH* section) or the **Windows Blue Team Cheatsheet** (*Remote Access* section) for hardening steps. Key items:
   - Linux: Disable root login over SSH, review `AllowUsers`/`AllowGroups`.
   - Windows: Enable NLA for RDP, restrict the Remote Desktop Users group.
   - Do **not** disable password authentication for SSH if SSH is scored -- Quotient authenticates with username/password from the credlist.
6. **Verify** you can still log in after making changes. Have a teammate test SSH or RDP to your host.
7. **Record** all credential changes and access configuration on the **Host Inventory Sheet** (*Access & Credentials* and *Remote Access Audit* tables).

---

## Minutes 5-8: Verify Scored Services

**Goal:** Confirm every scored service is functional. If a service is down, fixing it takes priority over hardening.

### Tasks

1. **Check the scoring sheet** (from team captain or competition packet) for which services are scored on your host.
2. **Test each scored service locally** the same way Quotient tests it. Quotient performs deep validation, not just port checks:

| Service    | Port(s) | How Quotient checks it                          |
|------------|---------|--------------------------------------------------|
| SSH        | 22      | Authenticates with credlist credentials; may run a command and validate output |
| RDP        | 3389    | TCP connection accepted                          |
| HTTP/HTTPS | 80/443  | Fetches specific URL path, checks status code (200) AND regex match on page content |
| DNS        | 53      | Queries specific A/MX records and verifies the answer matches expected values |
| SMTP       | 25      | Authenticates (if configured) and sends a real email through the server |
| FTP        | 21      | Authenticates, downloads a specific file, verifies content via regex or hash |
| SMB        | 445     | Authenticates via NTLM, mounts share, reads a specific file, verifies content |
| LDAP       | 636     | Binds (authenticates) using `username@domain` format |
| SQL        | 3306    | Authenticates, runs a query, validates the output |
| WinRM      | 5985    | Authenticates via NTLM, executes PowerShell commands, validates output |

::: {.callout-important}
Quotient checks **content**, not just connectivity. If red team defaces a web page, corrupts a database, deletes a file from an FTP/SMB share, or poisons a DNS record, the check fails even though the service is running. Protect the data, not just the process.
:::

3. **Record results** on the **Host Inventory Sheet** (*Listening Services* table): service name, port, status (UP/DOWN), and whether it is scored. Also note what content each service serves (web page paths, DNS records, shared files) so you can detect and restore tampering later.
4. **If a service is down**, troubleshoot it now. Refer to the **Service Operations Playbook** (Linux or Windows) for service-specific troubleshooting steps. Remember: 5 consecutive failed rounds triggers an SLA penalty. If you cannot fix it in 2 minutes, note the issue and move on -- return to it after minute 15.
5. **Check the Quotient dashboard** if `ShowDebugToBlueTeam` is enabled. Failed check debug output tells you exactly why a check failed (timeout, auth failure, wrong content, connection refused).

---

## Minutes 8-12: Snapshot & Backup

**Goal:** Create restore points for critical data and configs before hardening. If red team destroys something or your changes break a service, you need to roll back.

### Tasks

1. **Back up system configuration.** Archive `/etc/` (Linux) or export registry hives and firewall policy (Windows).
2. **Back up service data.** This includes:
   - Web content (web roots)
   - Database dumps (MySQL, PostgreSQL, MSSQL)
   - DNS zones (if DNS server)
   - Group Policy (if domain controller)
3. **Back up scheduled tasks and cron jobs.**
4. **Record backup locations** on the **Host Inventory Sheet** (*Security Posture --- Initial Hardening Checklist* and *Data --- Sensitive Data* tables).
5. **Verify at least one backup.** Spot-check that a backup file exists and is non-zero in size.

Refer to the **Service Operations Playbook** (Linux or Windows) for backup commands specific to each service type.

::: {.callout-tip}
Store backups in a consistent location across all hosts (e.g., `/root/backups` on Linux, `C:\Backups` on Windows) so any teammate can find them.
:::

---

## Minutes 12-15: Harden & Firewall

**Goal:** Reduce the attack surface. Enable the firewall with a default-deny policy and allow only scored service ports.

### Tasks

1. **Enable the host firewall with default-deny inbound.** Refer to the **Linux Blue Team Cheatsheet** (*iptables* or *firewalld* section) or the **Windows Blue Team Cheatsheet** (*Firewall* section) for the commands.
   - Allow loopback and established/related connections.
   - Allow ICMP (Quotient may use ping checks).
   - Allow only the ports your scored services need (from the list you built in Minutes 5-8).
   - **Allow the scoring engine subnet.** Get the Quotient runner subnet from the competition packet and ensure it is not blocked. Quotient may rotate source IPs across its runner pool (via its Divisor component), so whitelist the entire subnet, not a single IP.
2. **Record your firewall rules** on the **Host Inventory Sheet** (*Firewall Rules* table).
3. **Disable unnecessary services.** Stop and disable anything that is not scored and not needed for operations (telnet, remote registry, SMBv1, etc.). The blue team cheatsheets list common services to disable.
4. **Test all scored services again** after applying firewall rules. If anything broke, check that you allowed the correct port. Also check the Quotient dashboard -- if scores drop immediately after firewall changes, you are blocking the scoring engine.
5. **Mark the hardening checklist** on the **Host Inventory Sheet** (*Security Posture --- Initial Hardening Checklist*).

---

## After 15 Minutes: Inventory, Monitor, Defend

**Goal:** Complete your inventory, establish monitoring, and shift into ongoing defense.

### Complete the Inventory Sheet

Use the **Host Enumeration Cheatsheet** to fill any remaining gaps on the **Host Inventory Sheet**. Key sections that may still be incomplete:

- *Accounts* -- all users, privilege levels, disabled accounts
- *Services --- Inventory* -- every running service, not just scored ones
- *Scheduled Tasks / Cron / Startup* -- use the **Host Enumeration Cheatsheet** (*Services, Tasks, Logs* section)
- *Persistence --- Backdoor Hunt Checklist* -- refer to the **Linux Blue Team Cheatsheet** (*Persistence Hunting* section) or the **Windows Blue Team Cheatsheet** (*Persistence* section)
- *Data --- Sensitive Data / Databases* -- use the **Host Enumeration Cheatsheet** (*Data, Databases, Shares* section)

### Set Up Logging

If the team is deploying centralized logging, refer to the **Logging Playbooks**:

- Rsyslog server setup
- Rsyslog client (Linux)
- Rsyslog client (Windows)

### Begin Monitoring

Refer to the **Linux Blue Team Cheatsheet** (*Log Analysis* section) or **Windows Blue Team Cheatsheet** (*Event Log Analysis* section) for monitoring commands. Key areas to watch:

- Authentication logs (failed logins, new sessions)
- New network connections
- Process activity (unexpected processes, high CPU)
- Recently modified files in critical directories
- **Quotient dashboard** -- monitor your service check results in real time. If a service starts failing intermittently, it may indicate red team tampering with content (defaced pages, corrupted data, poisoned DNS) rather than a service outage.

### Communicate with Your Team

- Report your host status to the team captain: hostname, OS, scored services, anything broken or suspicious.
- If you found suspicious accounts, processes, or files, report them immediately.
- Ask if any injects have arrived that need information from your host.

---

## Decision Tree: What Do I Do Next?

```
START
  |
  +--> Is a scored service DOWN on your host?
  |      YES --> Service Operations Playbook (Linux or Windows)
  |       NO --> continue
  |
  +--> Do you see signs of compromise?
  |    (unknown processes, unauthorized accounts,
  |     modified files, suspicious connections)
  |      YES --> Incident Response Playbook
  |              - Linux: ir-playbook-linux
  |              - Windows: ir-playbook-windows
  |       NO --> continue
  |
  +--> Has an inject been assigned to your team?
  |      YES --> Inject Response Templates
  |       NO --> continue
  |
  +--> Is your inventory sheet complete?
  |      NO  --> Finish it using the Host Enumeration Cheatsheet
  |      YES --> continue
  |
  +--> All clear: Continue monitoring.
         - Watch logs and connections
         - Re-check scored services every few minutes
         - Hunt for persistence (use blue team cheatsheet)
         - Assist teammates who need help
```

---

## Quick Reference: Your Printed Documents

| Document | When to use it |
|----------|----------------|
| **Host Enumeration Cheatsheet** | Gathering system info (OS, network, users, services, data) |
| **Host Inventory Sheet** | Recording everything you find (one per host) |
| **Linux Blue Team Cheatsheet** | Hardening, iptables, user lockdown, persistence hunting, forensics |
| **Windows Blue Team Cheatsheet** | Hardening, firewall, user lockdown, persistence hunting, forensics |
| **Service Ops Playbook (Linux)** | Troubleshooting and restoring Linux services |
| **Service Ops Playbook (Windows)** | Troubleshooting and restoring Windows services |
| **IR Playbook (Linux)** | Responding to compromise on Linux hosts |
| **IR Playbook (Windows)** | Responding to compromise on Windows hosts |
| **Inject Response Templates** | Completing business-scenario tasks from white team |
| **Network Discovery Playbook** | Mapping the environment from scratch |
| **Logging Playbooks** | Setting up centralized log collection |
| **Sysadmin Cheatsheet** | General admin reference |
