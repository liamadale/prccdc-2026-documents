---
title: "CCDC Script Analysis Cheatsheet"
subtitle: "Quick Reference for Scripts from Partner Team Repositories"
format:
  pdf:
    toc: true
    toc-depth: 2
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
---

# Source Repositories

- **[GH]** GrayHats -- https://github.com/GrayHatsUWB/ccdc-2026
- **[UI]** UIdaho -- https://github.com/Braxton-Marlatt/PRCCDC-UIdaho-Scripts/tree/main
- **[OS]** OSUSec -- https://github.com/osusec/prccdc26

---

# First Response / Time Zero

**`Linux/asap.sh`** [GH]
: Changes root password, locks root, creates sudo user, hardens SSH, fixes key perms, runs inventory. All-in-one. `sudo bash asap.sh`

**`Linux/hardenScripts/firstrun.sh`** [GH]
: Installs essentials (net-tools, auditd, rsyslog), backs up passwd/network state, removes shell profiles, hardens all php.ini, restarts web servers. `sudo bash firstrun.sh`

**`Linux/main.sh`** [UI]
: Menu-driven hardening orchestrator. Run all modules or pick one. `bash main.sh --all` or `--module <name>` or `--list`

**`Windows/T0-Windows.ps1`** [OS]
: Disables SMBv1, creates local admin, resets AD Administrator password, disables PS v2. Admin. Interactive.

**`Windows/Auto-Patch-Exploits.ps1`** [GH]
: 5-phase: Zerologon KB patches, registry backup, Netlogon enforcement, SMB hardening (v1 off, WDigest off, LSA protection), disables Print Spooler/LLMNR/NetBIOS. Admin. Reboot required.

**`Windows/startup.ps1`** [UI]
: Two-liner: disables NetBIOS and IPv6. Admin. Instant effect.

**`Windows/Account-Cleanup.ps1`** [GH]
: Disables Guest, sets UAC to Always Notify, disables reversible password encryption. Admin.

---

# Password Rotation

**`Linux/utils/changeAllPasswords.sh`** [GH]
: Sets random 20-char passwords for ALL users in /etc/passwd. Outputs CSV. `sudo bash changeAllPasswords.sh > creds.csv`

**`Linux/utils/changePassword.sh`** [GH]
: Sets random 20-char password for one user. `sudo bash changePassword.sh <username>`

**`Linux/utils/credsChanger.sh`** [GH]
: Interactive: prompts for username/password pairs. Type "done" to finish. `sudo bash credsChanger.sh`

**`Linux/modules/rotatePasswords.sh`** [UI]
: Interactive per-user rotation via `passwd` prompt for all human users + root. Root required.

**`Windows/Change-Domain-User-Passwords.ps1`** [GH]
: Resets ALL domain users (except krbtgt, svc\_\*) with 24-char random passwords. Domain Admin. Outputs `DomainUsersPasswords.csv`.

**`Windows/updateADPassword.ps1`** [GH]
: Same goal via AD PowerShell module. 20-char passwords. Needs AD module + Domain Admin.

**`Windows/Reset-KrbTgt-Password-For-RWDCs-And-RODCs.ps1`** [UI]
: Enterprise krbtgt password reset with replication monitoring. Golden ticket remediation. DC only. Use test mode first.

---

# Inventory / Enumeration

**`Linux/inventory/inventory.sh`** [GH]
: Full system inventory: hostname, IP, OS, ports, Docker containers, users, services (~30 checked), K8s role, domain join. `sudo bash inventory.sh`

**`Linux/inventory/web.sh`** [GH]
: Detailed web/service enumeration with config details, container mounts, volumes per service. `sudo bash web.sh` (set `COLOR=1`)

**`Linux/inventory/bad.sh`** [GH]
: Security audit: SUID/SGID vs GTFOBins list, capabilities, world-writable files, sudoers, ld.so.preload. `sudo bash bad.sh`

**`Linux/inventory/findSudoUsers.sh`** [GH]
: Lists all users with root/sudo privileges from all sources. `sudo bash findSudoUsers.sh`

**`Linux/inventory/docker1.sh`** [GH]
: Finds docker-compose, Dockerfiles, .env on host and inside running containers. Needs Docker access.

**`Linux/inventory/kubectlInventory.sh`** [GH]
: Full K8s audit: nodes, pods, ingress, images, privileged pods, PVCs, events. Needs kubectl.

