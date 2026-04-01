# PRCCDC Playbooks

A collection of blue team playbooks, runbooks, and reference sheets for the Pacific Rim Collegiate Cyber Defense Competition (PRCCDC). Covers incident response, service operations, centralized logging, network discovery, and inject response templates — all formatted for print via Quarto and pdflatex.

---

# Forward:

I'll be transparent that many of these were made with heavy use of AI, I'm just sharing my process with how I went about creating the documents.

Nearly all documents I used quarto to make into PDFs, this is a very powerful markdown -> PDF,HTML,DOCX toolkit, which is very much pandoc if you've heard of that, even quarto uses pandoc to complete it's work!

For the cheatsheets I switched to only LaTeX, quarto uses LaTeX to render it's documents but it's structure is very markdown focused, so using a less limiting language like LaTeX worked much better for the more custom documents like the cheatsheets.

In essence though I can have an AI generate rich documents relatively easy by providing it the necessary context and a prompt and have a readable, printable, document to share with people.

I proof read some of the documents, but in my experience good enough is fine, since even if the document can solve a specific problem it at least leads people to the right place. If I had more time before the competition to focus on this, I think that the documents could have been much higher quality but I'm fairly happy with the results as they are.

When using AI tooling, asking to use a PDF to image toolkit works fairly well for letting AI quickly fix formatting issues like text overlapping or running off of the page. Also, resetting AI sessions for every document and having pointed clear instructions for your documents and providing context like another cheatsheet, document or website leads much better results than simpler prompts.

I will say some documents we didn't use at all during the PRCCDC, any of the **logging & SEC** docs, we had practiced setting up rsyslog & SEC but it seemed too complicated in practice. Similar with the **network discovery**, I used SaaS, RunZero, tooling which was much simpler to setup and get results.

Big wins were:
- Script Analysis (UWB, OS, UI), **very helpful**
- All cheatsheets, **very helpful**
- IR & Service Ops Playbooks
- Inject Response Templates
- ~ First 15 Min 
  - def the least useful on this list, needed more time in oven, still having something like this provides structure for your team

---

## Folder Contents

### Playbooks (Quarto/Markdown → PDF)

| Document | Source | Description |
|---|---|---|
| [first-15-minutes/](first-15-minutes/) | `.md` | Entry point — what every team member does the moment they sit down. Covers immediate triage, credential rotation, service inventory. |
| [incident-response/ir-playbook-linux.md](incident-response/ir-playbook-linux.md) | `.md` | Linux IR following NIST SP 800-61: Identification, Containment, Eradication, Recovery. Forensics, persistence hunting, log analysis. |
| [incident-response/ir-playbook-windows.md](incident-response/ir-playbook-windows.md) | `.md` | Windows equivalent — event log analysis, autoruns, lateral movement detection, remediation. |
| [service-operations/service-ops-playbook-linux.md](service-operations/service-ops-playbook-linux.md) | `.md` | Service uptime, troubleshooting, backup/restore for Linux hosts. |
| [service-operations/service-ops-playbook-windows.md](service-operations/service-ops-playbook-windows.md) | `.md` | Same for Windows — IIS, AD, common service failures. |
| [network-discovery/network-discovery-playbook.md](network-discovery/network-discovery-playbook.md) | `.md` | Map an unknown environment from scratch using nmap, arp, and passive enumeration. |
| [logging/rsyslog-server-setup.md](logging/rsyslog-server-setup.md) | `.md` | Stand up a centralized rsyslog server with TLS. |
| [logging/rsyslog-client-linux.md](logging/rsyslog-client-linux.md) | `.md` | Configure Linux hosts to forward logs to the central server. |
| [logging/rsyslog-client-windows.md](logging/rsyslog-client-windows.md) | `.md` | Configure Windows hosts (via nxlog/winlogbeat) to forward logs. |
| [sec/sec-playbook.qmd](sec/sec-playbook.qmd) | `.qmd` | Deploy SEC (Simple Event Correlator) on the rsyslog server for real-time log correlation and automated iptables response. |
| [inject-response/inject-response-templates.md](inject-response/inject-response-templates.md) | `.md` | Fill-in templates for common white team inject deliverables (network diagrams, incident reports, password policies). |
| [script-analysis/script-analysis-cheatsheet.md](script-analysis/script-analysis-cheatsheet.md) | `.md` | Quick reference for reading and triaging unfamiliar bash/PowerShell scripts found on compromised hosts. |

