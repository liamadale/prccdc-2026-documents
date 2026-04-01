---
title: "Inject Response Templates"
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
---

# Inject Response Templates

Skeleton templates and preparation checklists for common CCDC inject categories. These templates are aggregated from inject patterns observed across multiple PRCCDC team repositories (GrayHats, UIdaho, OSUSec). Fill in the blanks with your environment-specific data.

---

## Response Format Guide

Every inject response should follow a professional format. Use this structure unless the inject specifies otherwise:

```
Greetings [Name from inject / "Management"],

[Body of response -- organized with clear headings, tables, and screenshots]

Sincerely,
Team [##]

References
1. [Any external sources cited]
```

**Key rules:**

- Address the person who sent the inject by name
- Match the tone of the inject (formal memo, technical report, etc.)
- Include screenshots wherever the inject asks for "proof" or "evidence"
- Break reports down by host when covering multiple systems
- Label all tables and figures
- When evaluating options, always compare at least 3 candidates with a structured comparison
- Submit on time -- an imperfect on-time submission scores more than a perfect late one

---

## Information You Should Have Ready

Before the first inject arrives, every team member should have the following written down or immediately accessible:

| Data Point | Where to Get It |
|---|---|
| IP address and hostname of every host | Host inventory sheet, `ip a` / `ipconfig` |
| OS and version of every host | Host inventory sheet, `cat /etc/os-release` / `winver` |
| All scored services and which host runs each | Competition packet, scoring engine dashboard |
| All user accounts on each host | `cat /etc/passwd` / `Get-LocalUser` / `Get-ADUser -Filter *` |
| All listening ports and associated services | `ss -tlnp` / `netstat -ano` |
| Database names, locations, and credentials | Host inventory sheet, service config files |
| Network diagram (subnets, gateways, VLANs) | Scan results, `ip route` / `route print` |
| Backup locations and timestamps | Your backup log |
| Team member assignments (who owns which host) | Team captain's assignment sheet |
| Competition packet rules and contact info | Printed competition packet |
| Containers and VMs on each host | `docker ps -a` / VM manager / `kubectl get pods -A` |
| Firewall rules on each host | `iptables -L -n` / `Get-NetFirewallRule` |
| Installed software per host | `dpkg -l` / `rpm -qa` / `Get-ItemProperty HKLM:\...\Uninstall\*` |

**Pre-competition task:** Print blank copies of every template below. During the first 15 minutes, begin filling in environment-specific data so templates are partially complete before injects arrive.

---

## Inject Types -- Reporting & Documentation

### Incident Report

Use this template when asked to document a security incident, breach, or compromise. Injects typically ask for the who, what, when, where, why, and how.

**Title:** Incident Report -- [Brief Description]

**Date/Time of Report:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_  (Name, Role)

**1. Executive Summary**

On [date] at approximately [time], [brief 1-2 sentence description of what happened]. The incident affected [host(s)/service(s)]. The incident has been [contained / eradicated / is ongoing].

**2. Timeline of Events**

| Time | Event | Source |
|---|---|---|
| \_\_:\_\_ | Initial indicator observed | [log file / alert / team member] |
| \_\_:\_\_ | Investigation began | [who investigated] |
| \_\_:\_\_ | Root cause identified | [description] |
| \_\_:\_\_ | Containment action taken | [what was done] |
| \_\_:\_\_ | Eradication completed | [what was removed/fixed] |
| \_\_:\_\_ | Service restored and verified | [verification method] |

**3. Affected Systems**

| Hostname | IP Address | OS | Services Affected | Data at Risk |
|---|---|---|---|---|
| | | | | |

**4. Root Cause Analysis**

- **Attack vector:** [e.g., brute-force SSH, exploited CVE-XXXX-XXXX, phishing, compromised credentials]
- **Vulnerability exploited:** [misconfiguration, unpatched software, weak password, etc.]
- **Attacker actions on system:** [what files were accessed/modified, what processes were run, lateral movement]

**5. Actions Taken**

- Containment: [e.g., blocked attacker IP, disabled compromised account, isolated host]
- Eradication: [e.g., removed backdoor at /path, killed malicious process, rotated credentials]
- Recovery: [e.g., restored config from backup, restarted service, verified scoring]

**6. Impact Assessment**

- **Services impacted:** [list services and downtime duration]
- **Data impacted:** [were records accessed, modified, exfiltrated?]
- **Users impacted:** [which accounts were compromised or affected]

**7. Recommendations**

- [Immediate hardening steps taken]
- [Monitoring improvements]
- [Policy or procedure changes]

---

### Incident Response Policy

Use this template when asked to create an IR policy (distinct from a single incident report).

**Title:** Incident Response Policy

**Effective Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Purpose**

This policy establishes procedures for identifying, responding to, and recovering from security incidents affecting [organization] infrastructure.

**2. Scope**

This policy applies to all systems, networks, and data managed by [organization], including physical hosts, virtual machines, and containers.

**3. Incident Classification**

