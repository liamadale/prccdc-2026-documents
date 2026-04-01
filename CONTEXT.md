# PRCCDC Playbook Context

## What is PRCCDC?

Pacific Rim Collegiate Cyber Defense Competition. Blue teams defend a pre-built enterprise network against a live red team over the course of the competition. Teams inherit an environment they did not build and must secure it while keeping services running.

## Environment

- Bespoke per team: mixed Windows Server versions (2008-2022), mixed Linux distros (Ubuntu, CentOS, Debian, RHEL, SUSE, potentially obscure/old ones like Ubuntu 9).
- Each team member is typically assigned one or more hosts they may have never used before.
- Systems may have pre-existing misconfigurations, backdoors, or weak credentials planted by competition organizers.
- Systems can break on their own from age, misconfiguration, or dependency issues when updating/installing packages.

## Scoring (3 Categories)

### 1. Service Uptime (Continuous, Automated)
A scoring engine periodically checks that specific services are functional:
- **SSH:** Can it log in with a valid username/password?
- **RDP:** Is port 3389 open and accepting TCP connections?
- **HTTP/HTTPS:** Is the website serving the correct content (specific text/JS must be visible)?
- **Other services** (DNS, SMTP, FTP, databases, SMB, etc.) are tested similarly -- the scorer checks that the service responds correctly, not just that the port is open.

Points are awarded continuously for every successful check. Downtime = lost points you can never recover.

### 2. Defense (Red Team Resistance)
Points are lost or flags are captured when:
- PII or sensitive data is exfiltrated from databases.
- Machines are fully compromised/rooted.
- Red team plants flags or demonstrates persistent access.

### 3. Injects (Timed Tasks from White Team)
Business-scenario tasks delivered during competition. Examples:
- "Produce a network diagram of your environment."
- "Write an incident report for the breach on host X."
- "Provide a password policy document."
- "Deploy a new service on host Y."
- "Where are all database files stored across your network?"

Injects require you to already have detailed system knowledge (IPs, OS versions, service inventory, database locations, account lists, network topology). The inventory sheets exist to capture this information rapidly.

## Playbook Structure

```
playbooks/
├── incident-response/       # "Someone broke in" -- detection, containment, eradication
│   ├── ir-playbook-linux.md
│   └── ir-playbook-windows.md
├── service-operations/       # "Something broke" -- uptime, troubleshooting, backup/restore
│   ├── service-ops-playbook-linux.md
│   └── service-ops-playbook-windows.md
├── first-15-minutes/         # Entry point -- what to do the moment you sit down
│   └── first-15-minutes-runbook.md
├── network-discovery/        # Map the environment from scratch
│   └── network-discovery-playbook.md
├── inject-response/          # Templates for common inject deliverables
│   └── inject-response-templates.md
├── logging/                  # Centralized log collection with TLS
│   ├── rsyslog-server-setup.md
│   ├── rsyslog-client-linux.md
│   └── rsyslog-client-windows.md
└── sec/                      # Real-time log correlation and automated response
    └── sec-playbook.qmd
```

## Key Constraints

- **Documents will be printed.** All links must be bare URLs, not markdown hyperlinks. Quarto renders to PDF via pdflatex.
- **Team members vary in skill.** Procedures must be explicit and step-by-step, not assumed knowledge.
- **Systems are unfamiliar.** Commands should include distro-agnostic alternatives or note which distro they apply to.
- **No internet during competition** (typically). All reference material must be on paper.
- **Quarto YAML header** is required on all .md playbooks for PDF rendering. Include toc, pdflatex engine, fvextra for code wrapping, and xurl for URL wrapping.

## Existing Supporting Documents (Outside This Directory)

- **Host Inventory Sheet** (`inventorysheets/prccdc-host-inventory.tex`): 11-page fill-in form, one per host. Covers identity, credentials, security posture, networking, services, accounts, data, findings, timeline.
- **Host Enumeration Cheatsheet** (`inventorysheets/prccdc-host-enum-cheatsheet.tex`): 1-page, 2-column quick-reference for gathering inventory data on both Linux and Windows.
- **Linux Blue Team Cheatsheet** (`inventorysheets/linux-blueteam-cheatsheet.tex`): 3-column dense reference covering triage, ejection, iptables, user hardening, persistence hunting, log analysis, forensics, auditd, hardening checklist.
- **Windows Blue Team Cheatsheet** (`Cheat Sheets PRCCDC/windows/windows-blueteam-cheatsheet.tex`): Same format for Windows -- triage, ejection, firewall, user hardening, persistence, event log analysis, forensics, hardening checklist.

## Writing Style

- Professional IT admin / technical writer tone.
- Every action must have a clear, copy-pasteable procedure.
- Verification step after every significant action ("confirm it worked").
- No chatbot-style language, no emojis, no trailing summaries.
- Concise but complete. Favor tables for reference data, code blocks for commands.
