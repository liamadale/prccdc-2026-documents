---
title: "Linux Incident Response & Intrusion Detection Playbook"
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

# Linux Incident Response & Intrusion Detection Playbook

This guide details forensic methods for identifying intrusion and immediate incident response techniques for terminating unauthorized access on Linux systems. It follows the NIST SP 800-61 incident handling lifecycle: Identification, Containment, Eradication, and Recovery.

---

## Phase 1: Identification

This phase focuses on forensic visibility -- determining *if* and *how* a system has been compromised.

### 1. SSH Login Analysis

SSH logs are the primary source of truth for remote access attempts. You are looking for anomalies in volume (brute force) or context (unfamiliar IPs, logins at odd hours, or unexpected usernames).

**Log Locations:**

| Distribution | Log File |
|---|---|
| Debian/Ubuntu | `/var/log/auth.log` |
| RHEL/CentOS/Fedora | `/var/log/secure` |

::: {.callout-note}
Adjust the log path in the commands below to match your distribution.
:::

**View Successful Logins:**

```bash
# Show all accepted authentication events (password or public key)
grep "Accepted" /var/log/auth.log | less
```

**Identify Brute Force Attempts (Failed Logins):**

```bash
# Count failed password attempts by source IP, sorted by frequency
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

Any IP with hundreds or thousands of failed attempts is a brute force source. Note these IPs for firewall blocking in Phase 2.

**Check Currently Logged-In Users:**

```bash
# Show who is logged in right now and what they are running
w

# Show the 20 most recent login records, including the source host
last -a | head -n 20
```

**Check for Logins by Non-Human or Unexpected Accounts:**

```bash
# List all users that have successfully authenticated via SSH
grep "Accepted" /var/log/auth.log | awk '{print $9}' | sort | uniq
```

Compare this output against known authorized users. Any unfamiliar username warrants immediate investigation.

---

### 2. Detecting Service Hijacking & Persistence

Attackers often replace legitimate service binaries with malicious ones, create new systemd units, or install cron jobs to maintain access across reboots.

**Check for Recently Added or Modified Systemd Services:**

```bash
# List all unit files in /etc/systemd/system sorted by modification time (newest first)
ls -lt /etc/systemd/system/

# List all enabled services and look for anything unfamiliar
systemctl list-unit-files --type=service --state=enabled
```

Review each unfamiliar service. Check its `ExecStart` path and verify the binary it points to.

**Inspect a Suspicious Service:**

```bash
# View the full unit file for a given service
systemctl cat <service-name>

# Check whether the service is active and when it was last started
systemctl status <service-name>
```

**Verify Service Binaries Against Package Manager Records:**

If a standard service (e.g., `nginx`, `sshd`) behaves unexpectedly, verify that its binary has not been tampered with.

```bash
# Debian/Ubuntu: verify installed package integrity (reports modified files)
dpkg -V

# RHEL/CentOS: verify all installed RPM packages (reports modified files)
rpm -Va
```

Any binary that shows a hash mismatch (`5` in the output) has been modified since installation and should be treated as compromised.

**Audit All Cron Jobs:**

Attackers frequently use `cron` to re-establish reverse shells or re-download malware on a schedule.

```bash
# List crontabs for every user on the system
for user in $(cut -f1 -d: /etc/passwd); do
  echo "=== Crontab for $user ===";
  crontab -u "$user" -l 2>/dev/null;
done

# Check system-wide cron directories for unexpected scripts
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/
```

Look for entries that download files (`wget`, `curl`), execute scripts from `/tmp` or `/dev/shm`, or pipe output to `bash`.

**Check Init Scripts (SysVinit):**

On older systems or those that still support SysVinit alongside systemd:

```bash
ls -la /etc/init.d/
```

Compare against known services. Any unfamiliar script should be inspected.

---

### 3. Identifying Suspicious Processes

You are looking for processes with high CPU usage, deceptive names (e.g., a process named `[kworker]` that runs from `/tmp` instead of as a kernel thread), or unexpected network connections.

**Process Tree View:**

```bash
# Display the full process tree with PIDs
pstree -p
```

A shell (`sh`, `bash`) spawned by a web server process (`apache2`, `nginx`, `www-data`) is a strong indicator of a web shell exploit.

**Real-Time Monitoring:**

```bash
# Interactive process viewer (press F6 to sort, F9 to kill)
htop
```

If `htop` is not installed, use `top`. Press `M` to sort by memory, `P` to sort by CPU.

**List All Listening Ports and Their Owning Processes:**

```bash
# -t tcp, -u udp, -l listening, -p show process, -n numeric addresses
sudo ss -tulpn
```

Compare against known services. Any listening port you do not recognize requires investigation.

**List Established Connections to External Hosts:**

```bash
# Filter for established TCP connections and display the owning process
sudo ss -tunp | grep ESTAB
```

Look for connections to unknown external IPs, especially on non-standard ports.

**Investigate a Specific Suspicious Process:**

```bash
# Replace 1234 with the PID of the suspect process
# Show all open files (including network sockets)
lsof -p 1234