| Severity | Definition | Response Time | Examples |
|---|---|---|---|
| Critical | Active compromise, data exfiltration, service outage | Immediate (< 15 min) | Ransomware, active backdoor, scored service down |
| High | Attempted compromise, suspicious activity | < 30 min | Brute force attacks, unauthorized account creation |
| Medium | Policy violation, misconfiguration discovered | < 1 hour | Weak passwords found, unnecessary services running |
| Low | Informational, minor anomaly | < 4 hours | Failed login from unknown IP, port scan detected |

**4. Roles and Responsibilities**

| Role | Responsibility |
|---|---|
| Team Captain | Receives incident reports, assigns response, coordinates communication |
| Host Owner | Monitors assigned system, performs initial triage, executes containment |
| Inject Handler | Documents incident using IR form, prepares reports for management |
| All Members | Report suspicious activity immediately to Team Captain |

**5. Incident Response Phases**

1. **Detection:** Monitor logs, alerts, scoring dashboard, and network traffic for anomalies
2. **Triage:** Assess severity, identify affected systems, determine scope
3. **Containment:** Isolate affected systems, block attacker access, preserve evidence
4. **Eradication:** Remove malware/backdoors, patch vulnerabilities, rotate credentials
5. **Recovery:** Restore services from backup if needed, verify scoring, monitor for recurrence
6. **Documentation:** Complete incident report form, update change log, brief team

**6. Reporting Requirements**

All incidents must be documented using the Incident Report Form (see Incident Report template). Reports must include: date/time, type of incident, description, attack vector, impact on services, impact on data, and actions taken.

**7. Evidence Preservation**

- Do not delete malicious files before documenting them (hash, path, timestamps)
- Take memory dumps of suspicious processes before killing them
- Export relevant log segments before they rotate
- Screenshot alerts and anomalous activity

---

### Asset Inventory Report

Use this template when asked to provide an asset inventory, system listing, or infrastructure map. Injects typically require listing all hosts, VMs, and containers with their services.

**Title:** Asset Inventory Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. System Inventory**

| # | Hostname | IP Address | OS / Version | Role | Scored Services | Ports Open | Admin Account | Notes |
|---|---|---|---|---|---|---|---|---|
| 1 | | | | | | | | |
| 2 | | | | | | | | |
| 3 | | | | | | | | |

**2. Virtual Machine and Container Inventory**

| Parent Host | VM/Container Name | Type | OS/Image | IP Address | Services | Ports |
|---|---|---|---|---|---|---|
| | | Docker / VM / K8s Pod | | | | |

**3. Network Summary**

| Subnet | CIDR | Gateway | Purpose | Hosts |
|---|---|---|---|---|
| | | | | |

**4. Service Summary**

| Service | Protocol/Port | Host(s) | Status | Scoring? |
|---|---|---|---|---|
| SSH | TCP/22 | | UP / DOWN | Y / N |
| RDP | TCP/3389 | | UP / DOWN | Y / N |
| HTTP | TCP/80 | | UP / DOWN | Y / N |
| HTTPS | TCP/443 | | UP / DOWN | Y / N |
| DNS | TCP+UDP/53 | | UP / DOWN | Y / N |
| SMTP | TCP/25 | | UP / DOWN | Y / N |
| FTP | TCP/21 | | UP / DOWN | Y / N |
| MySQL | TCP/3306 | | UP / DOWN | Y / N |
| MSSQL | TCP/1433 | | UP / DOWN | Y / N |
| SMB | TCP/445 | | UP / DOWN | Y / N |

**5. Service Dependencies**

| Service | Depends On | Notes |
|---|---|---|
| Web application | MySQL on [host] | Database connection string in /var/www/config |
| DNS | AD on [DC host] | AD-integrated DNS zones |

**Data populated from:** Host inventory sheets, network discovery scans, competition packet, `docker ps`, `kubectl get pods -A`.

---

### Network Diagram

Use this template when asked to produce a network diagram or topology map. Include all physical hosts, VMs, and containers.

**Preparation steps:**

1. Gather all IP addresses from host inventory sheets.
2. Identify subnets and gateways from `ip route` / `route print` output.
3. Identify VLANs if applicable from switch configs or competition packet.
4. Map service dependencies (which hosts talk to which).
5. Enumerate containers and VMs on each host (`docker ps -a`, VM manager).

**Diagram layout (draw by hand or use a whiteboard tool):**

```
                    [Internet / Scoring Engine]
                              |
                         [Firewall/Router]
                         Gateway: ___.___.___.__
                              |
              --------------------------------
              |               |              |
         [Subnet A]     [Subnet B]     [Subnet C]
         ___.___.___     ___.___.___    ___.___.___
         .0/24           .0/24          .0/24
              |               |              |
         +--------+     +--------+     +--------+
         | Host 1 |     | Host 3 |     | Host 5 |
         | IP:    |     | IP:    |     | IP:    |
         | OS:    |     | OS:    |     | OS:    |
         | Svcs:  |     | Svcs:  |     | Svcs:  |
         | [Docker|     +--------+     +--------+
         |  ctnrs]|     +--------+
         +--------+     | Host 4 |
         +--------+     | IP:    |
         | Host 2 |     | OS:    |
         | IP:    |     | Svcs:  |
         | OS:    |     +--------+
         | Svcs:  |
         +--------+
```