**`Linux/inventory/mysqlInventory.sh`** [GH]
: MySQL security audit: users, auth plugins, grants, databases, sizes, security vars. `bash mysqlInventory.sh <user> <pass> <host> <port>`

**`Linux/inventory/psqlInventory.sh`** [GH]
: PostgreSQL audit: users, superuser status, per-DB permissions, sizes. `bash psqlInventory.sh <user> <host> <port>`

**`Linux/inventory/svc_enum.sh`** [GH]
: Per-service config dump. Supports SSH, FTP, APACHE, NGINX, SMB. `bash svc_enum.sh APACHE`

**`Linux/inventory/baseq.sh`** [GH]
: One-liner: prints hostname + OS. `bash baseq.sh`

**`Linux/utils/baselineHelper.sh`** [GH]
: Massive enumeration dump: passwd, shadow, groups, sudoers, SSH, crons, services, SUID, world-writable, recent files, process tree. `sudo bash baselineHelper.sh`

**`Windows/enumeration.ps1`** [GH]
: Non-interactive baseline: system info, users, network, services, software, Defender, firewall, tasks. Saves to timestamped folder. Elevated.

**`Windows/InternalRecon.ps1`** [GH]
: Maps listening ports to processes, checks binary signatures, categorizes as Core/MS/Third-Party/Unknown. `.\InternalRecon.ps1 [-OutputFile scan.csv]`

**`Windows/InternalReconV2.ps1`** [GH]
: Enhanced: full scan or network-only mode, flags suspicious paths. `.\InternalReconV2.ps1 [-NetworkOnly] [-OutputFile out.csv]`

**`Windows/baseline.ps1`** [GH]
: Interactive 15-option menu: services, tasks, netstat, autoruns, local admins, AD admins. Elevated.

**`Windows/network_check.bat`** [GH]
: Shows connections, NetBIOS/SMB sessions, shares, firewall, NICs. Admin.

**`Windows/user_check.bat`** [GH]
: Account policies, user list, whoami /all, WMIC user/group info. Admin.

**`network-topology/ascii-top.py`** [GH]
: Generates ASCII network topology from Nmap XML. `python3 ascii-top.py scan.xml [router_ip] [--compact]`

---

# Linux Hardening

**`Linux/modules/sshConfig.sh`** [UI]
: Comprehensive SSH hardening: disables root login, strong ciphers/MACs/KEX, disables forwarding, sets banner, configures fail2ban. Validates with `sshd -t`, auto-rollback on failure. Root. Test SSH in new terminal after.

**`Linux/hardenScripts/ssh.sh`** [GH]
: Drop-in SSH config: disables pubkey, enables password auth, disables root, enables PAM. Root. Lighter version.

**`Linux/modules/firewallRules.sh`** [UI]
: Default-deny iptables: allows SSH in, DNS/HTTP/HTTPS out, loopback, established. Root. **Add scored service ports first.**

**`Linux/modules/hardenSysctl.sh`** [UI]
: 20+ kernel params: SYN cookies, ASLR, ptrace restrict, kptr\_restrict, disable SysRq, symlink protection. Root. Appends to sysctl.conf (duplicates on re-run).

**`Linux/modules/lockAccounts.sh`** [UI]
: Creates new admin, locks ALL other human accounts (passwd -l + chage -E 0). Root. Interactive. Shows reversal commands.

**`Linux/modules/cronControl.sh`** [UI]
: **Nuclear:** deletes ALL crontabs, truncates /etc/crontab, empties all cron.d directories. Root. **Destroys legitimate crons too.**

**`Linux/modules/configureAppArmor.sh`** [UI]
: Installs AppArmor, switches complain to enforce, loads extra profiles. Root. May need reboot.

**`Linux/modules/bulkDisableServices.sh`** [UI]
: Stops/disables ~30 risky services (xinetd, rsh, tftp, cups, NFS, etc.). Root. **Review list -- may break scored services.**

**`Linux/modules/removeUnusedPackages.sh`** [UI]
: Removes netcat, gcc, cmake, make, telnet. Root.

**`Linux/modules/patchPrivEsc.sh`** [UI]
: Patches PwnKit (CVE-2021-4034) and CVE-2023-32233. Root. Safe to repeat.

**`Linux/hardenScripts/ai.sh`** [GH]
: Disables unprivileged user namespaces (skips if containers detected). Root.