# Show the full command line used to start the process
cat /proc/1234/cmdline | tr '\0' ' '; echo

# Show the actual binary on disk (may differ from the reported name)
ls -la /proc/1234/exe
```

If `/proc/<PID>/exe` points to a deleted file or a path in `/tmp`, `/dev/shm`, or a user's home directory, the process is almost certainly malicious.

---

### 4. Additional Indicators of Compromise (IoC)

**Suspicious Files in World-Writable Directories:**

Attackers commonly stage tools and malware in directories that any user can write to.

```bash
# List files in common staging directories, sorted by modification time
ls -lat /tmp/
ls -lat /var/tmp/
ls -lat /dev/shm/

# Find executables in these directories
find /tmp /var/tmp /dev/shm -type f -executable 2>/dev/null

# Find files modified in the last 24 hours in these directories
find /tmp /var/tmp /dev/shm -type f -mtime -1 2>/dev/null
```

Any compiled binary, shell script, or encoded payload in these locations is suspicious.

**Unauthorized SSH Keys:**

Attackers add their own public key to `authorized_keys` for persistent, password-less access.

```bash
# Check authorized_keys for every user with a home directory
for user in $(awk -F: '$6 ~ /home/ || $1 == "root" {print $1}' /etc/passwd); do
  keyfile="$(eval echo ~$user)/.ssh/authorized_keys"
  if [ -f "$keyfile" ]; then
    echo "=== $user ($keyfile) ==="
    cat "$keyfile"
  fi
done
```

Compare each key against a known-good list. Remove any unrecognized keys immediately.

**Unexpected User Accounts:**

```bash
# List accounts with login shells (not nologin/false)
grep -v -E '(nologin|false)$' /etc/passwd

# List accounts with UID 0 (root-equivalent)
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

Only `root` should have UID 0. Any other account with UID 0 is a backdoor.

**Suspicious SUID/SGID Binaries:**

Attackers sometimes set the SUID bit on a copy of `/bin/bash` to retain root access.

```bash
# Find all SUID binaries on the system
find / -perm -4000 -type f 2>/dev/null

# Find all SGID binaries on the system
find / -perm -2000 -type f 2>/dev/null
```

Compare the output against a known baseline. Common legitimate SUID binaries include `/usr/bin/passwd`, `/usr/bin/sudo`, and `/usr/bin/su`. Any SUID binary in `/tmp`, `/home`, or `/var` is almost certainly malicious.

**Check for Kernel Module Persistence:**

Attackers may load malicious kernel modules (rootkits) to hide processes or files.

```bash
# List all currently loaded kernel modules
lsmod

# Check for modules set to load at boot
ls -la /etc/modules-load.d/
cat /etc/modules 2>/dev/null
```

Compare loaded modules against a known-good list. Unfamiliar module names warrant investigation.

---

### 5. Rootkit Detection

Rootkits are designed to hide attacker activity from standard tools. Dedicated scanning tools can detect their signatures and behavioral patterns.

**Using chkrootkit:**

```bash
# Install (if not already present)
sudo apt install chkrootkit    # Debian/Ubuntu
sudo yum install chkrootkit    # RHEL/CentOS

# Run a full scan
sudo chkrootkit
```

Review the output for any lines marked `INFECTED`. Note that chkrootkit can produce false positives; cross-reference any findings before taking action.

**Using rkhunter (Rootkit Hunter):**

```bash
# Install (if not already present)
sudo apt install rkhunter      # Debian/Ubuntu
sudo yum install rkhunter      # RHEL/CentOS

# Update the file properties database
sudo rkhunter --propupd

# Run a full system check
sudo rkhunter --check --sk
```

The `--sk` flag skips interactive key presses. Review the log at `/var/log/rkhunter.log` for any warnings.