**Required elements for the diagram:**

- All hosts with IP, hostname, OS
- Subnets with CIDR notation
- Gateway/router with IP
- Scored services labeled on each host
- Arrows showing service dependencies (e.g., web server arrow to DB server)
- Any VLANs or network segmentation
- Firewall/router position
- Docker containers and VMs shown within their parent host

**Supporting table (attach alongside diagram):**

| Hostname | IP Address | OS | Subnet | Scored Services | Dependencies |
|---|---|---|---|---|---|
| | | | | | |

---

### Network Security Current State Report

Use this template when asked to report on the current state of network protections. Include screenshots of each protection in place.

**Title:** Network Security Current State Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Firewall Status**

| Host | Firewall Tool | Status | Default Policy | Key Rules | Screenshot |
|---|---|---|---|---|---|
| | iptables / ufw / Windows Firewall | Enabled / Disabled | ACCEPT / DROP | [summary] | Fig. X |

**2. Password and Authentication**

| Host | Password Policy Enforced | Complexity | Rotation | Lockout | MFA |
|---|---|---|---|---|---|
| | Y / N | [requirements] | [interval] | [threshold] | Y / N |

**3. Encryption and Certificates**

| Host | Service | TLS/SSL | Certificate Source | Expiry |
|---|---|---|---|---|
| | HTTPS | Y / N | Self-signed / CA | |
| | SMTP | STARTTLS / N | | |

**4. Intrusion Detection / Monitoring**

| Host | Tool | Status | Log Location |
|---|---|---|---|
| | Sysmon / auditd / Wazuh / Suricata | Running / Not installed | |

**5. Antivirus / Endpoint Protection**

| Host | Tool | Signatures Updated | Last Scan |
|---|---|---|---|
| | Defender / ClamAV / None | Y / N (date) | |

**6. Policies in Place**

- [ ] Incident Response Policy
- [ ] Password / Account Policy
- [ ] Network Usage Policy
- [ ] Backup and Recovery Plan
- [ ] Change Management Procedures

**7. Known Gaps and Remediation Plan**

| Gap | Risk | Planned Remediation | Timeline |
|---|---|---|---|
| | | | |

*Attach screenshots showing firewall rules, audit policies, IDS dashboards, and service configurations.*

---

## Inject Types -- Auditing & Compliance

### Login Attempt Audit

Use this template when asked to audit login attempts across all hosts.

**Title:** Login Attempt Audit Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**Per-host results:**

| Hostname | OS | Total Logins | Failed Logins | Successful Logins |
|---|---|---|---|---|
| | | | | |

**Top 10 accounts by login activity (per host):**

**Host: [hostname] ([IP])**

| # | Account | Total Attempts | Failed | Successful | Source IPs |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |

**Commands used to gather data:**

Linux:

```bash
# Failed logins
grep "Failed password" /var/log/auth.log | wc -l
grep "Failed password" /var/log/auth.log | grep -oP 'for \K\S+' | sort | uniq -c | sort -rn | head -10

# Successful logins
grep "Accepted" /var/log/auth.log | wc -l
grep "Accepted" /var/log/auth.log | grep -oP 'for \K\S+' | sort | uniq -c | sort -rn | head -10

# journalctl alternative
journalctl _COMM=sshd | grep "Failed" | wc -l
```

Windows:

```powershell
# Failed logons (Event 4625)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} |
    Group-Object {$_.Properties[5].Value} | Sort-Object Count -Desc | Select -First 10

# Successful logons (Event 4624)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} |
    Where-Object {$_.Properties[8].Value -in @(2,10,11)} |
    Group-Object {$_.Properties[5].Value} | Sort-Object Count -Desc | Select -First 10
```

**Findings and recommendations:**

[Summarize any accounts with abnormally high failed login counts, source IPs of brute force attempts, and actions taken (lockout, IP ban, etc.)]

---

### Admin Account Audit

Use this template when asked to audit administrative or privileged accounts across all hosts.

**Title:** Administrative Account Audit Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**Per-host breakdown:**

**Host: [hostname] ([IP], [OS])**

| Account | Privilege Level | Authorized? | Action Taken |
|---|---|---|---|
| root | root | Y | N/A |
| admin | sudo group | Y | N/A |
| unknownuser | sudo group | N | Removed from sudo group |

**Summary by category:**

| Host | Authorized Admins | Unauthorized Admins Found | Permissions Removed | Normal Users |
|---|---|---|---|---|
| | | | Y / N | |

**Commands used:**

Linux:

```bash
# Users with UID 0
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Users in sudo/wheel group
getent group sudo wheel

# Users with sudo privileges
grep -v '^#' /etc/sudoers | grep -v '^$'
cat /etc/sudoers.d/*

# Remove unauthorized sudo access
deluser baduser sudo                    # Debian/Ubuntu
gpasswd -d baduser wheel               # RHEL/CentOS
```

Windows:

