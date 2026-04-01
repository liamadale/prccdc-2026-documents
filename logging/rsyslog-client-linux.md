---
title: "Rsyslog Client Setup -- Linux"
subtitle: "Forwarding Logs to Centralized Rsyslog Server over TLS"
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

This playbook configures a Linux host to forward all syslog messages to the centralized rsyslog server over TLS (TCP port 6514). Clients only need the CA certificate to verify the server's identity -- no client certificates are required.

**What gets forwarded:** All syslog messages (`*.*`) -- authentication events, service status, cron jobs, kernel messages, application logs routed through syslog.

**Prerequisites:**

- The rsyslog server is deployed and listening on TCP 6514 (see Rsyslog Server Setup playbook)
- You have the CA certificate file (`ca.crt`) from the server at `/etc/rsyslog-server/tls/ca/ca.crt`
- You know the rsyslog server's IP address
- SSH access to the target Linux client

---

# Supported Distributions

This playbook covers two families:

| Family | Examples | Init System | Package Manager |
|---|---|---|---|
| Debian-like | Ubuntu, Debian, Kali | systemd | apt |
| Alpine | Alpine Linux | OpenRC | apk |

For RHEL/CentOS/Fedora, the procedure is identical to Debian-like except use `dnf install rsyslog-gnutls` instead of `apt-get install`.

---

# Automated Setup (From Server)

If you have SSH access from the rsyslog server to the client, you can configure everything remotely. Set the environment variables and run from the rsyslog server host:

```bash
TARGET_USER=svcadmin TARGET_HOST=10.0.0.50 ./client_setup.sh
```

Optional variables:

| Variable | Default | Purpose |
|---|---|---|
| `TARGET_USER` | (required) | SSH username on target host |
| `TARGET_HOST` | (required) | IP or hostname of target client |
| `HOST_IP` | `10.0.0.44` | IP of the rsyslog server (what the client connects to) |
| `USE_SUDO_ON_TARGET` | `true` | Set to `false` if already root on target |

The script auto-detects whether the target is Debian-like or Alpine and runs the appropriate setup.

---

# Manual Setup -- Debian/Ubuntu

## Step 1: Transfer CA Certificate

From the rsyslog server:

```bash
scp /etc/rsyslog-server/tls/ca/ca.crt \
    <user>@<client-ip>:/tmp/
```

On the client:

```bash
sudo mkdir -p /etc/rsyslog-tls
sudo mv /tmp/ca.crt /etc/rsyslog-tls/
sudo chmod 644 /etc/rsyslog-tls/ca.crt
```

## Step 2: Install TLS Module

```bash
sudo apt-get update
sudo apt-get install -y rsyslog-gnutls
```

For RHEL/CentOS/Fedora:

```bash
sudo dnf install -y rsyslog-gnutls
```

## Step 3: Configure Rsyslog Forwarding

Create `/etc/rsyslog.d/tls-client.conf`:

```bash
sudo tee /etc/rsyslog.d/tls-client.conf > /dev/null << 'EOF'
# TLS Setup
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/rsyslog-tls/ca.crt"
)

# Forward all logs to server
*.* action(
    type="omfwd"
    Target="SERVER_IP_HERE"
    Port="6514"
    Protocol="tcp"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="anon"
    queue.type="LinkedList"
    queue.filename="syslog_fwd"
    queue.maxDiskSpace="1g"
    queue.saveOnShutdown="on"
    action.resumeRetryCount="-1"
)
EOF
```

**Replace `SERVER_IP_HERE`** with your rsyslog server's IP address (e.g., `10.0.0.44`).

**Configuration details:**

| Parameter | Value | Purpose |
|---|---|---|
| `Target` | Server IP | Where to send logs |
| `Port` | 6514 | TLS syslog port |
| `StreamDriverAuthMode` | anon | Server-only authentication (no client cert needed) |
| `queue.type` | LinkedList | In-memory queue with disk spillover |
| `queue.maxDiskSpace` | 1g | Buffer up to 1 GB if server is unreachable |
| `queue.saveOnShutdown` | on | Persist queued messages across rsyslog restarts |
| `action.resumeRetryCount` | -1 | Retry forever until server is reachable |

## Step 4: Restart and Verify

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

Check for errors:

```bash
# Validate config syntax
sudo rsyslogd -N1
```

---

# Manual Setup -- Alpine Linux

Alpine uses OpenRC instead of systemd and may not have `/etc/rsyslog.d/` by default.

## Step 1: Transfer CA Certificate

From the rsyslog server:

```bash
scp /etc/rsyslog-server/tls/ca/ca.crt \
    <user>@<client-ip>:/tmp/
```

On the client:

```bash
sudo mkdir -p /etc/rsyslog-tls
sudo mv /tmp/ca.crt /etc/rsyslog-tls/
sudo chmod 644 /etc/rsyslog-tls/ca.crt
```