---

### 6. Log Integrity Verification

A sophisticated attacker may tamper with or delete logs to cover their tracks. Check for evidence of this.

**Check for Gaps in Log Timestamps:**

```bash
# Look for time gaps in auth.log (entries should be roughly continuous)
awk '{print $1, $2, $3}' /var/log/auth.log | head -n 20
awk '{print $1, $2, $3}' /var/log/auth.log | tail -n 20
```

If the earliest timestamp in the log is suspiciously recent, the log may have been truncated.

**Check for Cleared or Truncated Logs:**

```bash
# Check file sizes -- a zero-byte log file is a red flag
ls -la /var/log/auth.log /var/log/syslog /var/log/secure 2>/dev/null

# Check for evidence of log clearing tools or commands in shell history
grep -E '(truncate|> /var/log|shred|rm.*log)' /home/*/.bash_history /root/.bash_history 2>/dev/null
```

**Review Shell History for All Users:**

```bash
# Check bash history for every user
for user in $(cut -f1 -d: /etc/passwd); do
  histfile="$(eval echo ~$user)/.bash_history"
  if [ -f "$histfile" ]; then
    echo "=== History for $user ==="
    cat "$histfile"
  fi
done
```

Look for evidence of reconnaissance (`whoami`, `id`, `uname -a`, `cat /etc/shadow`), tool downloads (`wget`, `curl`), or lateral movement (`ssh` to other internal hosts).

---

## Phase 2: Containment

Once an active intruder is identified, you must sever their access immediately and prevent reconnection. Speed is critical -- an attacker who realizes they have been detected may attempt to escalate privileges or destroy data.

### 1. Terminating Active Sessions

If you see an attacker currently logged in (identified via `w` or `who`):

**Surgical Method (Kill a Specific Session):**

1. Identify the attacker's pseudo-terminal allocation (e.g., `pts/2`) from the output of `w`.
2. Find the PID of their shell process:

    ```bash
    ps -ft pts/2
    ```

3. Kill the process:

    ```bash
    kill -9 <PID>
    ```

4. Verify the session is gone:

    ```bash
    w
    ```

**Nuclear Method (Kill All Processes Owned by a User):**

If the compromised user account is `hacker`, kill every process they own:

```bash
pkill -KILL -u hacker
```

Then verify with:

```bash
ps -u hacker
```

This should return no results.

---

### 2. Blocking Network Access (Firewalls)

Block the attacker's source IP address immediately. Choose the method that matches your system's firewall.

**Using UFW (Uncomplicated Firewall -- Debian/Ubuntu):**

```bash
# Insert a deny rule at the top of the chain (highest priority)
sudo ufw insert 1 deny from 192.168.1.50

# Reload to apply
sudo ufw reload

# Verify the rule is in place
sudo ufw status numbered
```

**Using iptables (Standard Linux):**

```bash
# Drop all inbound packets from the attacker's IP
sudo iptables -I INPUT -s 192.168.1.50 -j DROP

# Also block forwarded traffic (prevents pivoting through this host)
sudo iptables -I FORWARD -s 192.168.1.50 -j DROP

# Verify
sudo iptables -L -n | grep 192.168.1.50
```

**Using firewalld (RHEL/CentOS/Fedora):**

```bash
# Add a permanent reject rule for the attacker's IP
sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='192.168.1.50' reject"

# Reload to apply
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-rich-rules
```

::: {.callout-tip}
If you have identified multiple attacker IPs, repeat the appropriate command for each one. Consider blocking entire subnets if the IPs share a common range.
:::

---

### 3. Locking the Account

Prevent the compromised user from logging back in while you investigate.

**Lock the Password (Prevents Password Authentication):**

```bash
passwd -l <username>
```

**Disable the Login Shell (Prevents All Interactive Access):**

```bash
usermod -s /sbin/nologin <username>
```

**Remove All SSH Authorized Keys (Prevents Key-Based Access):**

```bash
rm -f /home/<username>/.ssh/authorized_keys
```

**Verify the Account is Locked:**

```bash
# The password field should show '!' or '!!' indicating it is locked
grep <username> /etc/shadow
```

---

### 4. Network Isolation (Nuclear Option)

If you cannot determine the scope of the compromise or suspect lateral movement, isolate the machine from the network entirely.

**Identify the Active Network Interface:**

```bash
ip link show
```

**Disable the Interface:**

```bash
sudo ip link set <interface> down
```