```powershell
# Local administrators
Get-LocalGroupMember -Group "Administrators"

# Domain admins (on DC)
Get-ADGroupMember "Domain Admins" | Select Name
Get-ADGroupMember "Enterprise Admins" | Select Name
Get-ADGroupMember "Schema Admins" | Select Name

# Remove unauthorized admin
Remove-LocalGroupMember -Group "Administrators" -Member "baduser"
Remove-ADGroupMember -Identity "Domain Admins" -Members "baduser" -Confirm:$false
```

---

### Unnecessary Software Audit

Use this template when asked to audit and document unnecessary software, services, or open ports.

**Title:** Unnecessary Software and Services Audit

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**Per-host findings:**

**Host: [hostname] ([IP], [OS])**

| Item | Type | Risk | Action Taken |
|---|---|---|---|
| telnet-server | Unnecessary service | Cleartext remote access | Stopped and disabled |
| gcc | Development tool | Enables on-host compilation | Removed |
| 0.0.0.0:8888 | Open port (unknown) | Unauthorized listener | Killed process, blocked port |

**Summary:**

| Host | Unnecessary Services Found | Packages Removed | Ports Closed |
|---|---|---|---|
| | | | |

**Commands used:**

Linux:

```bash
# List all running services
systemctl list-units --type=service --state=running

# List all listening ports
ss -tlnp

# Remove package
apt remove --purge <package>    # Debian/Ubuntu
yum remove <package>            # RHEL/CentOS

# Disable service
systemctl disable --now <service>
```

Windows:

```powershell
# List installed software
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select DisplayName,Publisher | Sort DisplayName

# List running services
Get-Service | Where-Object {$_.Status -eq "Running"} | Sort DisplayName

# List listening ports
netstat -ano | findstr LISTENING

# Remove software
Get-Package -Name "*<name>*" | Uninstall-Package
```

---

### Password Strength Audit

Use this template when asked to audit password policies and enforce complexity across all hosts.

**Title:** Password Strength Audit and Enforcement Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Current Policy Status (per host)**

| Host | OS | Min Length | Complexity | Max Age | Lockout | Policy Source | Compliant? |
|---|---|---|---|---|---|---|---|
| | | | | | | PAM / GPO / local | Y / N |

**2. Hosts Where Policy Was Missing**

| Host | Policy Applied | Settings |
|---|---|---|
| | [describe what was configured] | |

**3. Accounts with Forced Password Change**

| Host | Account | Reason | Password Changed? |
|---|---|---|---|
| | | Inherited / weak / default | Y / N |

**4. Implementation**

Linux:

```bash
# Check current policy
grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS|PASS_MIN_LEN" /etc/login.defs
grep pam_pwquality /etc/pam.d/common-password 2>/dev/null || \
grep pam_pwquality /etc/pam.d/system-auth 2>/dev/null

# Set policy
# /etc/login.defs
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_MIN_LEN    12

# /etc/pam.d/common-password (Debian) or /etc/pam.d/system-auth (RHEL)
password requisite pam_pwquality.so retry=3 minlen=12 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1

# Force password change on next login
chage -d 0 <username>
```

Windows:

```powershell
# Check current policy
net accounts

# Set policy
net accounts /minpwlen:12 /maxpwage:90 /minpwage:1 /uniquepw:5 /lockoutthreshold:5

# Force password change on next login
Set-ADUser -Identity <username> -ChangePasswordAtLogon $true

# Or for local users
net user <username> /logonpasswordchg:yes
```

---

### PII Discovery Report

Use this template when asked to find and secure personally identifiable information (PII) across infrastructure.

**Title:** PII Discovery and Remediation Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. PII Discovery Results**

| # | Host | File Path | PII Type | Records Found | Sensitivity |
|---|---|---|---|---|---|
| 1 | | /path/to/file | SSN / Credit Card / Name+DOB | [count] | High / Medium |

**2. Remediation Actions**

| # | Host | File | Action Taken | Verified |
|---|---|---|---|---|
| 1 | | /path/to/file | Encrypted / Deleted / Moved to secure location | Y / N |

**3. Search Methods Used**

```bash
# Search for SSN patterns (XXX-XX-XXXX)
grep -rn '[0-9]\{3\}-[0-9]\{2\}-[0-9]\{4\}' /home/ /var/www/ /tmp/ 2>/dev/null

# Search for credit card patterns
grep -rn '[0-9]\{4\}[ -]\?[0-9]\{4\}[ -]\?[0-9]\{4\}[ -]\?[0-9]\{4\}' /home/ /var/www/ 2>/dev/null

# Search for email addresses
grep -rn '[a-zA-Z0-9._%+-]\+@[a-zA-Z0-9.-]\+\.[a-zA-Z]\{2,\}' /home/ /var/www/ 2>/dev/null

# Search common file types
find /home /var/www /tmp -name "*.csv" -o -name "*.xlsx" -o -name "*.txt" -o -name "*.sql" \
    2>/dev/null | head -50
```

```powershell
# Windows -- search for SSN patterns
Select-String -Path C:\Users\*\*.txt,C:\Users\*\*.csv -Pattern '\d{3}-\d{2}-\d{4}' -Recurse
```