## Step 2: Install TLS Module

```bash
sudo apk add rsyslog-tls
```

If rsyslog itself is not installed:

```bash
sudo apk add rsyslog rsyslog-tls
sudo rc-update add rsyslog default
```

## Step 3: Create Config Directory and File

```bash
sudo mkdir -p /etc/rsyslog.d
```

Verify that `/etc/rsyslog.conf` includes the drop-in directory. Look for this line:

```
$IncludeConfig /etc/rsyslog.d/*.conf
```

If it is missing, add it to the end of `/etc/rsyslog.conf`:

```bash
echo '$IncludeConfig /etc/rsyslog.d/*.conf' | sudo tee -a /etc/rsyslog.conf
```

Now create the TLS forwarding config:

```bash
sudo tee /etc/rsyslog.d/tls-client.conf > /dev/null << 'EOF'
# TLS Setup
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/rsyslog-tls/ca.crt"
)

# Forward all logs to server
*.* action(
    type="omfwd"
    Target="SERVER_IP_HERE"
    Port="6514"
    Protocol="tcp"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="anon"
    queue.type="LinkedList"
    queue.filename="syslog_fwd"
    queue.maxDiskSpace="1g"
    queue.saveOnShutdown="on"
    action.resumeRetryCount="-1"
)
EOF
```

**Replace `SERVER_IP_HERE`** with your rsyslog server's IP address.

## Step 4: Restart and Verify

```bash
sudo rc-service rsyslog restart
sudo rc-service rsyslog status
```

---

# Testing

## Send a Test Message

```bash
logger -t TEST "TLS syslog test from $(hostname)"
```

## Verify on the Server

On the rsyslog server:

```bash
# Check for the client's hostname directory
sudo ls -la /var/log/remote/

# Look for the test message
sudo grep -r "TLS syslog test" /var/log/remote/
```

## Quick Test Over Plain TCP (Before TLS)

If you want to verify basic network connectivity before configuring TLS, test against the plain TCP port (1514):

```bash
logger -n <server-ip> -P 1514 -T "Plain TCP test from $(hostname)"
```

---

# Troubleshooting

## Common Issues

| Symptom | Likely Cause | Solution |
|---|---|---|
| `rsyslogd: error loading module 'lmnsd_gtls'` | rsyslog-gnutls not installed | Install `rsyslog-gnutls` (Debian) or `rsyslog-tls` (Alpine) |
| `could not load CA certificate` | Wrong path or missing file | Verify `/etc/rsyslog-tls/ca.crt` exists and is `644` |
| `TLS handshake failed` | CA cert does not match server cert | Re-copy `ca.crt` from server |
| Config syntax error on restart | Typo in tls-client.conf | Run `sudo rsyslogd -N1` to see exact error |
| Queue growing, logs not delivered | Server unreachable | Check firewall, network path, server status |
| No `/etc/rsyslog.d/` (Alpine) | Directory not created | `sudo mkdir -p /etc/rsyslog.d` and add `$IncludeConfig` line |

## Debug Mode

```bash
# Stop rsyslog and run in foreground with debug output
sudo systemctl stop rsyslog     # or: sudo rc-service rsyslog stop
sudo rsyslogd -dn

# Look for TLS-related errors in the output
# Press Ctrl+C to stop, then restart normally
sudo systemctl start rsyslog    # or: sudo rc-service rsyslog start
```

## Verify TLS Connection

From the client, test that the server's TLS certificate is valid:

```bash
openssl s_client \
    -connect <server-ip>:6514 \
    -CAfile /etc/rsyslog-tls/ca.crt

# Should show "Verify return code: 0 (ok)"
```

## Check Queue Status

```bash
# View rsyslog internal stats (if impstats module is loaded)
sudo cat /var/log/syslog | grep -i "queue"
```

---

# Quick Reference

## Files on Client

```
/etc/rsyslog-tls/
+-- ca.crt                  <-- CA certificate (from server)

/etc/rsyslog.d/
+-- tls-client.conf         <-- TLS forwarding configuration
```

## Essential Commands

```bash
# Debian/Ubuntu/RHEL
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
sudo rsyslogd -N1              # Validate config

# Alpine
sudo rc-service rsyslog restart
sudo rc-service rsyslog status

# Test
logger -t TEST "test message"
```

## Automated Client Setup (Run from Server)

```bash
# Debian-like target
TARGET_USER=svcadmin TARGET_HOST=10.0.0.50 \
    HOST_IP=10.0.0.44 ./client_setup.sh

# Alpine target (auto-detected)
TARGET_USER=root TARGET_HOST=10.0.0.60 \
    HOST_IP=10.0.0.44 USE_SUDO_ON_TARGET=false ./client_setup.sh
```