::: {.callout-warning}
This will sever your own remote session if you are connected via SSH. Only do this if you have out-of-band access (console, IPMI/iLO/iDRAC, or physical access).
:::

---

## Phase 3: Eradication & Recovery

After containment, remove all attacker artifacts and restore the system to a known-good state.

### 1. Removing Persistence Mechanisms

Systematically remove every persistence mechanism identified in Phase 1.

**Remove Malicious Cron Jobs:**

```bash
# Edit the user's crontab and delete the offending entries
crontab -u <username> -e

# Or remove the crontab entirely
crontab -u <username> -r

# Remove any malicious files from system cron directories
sudo rm /etc/cron.d/<malicious-file>
```

**Remove Malicious Systemd Services:**

```bash
# Stop and disable the service
sudo systemctl stop <malicious-service>
sudo systemctl disable <malicious-service>

# Remove the unit file
sudo rm /etc/systemd/system/<malicious-service>.service

# Reload the systemd daemon
sudo systemctl daemon-reload
```

**Remove Unauthorized SSH Keys:**

```bash
# Edit the authorized_keys file and remove unrecognized keys
nano /home/<username>/.ssh/authorized_keys
```

**Remove Malicious Kernel Modules:**

```bash
# Unload the module from the running kernel
sudo rmmod <module-name>

# Remove the module file and any auto-load configuration
sudo rm /etc/modules-load.d/<module-name>.conf
```

---

### 2. File Integrity Restoration

Replace any tampered binaries with known-good versions from the package manager.

**Debian/Ubuntu:**

```bash
# Reinstall a specific package whose binaries were modified
sudo apt install --reinstall <package-name>
```

**RHEL/CentOS:**

```bash
# Reinstall a specific package whose binaries were modified
sudo yum reinstall <package-name>
```

**Remove Attacker Tools and Payloads:**

```bash
# Delete any malicious files identified during the investigation
sudo rm -f /tmp/<malicious-file>
sudo rm -f /dev/shm/<malicious-file>
sudo rm -f /var/tmp/<malicious-file>
```

---

### 3. Credential Reset

Assume all credentials on the compromised system are exposed. Reset them.

1. **Change all user passwords:**

    ```bash
    passwd <username>
    ```

2. **Regenerate SSH host keys** (if the attacker had root access):

    ```bash
    sudo rm /etc/ssh/ssh_host_*
    sudo ssh-keygen -A
    sudo systemctl restart sshd
    ```

3. **Rotate any API keys, tokens, or secrets** stored on the system (check environment variables, config files, `.env` files).

4. **Force all users to change passwords at next login:**

    ```bash
    chage -d 0 <username>
    ```

---

### 4. System Hardening

After restoring the system, apply hardening measures to prevent recurrence.

**Disable Root Login Over SSH:**

```bash
# In /etc/ssh/sshd_config, set:
#   PermitRootLogin no
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

**Enforce Key-Based Authentication Only:**

```bash
# In /etc/ssh/sshd_config, set:
#   PasswordAuthentication no
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

**Enable Automatic Security Updates:**

```bash
# Debian/Ubuntu
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# RHEL/CentOS
sudo yum install yum-cron
sudo systemctl enable yum-cron --now
```

**Restrict Cron Access:**

```bash
# Only allow specific users to use cron
echo "root" | sudo tee /etc/cron.allow
```

---

## Resources & Documentation

**Log Analysis:**

- Linux Logging Basics (Red Hat).\
  <https://www.redhat.com/sysadmin/linux-logs>
- Linux Logs Explained (Plesk).\
  <https://www.plesk.com/blog/product-technology/linux-logs-explained/>

**Rootkit Detection Tools:**

- chkrootkit -- Checks locally for signs of a rootkit.\
  <http://www.chkrootkit.org/>
- rkhunter (Rootkit Hunter) -- Scans for rootkits, backdoors, and local exploits.\
  <http://rkhunter.sourceforge.net/>

**Process & Network Analysis:**

- lsof Man Page.\
  <https://man7.org/linux/man-pages/man8/lsof.8.html>
- ss (Socket Statistics) Command Guide.\
  <https://linux.die.net/man/8/ss>

**Incident Response Frameworks:**

- NIST SP 800-61 Rev. 2 -- Computer Security Incident Handling Guide.\
  <https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final>
- SANS Incident Handler's Handbook.\
  <https://www.sans.org/white-papers/33901/>