**4. Ongoing Controls**

- [Describe monitoring put in place to detect new PII]
- [Describe access controls applied to directories containing PII]

---

## Inject Types -- Policy & Communication

### Password / Account Policy

Use this template when asked to produce an account or password policy document.

**Title:** Password and Account Security Policy

**Effective Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Password Complexity Requirements**

- Minimum length: 12 characters
- Must contain at least three of: uppercase letter, lowercase letter, digit, special character
- Must not contain the username or common dictionary words
- Must not reuse any of the last 5 passwords

**2. Password Rotation**

- All user passwords must be changed every 90 days
- Service account passwords must be changed every 180 days
- All default and inherited passwords must be changed immediately upon system access

**3. Account Lockout Policy**

- Lock account after 5 failed login attempts within 15 minutes
- Lockout duration: 30 minutes (auto-unlock) or manual unlock by administrator
- Administrator accounts: lock after 3 failed attempts, manual unlock only

**4. Privileged Access**

- Administrative access is granted only to designated team members
- No shared administrator passwords; each admin has a unique account
- Privilege escalation (sudo / Run as Administrator) is logged
- Root / built-in Administrator direct login is disabled where possible

**5. Account Management**

- Disable all accounts not required for operations or scoring
- Remove or disable all default/vendor accounts (guest, test, etc.)
- Maintain a written list of all active accounts and their purpose
- Review active accounts every [competition day / shift change]

**6. Implementation Commands**

Linux (set password policy):

```bash
# /etc/login.defs
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_MIN_LEN    12

# /etc/pam.d/common-password (Debian/Ubuntu) or /etc/pam.d/system-auth (RHEL)
password requisite pam_pwquality.so retry=3 minlen=12 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1
```

Windows (Group Policy or local):

```powershell
# View current policy
net accounts

# Set password policy
net accounts /minpwlen:12 /maxpwage:90 /minpwage:1 /uniquepw:5 /lockoutthreshold:5
```

---

### Network and Asset Usage Policy

Use this template when asked to create a company usage policy. Injects may also require an employee agreement form and a company-wide memo.

**Title:** Network and Asset Usage Policy

**Effective Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Purpose**

This policy governs the acceptable use of [organization] network resources, physical assets, email and electronic communications, and the handling of sensitive information.

**2. Scope**

This policy applies to all employees, contractors, and authorized users who access company networks, systems, or data.

**3. Acceptable Use**

- Company network and internet access is provided for business purposes
- Limited personal use is permitted provided it does not interfere with work duties or consume excessive bandwidth
- All electronic communications using company systems are subject to monitoring

**4. Prohibited Use**

- Accessing, downloading, or distributing illegal or inappropriate content
- Using company resources for personal commercial gain
- Installing unauthorized software or hardware on company systems
- Sharing credentials or granting unauthorized access to company resources
- Using competitor services on company time and equipment
- Circumventing security controls (VPNs, proxies, firewall bypass)

**5. Sensitive Information Handling**

- PII and confidential data must not be stored on unapproved systems
- Sensitive files must be encrypted at rest and in transit
- Access to sensitive data follows least-privilege principles
- Report any accidental exposure of sensitive data immediately

**6. Enforcement**

- Violations will result in disciplinary action up to and including termination
- [Organization] reserves the right to monitor all network traffic and system usage
- Suspected policy violations will be investigated by the security team

**7. Reporting**

Report policy violations to: [Team Captain / Security Officer] at [contact method]

---

**Employee Agreement Form (attach as separate page):**

> **Network and Asset Usage Agreement**
>
> I, \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_, acknowledge that I have read and understood the [Organization] Network and Asset Usage Policy. I agree to comply with all terms and conditions. I understand that violation of this policy may result in disciplinary action.
>
> Signature: \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  Date: \_\_\_\_\_\_\_\_\_\_\_\_
>
> Printed Name: \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_  Department: \_\_\_\_\_\_\_\_\_\_\_\_

---

**Company-Wide Memo (attach as separate page):**

> **MEMORANDUM**
>
> **TO:** All Employees
>
> **FROM:** [Security Team / Management]
>
> **DATE:** \_\_\_\_\_\_\_\_\_\_\_\_
>
> **RE:** New Network and Asset Usage Policy
>
> A new Network and Asset Usage Policy has been implemented effective immediately. This policy covers acceptable use of company networks, devices, email, and handling of sensitive information.
>
> **Key points:**
>
> - Company resources are for business use; limited personal use is acceptable
> - Do not install unauthorized software or share your credentials
> - All network activity is subject to monitoring
> - Report security concerns immediately to [contact]
>
> **To obtain a full copy of the policy:** [location -- shared drive, email, printed copies in break room]
>
> **To report policy violations:** Contact [name] at [email/phone]
>
> Thank you for your cooperation in keeping our systems secure.

---

### Bug Bounty / Vulnerability Disclosure Program

Use this template when asked to create a bug bounty or vulnerability disclosure program.

**Title:** Vulnerability Disclosure and Bug Bounty Program

**Effective Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Scope**