### Reference & Cheat Sheets

| Document | Source | Description |
|---|---|---|
| [refrence-cheat-sheet/sysadmin-cheatsheet.qmd](refrence-cheat-sheet/sysadmin-cheatsheet.qmd) | `.qmd` | General sysadmin quick reference (networking, services, users, processes). |
| [tools-resource-guide/tools-resource-guide.qmd](tools-resource-guide/tools-resource-guide.qmd) | `.qmd` | Index of tools available during competition with usage examples. |
| [refrence-cheat-sheet/SANS_Cheatsheet-Trifold_Cyb-Def_Linux-Essentials.pdf](refrence-cheat-sheet/SANS_Cheatsheet-Trifold_Cyb-Def_Linux-Essentials.pdf) | External | SANS Linux essentials trifold (pre-built, print as-is). |

### Inventory Sheets (LaTeX → PDF)

| Document | Description |
|---|---|
| [inventorysheets/prccdc-host-inventory.tex](inventorysheets/prccdc-host-inventory.tex) | 11-page fill-in form, one per host. Identity, credentials, services, accounts, data locations, findings, timeline. |
| [inventorysheets/prccdc-host-enum-cheatsheet.tex](inventorysheets/prccdc-host-enum-cheatsheet.tex) | 1-page 2-column quick-reference for gathering inventory data on Linux and Windows. |
| [inventorysheets/linux-blueteam-cheatsheet.tex](inventorysheets/linux-blueteam-cheatsheet.tex) | 3-column dense Linux blue team reference: triage, ejection, iptables, user hardening, persistence hunting, auditd, hardening checklist. |

---

## Building PDFs

### Prerequisites

```bash
# Quarto CLI (renders .md and .qmd to PDF)
# https://quarto.org/docs/get-started/
quarto --version

# pdflatex (via TeX Live or MiKTeX)
pdflatex --version

# Required LaTeX packages (installed automatically by TeX Live on first render,
# or install manually via tlmgr):
#   fvextra    -- code block line wrapping
#   xurl       -- URL wrapping in PDF
#   enumitem   -- list formatting
#   longtable  -- multi-page tables
#   booktabs   -- table rules
```

On Ubuntu/Debian:
```bash
sudo apt install quarto texlive-full
```

On Windows: install [Quarto](https://quarto.org/docs/get-started/) and [MiKTeX](https://miktex.org/) or TeX Live.

---

### Rendering a Single Document

```bash
# From the playbooks/ directory:
quarto render incident-response/ir-playbook-linux.md
quarto render sec/sec-playbook.qmd
```

Output PDF is written next to the source file (e.g., `ir-playbook-linux.pdf`).

---

### Inventory Sheets (LaTeX)

```bash
cd inventorysheets/
pdflatex prccdc-host-inventory.tex
pdflatex prccdc-host-enum-cheatsheet.tex
pdflatex linux-blueteam-cheatsheet.tex
```

Run `pdflatex` twice if cross-references or page numbers are wrong on first pass.

---

### YAML Front Matter (Required on All Playbooks)

Every `.md` and `.qmd` playbook must start with this block (adjust title):

```yaml
---
title: "Document Title"
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
```

`fvextra` and `xurl` are mandatory — without them, long code lines and URLs will overflow the page margin and become unreadable when printed.

---

### Using Claude Code to Draft or Extend Playbooks

These playbooks were drafted with [Claude Code](https://claude.ai/code), Anthropic's CLI coding assistant. It has context about this project loaded via [CONTEXT.md](CONTEXT.md). Additionally, this project used [Quarto Authoring & Markdown skill created by posit-dev](https://mcpmarket.com/tools/skills/quarto-authoring-migration)

**What Claude Code does well here:**
- Generating complete step-by-step procedures from a brief description.
- Ensuring consistent formatting (tables, code blocks, verification steps) across all documents.
- Catching bare markdown hyperlinks that will break in print (should be `<https://...>` or plain URLs).
- Drafting inject response templates from a scenario description.

**What to verify manually:**
- Commands tested against the actual distro/version in competition.
- Any IP addresses, hostnames, or credentials are competition-specific placeholders.
- PDF renders correctly before printing — check code block wrapping and table layout.