**`Linux/hardenScripts/php.sh`** [GH]
: Hardens all php.ini: disables exec/system/shell\_exec, uploads, URL includes. Root.

**`Linux/hardenScripts/key_perms.sh`** [GH]
: Sets 600 on all SSH private keys. Root.

**`linux/disableptrace.sh`** [OS]
: Sets ptrace\_scope=1 in sysctl. Root.

---

# Windows Hardening

**`Windows/Local-Hardening.ps1`** [UI]
: Massive non-DC hardening: disables users, creates competition accounts, firewall config, IPv6/NetBIOS off, EternalBlue patches, SMB upgrade, Mimikatz patch, unquoted service paths, sethc/Utilman backdoor removal, ASR rules, LSASS protection. ~1000 lines. Admin. Interactive.

**`Windows/dchardening.ps1`** [UI]
: DC-specific: backs up FW/DNS, exports group memberships, mass-disable AD users, create tiered admin accounts, GPO import, DCSync patch, Mimikatz patch, IIS hardening, auditing. ~1200 lines. Admin on DC. Interactive.

**`Windows/groupManagement.ps1`** [UI]
: Strips unauthorized members from all sensitive AD groups (DA, EA, Schema Admins, etc.). Removes anonymous access. DC. **Verify needed members first.**

**`Windows/advancedAuditing.ps1`** [UI]
: Enables full Windows audit policy via registry: logon, account mgmt, object access, process tracking, policy changes. Admin. Called by other scripts.

**`Windows/Auto-Patch-Zerologon-V2.ps1`** [GH]
: Focused Zerologon fix: KB patch, Netlogon enforcement, machine password reset, double krbtgt reset. Admin on DC. AD module required. Reboot.

**`Windows/Revert-Auto-Patch-Exploits.ps1`** [GH]
: Rollback for Auto-Patch: restores registry, re-enables Print Spooler, reverts NetBIOS. Admin. Reboot.

**`Windows/Audit-Policy.ps1`** [GH]
: Configures auditpol categories + command-line auditing in Event 4688. Admin.

**`Windows/ww-hardening.ps1`** [UI]
: Web server hardening + IIS lockdown + PII scanning + system enumeration + optional process/service monitors. Admin. `-RunProcessMonitor` / `-RunServiceMonitor`.

**`harden_wimdows.ps1`** [OS]
: Comprehensive T0: password roll, SMBv1 off, SMB signing, NTLMv2, LSA protection, WDigest off, Print Spooler off, Defender on, UAC, purge autoruns, full audit, firewall. Self-deletes. Admin. Set `$NEW_PASSWORD` and `$PUBKEY` first.

**`harden_essembee.ps1`** [OS]
: SMB hardening: disables SMBv1, enables signing, restricts anonymous. Self-deletes. Admin. `[-keepSMB]` to keep SMBv1.

---

# Monitoring / Detection

**`Windows/monitor.ps1`** [GH]
: Continuous 15s loop: watches Security log for process creation, service installs, privilege use, user changes, failed logons, lockouts. Also scores services for suspicion. Elevated. Runs forever. Logs to `SuspiciousEvents.log`.

**`Windows/userMonitor.ps1`** [GH]
: Real-time user event monitoring: detects new accounts, flags users with excessive suspicious activity. Elevated. Runs forever.

**`Windows/processMonitor.ps1`** [GH]
: Scores services by suspicion (unsigned, user-dir path, entropy, no description). 1s polling. Elevated. Runs forever.

**`Linux/utils/LSMS_Setup.sh`** [GH]
: Clones and initializes LSMS (Linux Security Monitoring System) from GitHub. Needs git + python3.

**`Linux/utils/LSMS_monitor.sh`** [GH]
: Runs the LSMS monitoring tool. Needs LSMS initialized first.

**`Linux/utils/addFileMonitoring.sh`** [GH]
: Adds a path to Wazuh real-time file integrity monitoring. `sudo bash addFileMonitoring.sh /etc/passwd`

---

# Backup / Recovery

**`Linux/utils/baseline.sh`** [GH]
: Captures ports, processes, archives /etc and /opt with SHA256 checksums, stores in disguised dir. Inits git repos in /etc and /opt. Root. Needs git. SCP backup off-host.