This program covers the following services maintained by [organization]:

| Service | Repository / URL | In Scope |
|---|---|---|
| | | Y / N |

**2. Severity Classification (CVSS-based)**

| Severity | CVSS Score | Example | Reward |
|---|---|---|---|
| Critical | 9.0 -- 10.0 | Remote code execution, auth bypass | [reward] |
| High | 7.0 -- 8.9 | Privilege escalation, SQL injection | [reward] |
| Medium | 4.0 -- 6.9 | XSS, information disclosure | [reward] |
| Low | 0.1 -- 3.9 | Minor info leak, best-practice deviation | [reward] |

**3. Submission Process**

1. Reporter submits bug via [system -- e.g., Forgejo issue tracker, email, web form]
2. Include: description, steps to reproduce, severity assessment, proof of concept
3. Team acknowledges receipt within [1 hour]
4. Team triages and assigns severity within [2 hours]
5. Valid submissions receive reward; invalid submissions receive polite explanation

**4. Response Protocol for Confirmed Bugs**

1. Verify the vulnerability and assess impact
2. Develop and test a fix
3. Deploy the fix to production
4. Verify the system is not currently compromised via the reported vulnerability
5. Notify the reporter that the fix is deployed
6. Coordinate public disclosure timeline with reporter (default: 90 days)

**5. Rules of Engagement**

- Do not access, modify, or delete data belonging to other users
- Do not perform denial-of-service attacks
- Do not use automated scanning tools without prior approval
- Act in good faith and comply with all applicable laws

**6. Example Reports** (attach 3 examples: low, medium, high severity)

---

## Inject Types -- Service Deployment

### New Service Deployment

Use this template when an inject requires deploying a new service (IDS, VPN, SFTP, file sharing, etc.).

**Pre-deployment checklist:**

- [ ] Read the inject requirements completely -- what service, what configuration, what proof of completion
- [ ] Identify the target host and confirm OS / available resources
- [ ] Take a full backup of the host before making changes
- [ ] Check if the service package is already installed
- [ ] Add firewall rule for the new service port before starting

**Linux deployment procedure:**

```bash
# 1. Backup current state
tar czf /root/backups/pre-deploy-$(date +%Y%m%d-%H%M).tar.gz /etc/

# 2. Install the service (adjust for distro)
# Debian/Ubuntu
apt-get update && apt-get install -y <package>
# RHEL/CentOS
yum install -y <package>
# SUSE
zypper install -y <package>

# 3. Configure the service
cp /etc/<service>/config /etc/<service>/config.bak
# Edit configuration per inject requirements
vi /etc/<service>/config

# 4. Open firewall port
iptables -A INPUT -p tcp --dport <port> -j ACCEPT
# or: ufw allow <port>/tcp

# 5. Start and enable the service
systemctl enable --now <service>

# 6. Verify the service is running
systemctl status <service>
ss -tlnp | grep <port>

# 7. Test the service functions correctly
# (service-specific test command)

# 8. Verify existing scored services still pass
# Check scoring dashboard
```

**Windows deployment procedure:**

```powershell
# 1. Backup current state
reg export HKLM\SYSTEM C:\Backups\pre-deploy-system.reg

# 2. Install the role/feature
Install-WindowsFeature -Name <FeatureName> -IncludeManagementTools

# 3. Configure the service per inject requirements
# (service-specific configuration)

# 4. Open firewall port
New-NetFirewallRule -DisplayName "Allow <Service>" -Direction Inbound `
    -Protocol TCP -LocalPort <port> -Action Allow

# 5. Start the service
Start-Service <ServiceName>
Set-Service <ServiceName> -StartupType Automatic

# 6. Verify the service is running
Get-Service <ServiceName>
netstat -ano | findstr <port>

# 7. Test the service functions correctly
# (service-specific test)

# 8. Verify existing scored services still pass
```

**Post-deployment:**

- [ ] Service is running and responding
- [ ] Existing scored services are unaffected
- [ ] Document the change in the change log
- [ ] Note the new service in your host inventory sheet
- [ ] Take screenshots of the service running (for inject proof)

---

### Security Tool Evaluation and Deployment

Use this template when asked to evaluate and deploy a security tool (IDS, VPN, EDR, scanner, etc.). Injects typically require comparing at least 3 options.

**Title:** [Tool Category] Evaluation and Deployment Report

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Requirements**

[Summarize what the inject asks for and any constraints (must be free, must support Linux and Windows, must integrate with AD, etc.)]

**2. Candidates Evaluated**

| Criteria | Option A: [name] | Option B: [name] | Option C: [name] |
|---|---|---|---|
| License / Cost | | | |
| OS Support | | | |
| Ease of Deployment | | | |
| Feature Set | | | |
| Community / Support | | | |
| Resource Usage | | | |
| Integration | | | |

**3. Selected Solution: [name]**

**Rationale:** [2-3 sentences on why this option was chosen over the others]

**4. Deployment Documentation**

[Step-by-step installation and configuration with screenshots of each major step. This section should be detailed enough that another team could recreate the deployment from scratch.]

1. [Step]
2. [Step]
3. [Step]

**5. Operations Guide**

- **Starting the service:** [command]
- **Stopping the service:** [command]
- **Checking status:** [command]
- **Viewing logs / alerts:** [command or dashboard URL]
- **Updating signatures / rules:** [command]
- **Troubleshooting:** [common issues and fixes]

**6. Host Coverage**

| Host | Agent/Sensor Installed | Status | Notes |
|---|---|---|---|
| | Y / N | Running / Failed | [compatibility issues, alternative approach] |

*For hosts where the tool is incompatible, describe the alternative solution implemented.*

**7. Evidence**

- Screenshot: tool installed and running
- Screenshot: sample alert or detection
- Screenshot: dashboard or log output
- Screenshot: persistence across reboot (if applicable)

---

### Automated Inventory with Ansible

Use this template when asked to create automated solutions for inventory or log collection across hosts.

**Title:** Automated Application Inventory -- Ansible Implementation

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Solution Overview**

Ansible playbook that connects to all hosts and collects installed application lists, writing results to a centralized log file.

**2. Host Inventory File** (`hosts.ini`)

```ini
[linux]
web01 ansible_host=10.0.0.10 ansible_user=admin
db01  ansible_host=10.0.0.11 ansible_user=admin

[windows]
dc01  ansible_host=10.0.0.20 ansible_user=admin ansible_connection=winrm
web02 ansible_host=10.0.0.21 ansible_user=admin ansible_connection=winrm
```

**3. Playbook** (`inventory_apps.yml`)

```yaml
---
- name: Inventory installed applications (Linux)
  hosts: linux
  become: yes
  tasks:
    - name: Get installed packages (Debian)
      shell: dpkg -l | tail -n +6 | awk '{print $2, $3}'
      register: packages
      when: ansible_os_family == "Debian"

    - name: Get installed packages (RedHat)
      shell: rpm -qa --queryformat '%{NAME} %{VERSION}\n'
      register: packages
      when: ansible_os_family == "RedHat"

    - name: Save to log
      local_action:
        module: copy
        content: "=== {{ inventory_hostname }} ===\n{{ packages.stdout }}\n"
        dest: "/var/log/app_inventory_{{ inventory_hostname }}.log"

- name: Inventory installed applications (Windows)
  hosts: windows
  tasks:
    - name: Get installed software
      win_shell: |
        Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
          Select-Object DisplayName, DisplayVersion | Format-Table -Auto
      register: packages

    - name: Save to log
      local_action:
        module: copy
        content: "=== {{ inventory_hostname }} ===\n{{ packages.stdout }}\n"
        dest: "/var/log/app_inventory_{{ inventory_hostname }}.log"