**`linux/backup_automation/initialstate.sh`** [OS]
: Quick first-state capture: compresses /home, snapshots passwd/group/services/modules/logins. Root. Run before any changes.

**`linux/backup_automation/backup_lin.sh`** [OS]
: Full backup: /home, /etc, /bin, /var/www/html as tarballs + system state snapshots. Root. Slow.

**`linux/backup_automation/disperse_bkup.sh`** [OS]
: Copies backup tarball to multiple hidden locations for redundancy. Customize destination paths first.

**`windows/backup_automation/backup_win.ps1`** [OS]
: Backs up scheduled tasks, firewall rules, services, AD groups, autoruns, GPOs, SYSVOL, WMI subs, network state. Zips to `C:\temp\`. Admin.

**`windows/backup_automation/disperse_bkup.ps1`** [OS]
: Copies backup zip to multiple inconspicuous directories for redundancy. Run after backup\_win.ps1.

---

# Account Lockdown

**`linux/backup_automation/lock_noninitial_users.sh`** [OS]
: Compares current passwd against initial snapshot, locks any new accounts (disable + deny-ssh group + nologin). `sudo bash lock_noninitial_users.sh /path/to/init_passwd admin`

**`windows/backup_automation/disable_noninitial_users.ps1`** [OS]
: Disables AD users not in CSV whitelist. Optional: move to OU, enforce group memberships. `.\disable_noninitial_users.ps1 -CsvPath whitelist.csv -AdminUser admin`

**`Linux/modules/lockAccounts.sh`** [UI]
: Creates new admin, locks all other human accounts. Root. Interactive.

**`Linux/modules/groupMemberships.sh`** [UI]
: Audits/removes members from sudo/wheel/adm groups. `--remove` for interactive cleanup.

---

# Security Auditing

**`Linux/modules/auditServices.sh`** [UI]
: Read-only audit: running services, enabled services, suspicious services list, listening ports, failed services. Saves report to `/tmp/`.

**`Linux/modules/auditStartupTasks.sh`** [UI]
: Audits persistence: systemd, rc.local, init.d, profile.d, user bashrc. Finds red team backdoors. Read-only.

**`Linux/modules/checkPackageIntegrity.sh`** [UI]
: Verifies package checksums to detect tampering. Multi-distro (debsums/rpm -Va/pacman/apk). Saves to `/tmp/`.

**`Linux/modules/checkPermissions.sh`** [UI]
: Fixes shadow/passwd perms, audits SUID, world-writable, capabilities, ACLs. Multi-distro. Root.

**`Linux/modules/permissionAudit.sh`** [UI]
: More thorough: also covers /etc/group, /etc/gshadow, SGID binaries. Root.

**`Linux/modules/searchSSN.sh`** [UI]
: Searches /home for SSN patterns (XXX-XX-XXXX) in .txt/.csv files. For PII injects. Interactive.

**`Linux/modules/sudoCheck.sh`** [UI]
: Quick: shows `sudo -l` and sudo/wheel group members. Read-only.

**`linux/bashrc_enhancements`** [OS]
: Forensic bash history: append mode, timestamps, 5000 lines, read-only vars (tamper-proof). Append to `/etc/bash.bashrc`. Not a script.

---

# Wazuh / IDS

**`Linux/Wazuh/installWazuhAgent.sh`** [GH]
: Installs Wazuh agent v4.14.2 (deb or rpm), configures manager IP, starts service. `sudo bash installWazuhAgent.sh <manager_ip>`

**`Linux/Wazuh/initCustomWazuhManagerConfigs.sh`** [GH]
: Adds custom rule for non-local SSH auth detection + block-ssh active response + archive logging + Filebeat module. Run on Wazuh Manager.

**`Linux/Wazuh/initCustomWazuhAgentConfigs.sh`** [GH]
: Configures agent with block-ssh active response. Run on each agent host.

**`Linux/Wazuh/Active Response/block-ssh.sh`** [GH]
: Kills SSH sessions from attacking IP. Called automatically by Wazuh. Needs jq. Not manual.

---

# Ansible Automation (OSUSec)

All from: https://github.com/osusec/prccdc26/tree/main/prccdc-2026-main/2025_repo/ansible

**`linux_harden.yaml`**
: Master playbook: disable ptrace, disable IPv6, harden SSH, configure auditd, rotate passwords, save system info. `ansible-playbook linux_harden.yaml`

**`roles/cfg_auditd`**
: Installs auditd, downloads Neo23x0 rules, enables root execve logging.

**`roles/disable_ipv6`**
: Blocks all IPv6 via ip6tables REJECT.

**`roles/disable_ptrace`**
: Sets ptrace\_scope=3 (most restrictive). Needs Yama LSM.

**`roles/harden_ssh`**
: Disables root login, pubkey auth (disables password except root/dcuser via Match), deploys controller's pubkey.

**`roles/rotate_all_passwords`**
: Generates random passwords for all non-root users, sets via SHA-512 hash.

**`roles/save_system_info`**
: Snapshots system state, archives to tarball, fetches to controller.

**`windows/t0_win.yml`**
: Master Windows playbook: disable PS v2, backup DNS, EternalBlue, firewall, tools. `ansible-playbook windows/t0_win.yml`

---

# SIEM / Log Collection (OSUSec)

**LogCrunch** -- Lightweight SIEM for CCDC. Go-based server + agent with WebUI.

- Repo: https://github.com/osusec/prccdc26/tree/main/LogCrunch
- Cloned from: https://github.com/TLop503/LogCrunch (v0.11.1)
- Server needs TLS certs. Agents need `targets.yaml` config.
- Pre-built binaries in `LogCrunch/executables/`.
- Default WebUI password printed to stdout on first start.
- UNIX-only currently.

---

# Service / Web Hardening

**`Linux/services/flawless-hedgehog-apache.sh`** [GH]
: Interactive Apache2 hardening: server signature, directory listing, HTTP methods, cookie flags, XSS header, TRACE, X-Frame-Options, Slowloris timeout. Root.

**`Linux/hardenScripts/php.sh`** [GH]
: Hardens all php.ini: disables exec/system/shell\_exec, uploads, URL includes. Root.

**`broken/fail2ban.sh`** [UI]
: Full fail2ban setup: SSH jails (normal + aggressive + DDoS), auto-detects web/FTP/mail/DB services, creates custom filters, offers IP whitelisting. Root. Interactive. In `broken/` dir -- may have issues.

**`services/sql-docker-compose.yml`** [UI]
: MySQL 8 in Docker on port 13306. **Change default passwords before use.**

---

# Kubernetes

**`Linux/inventory/kubectlInventory.sh`** [GH]
: Full cluster audit: nodes, pods, ingress, images, privileged pods, PVCs, events. Needs kubectl.

**`Linux/kubernetes/rotate_creds.sh`** [GH]
: Rotates a K8s secret and rolling-restarts all referencing deployments. `bash rotate_creds.sh <ns> <secret> <key> <value>`. Needs kubectl + jq.

---

# GPO Backups (UIdaho)

20+ pre-built Group Policy Object backups in `Windows/gpos/`. Import on a DC:

```powershell
Import-GPO -BackupId "{GUID}" -Path ".\gpos" `
    -TargetName "PolicyName" -CreateIfNeeded
```