```

**4. Running the Playbook**

```bash
ansible-playbook -i hosts.ini inventory_apps.yml
```

**5. Screenshots**

- Screenshot: playbook execution output
- Screenshot: snippet of generated log file

---

## Inject Types -- Documentation & Change Management

### Change Request / Change Log

Use this template when asked to document changes made to the environment.

**Change Log**

| # | Date/Time | Host | Change Description | Made By | Reason | Rollback Plan | Verified |
|---|---|---|---|---|---|---|---|
| 1 | | | | | | | Y / N |
| 2 | | | | | | | Y / N |
| 3 | | | | | | | Y / N |

**Individual Change Request (for detailed submissions):**

- **Change ID:** \_\_\_\_
- **Date/Time:** \_\_\_\_\_\_\_\_\_\_\_\_
- **Requested By:** \_\_\_\_\_\_\_\_\_\_\_\_
- **Approved By:** \_\_\_\_\_\_\_\_\_\_\_\_
- **Host(s) Affected:** \_\_\_\_\_\_\_\_\_\_\_\_
- **Description of Change:** [What was changed and how]
- **Reason for Change:** [Why -- security hardening, inject requirement, service fix]
- **Files/Settings Modified:** [List specific files, registry keys, or settings]
- **Backup Taken:** Y / N -- Location: \_\_\_\_\_\_\_\_\_\_\_\_
- **Rollback Plan:** [Exact steps to undo this change]
- **Testing/Verification:** [How you confirmed the change worked and didn't break scoring]
- **Post-Change Status:** Service scoring: PASS / FAIL

---

### Backup & Recovery Plan

Use this template when asked to document backup and recovery procedures.

**Title:** Backup and Recovery Plan

**Date:** \_\_\_\_\_\_\_\_\_\_\_\_

**Prepared By:** \_\_\_\_\_\_\_\_\_\_\_\_

**1. Backup Inventory**

| Host | Data/Config Backed Up | Backup Method | Backup Location | Frequency | Last Backup |
|---|---|---|---|---|---|
| | /etc/ configs | tar archive | /root/backups/ | On change | |
| | Web root (/var/www/) | tar archive | /root/backups/ | On change | |
| | MySQL databases | mysqldump | /root/backups/ | Hourly | |
| | IIS site + config | Export-WebConfiguration | `C:\Backups\` | On change | |
| | AD/GPO | wbadmin system state | `C:\Backups\` | On change | |
| | Registry hives | reg export | `C:\Backups\` | Before changes | |

**2. Backup Procedures**

Linux -- full config backup:

```bash
tar czf /root/backups/etc-$(date +%Y%m%d-%H%M).tar.gz /etc/
tar czf /root/backups/webroot-$(date +%Y%m%d-%H%M).tar.gz /var/www/
```

Linux -- database backup:

```bash
mysqldump --all-databases -u root -p > /root/backups/mysql-all-$(date +%Y%m%d-%H%M).sql
pg_dumpall -U postgres > /root/backups/pgdump-all-$(date +%Y%m%d-%H%M).sql
```

Windows -- config backup:

```powershell
reg export HKLM\SYSTEM C:\Backups\system-hive.reg
reg export HKLM\SOFTWARE C:\Backups\software-hive.reg
Copy-Item C:\inetpub -Recurse -Destination C:\Backups\inetpub-backup
```

Windows -- system state backup:

```powershell
wbadmin start systemstatebackup -backuptarget:C:\Backups\
```

**3. Recovery Procedures**

Linux -- restore config:

```bash
tar xzf /root/backups/etc-YYYYMMDD-HHMM.tar.gz -C /
systemctl restart <affected-service>
```

Linux -- restore database:

```bash
mysql -u root -p < /root/backups/mysql-all-YYYYMMDD-HHMM.sql
psql -U postgres < /root/backups/pgdump-all-YYYYMMDD-HHMM.sql
```

Windows -- restore registry:

```powershell
reg import C:\Backups\system-hive.reg
```

**4. Recovery Targets**

- **RTO (Recovery Time Objective):** Service must be restored within 15 minutes of failure detection.
- **RPO (Recovery Point Objective):** Maximum acceptable data loss is 1 hour (based on backup frequency).

---

## Inject Types -- Creative / Infographic

### Infographic

Use this template when asked to create an infographic (password best practices, cyberattack types, phishing awareness, etc.).

**Infographic checklist:**

- [ ] Full page, visually appealing layout
- [ ] Use diagrams, icons, or images (cite sources for any images used)
- [ ] Simple language -- target audience is non-technical employees
- [ ] Include contact information for reporting security concerns

**Password Policy Infographic -- key content:**

- How to create a strong password (length > complexity, passphrases)
- Why password reuse is dangerous (credential stuffing)
- Examples of weak vs. strong passwords
- When to change your password
- How to use a password manager
- Who to contact if you suspect your password was compromised

**Cyberattack Infographic -- key content (top 5 attack types):**

For each attack type include:

1. What it is (1-2 sentence plain-English description)
2. Main attack vectors (how it reaches you)
3. Indicators of compromise (how to spot it)
4. What employees should do (initial response steps)

Suggested attack types: Phishing, Ransomware, Social Engineering, Malware/Trojans, Denial of Service

**Tools for creating infographics:**

- Canva (free tier): https://www.canva.com
- Google Slides (export as PDF)
- LibreOffice Draw / Impress
- PowerPoint (export as PDF)

**Image source citation (attach on separate page):**

| Image | Source URL | License |
|---|---|---|
| | | CC0 / CC-BY / etc. |

---

## Inject Response Workflow

**1. Receive and Read**

- Team captain (or designated inject handler) receives the inject.
- Read the entire inject before acting. Note the deadline, deliverable format, and any specific requirements.
- Identify exactly what deliverables are expected (report, screenshots, working service, policy document, etc.).

**2. Triage Priority**

| Priority | Criteria | Action |
|---|---|---|
| Critical | Affects scoring or has a short deadline (< 30 min) | Drop current task, assign immediately |
| High | Significant point value or moderate deadline (30-60 min) | Assign within 5 minutes |
| Normal | Standard deadline, standard point value | Queue and assign when someone is available |
| Low | Long deadline, low point value, or "nice to have" | Complete only if no higher-priority work exists |

**3. Delegate**

- Assign to the team member best suited (based on host ownership or skill).
- If the inject spans multiple hosts, break it into subtasks and assign each.
- The assignee confirms they understand the deliverable before starting.

**4. Execute**

- Use the relevant template above as a starting point.
- Fill in environment-specific data from inventory sheets.
- If the inject requires system changes, follow the change management discipline (backup first, verify after).
- Take screenshots as you go -- do not wait until the end.
- For evaluation injects (IDS, VPN, etc.), always compare at least 3 options.

**5. Review and Submit**

- A second team member reviews the deliverable before submission.
- Confirm the format matches what was requested (PDF, email, printed, etc.).
- Ensure professional formatting: greeting, organized body, signature block with team number.
- Submit before the deadline. An imperfect on-time submission scores more than a perfect late one.

**6. Log**

- Record: inject ID, summary, assignee, time received, time submitted, and whether it was accepted.

---

## Inject Tracking Sheet

| Inject # | Summary | Received | Deadline | Assigned To | Status | Submitted | Accepted |
|---|---|---|---|---|---|---|---|
| | | HH:MM | HH:MM | | Not Started / In Progress / Done | HH:MM | Y / N / Pending |