Repo: https://github.com/Braxton-Marlatt/PRCCDC-UIdaho-Scripts/tree/main/Windows/gpos

---

# Utility

**`Linux/utils/createSudoUser.sh`** [GH]
: Creates user with random 20-char password + sudo/wheel. `sudo bash createSudoUser.sh <username>`

**`Linux/utils/checkForDep.sh`** [GH]
: Checks if package is installed, installs if missing (apt/yum/dnf/pacman). `bash checkForDep.sh <pkg>`

**`Linux/utils/upload.sh`** [GH]
: Uploads file/dir to 0x0.st anonymous hosting. Returns link. `bash upload.sh <file>`

**`linux/opengrep-selfextracting/`** [OS]
: Self-extracting OpenGrep scanner with bundled rules for PHP/HTML/Java/JS/Python/Ruby/C. Build: `bash build.sh`. Run: `./prog.run python /path`

**`Windows/GPO-Update.ps1`** [OS]
: Forces `gpupdate /force`. One-liner.

**`Windows/Sysmon-Installed-Already.ps1`** [OS]
: Installs Sysmon with provided config. Place sysmon64.exe + sysmonconfig.xml in same dir.

**`Windows/TCP-View-Sysinternal.ps1`** [OS]
: `Get-NetTCPConnection | Out-GridView`. Quick GUI port viewer.

**`Windows/restart.ps1`** [UI]
: Remote-reboots list of computers via WinRM. Set `$computers` variable first.
