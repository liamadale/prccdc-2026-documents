---
title: "Rsyslog Server Setup"
subtitle: "Centralized Log Collection with TLS -- Server Deployment"
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

# Architecture Overview

Linux and Windows clients forward logs over TLS/TCP port 6514 to a centralized rsyslog server running in Docker. The server receives, organizes by hostname/program, and stores logs. Optionally forward to a SIEM.

**Why TLS?** Standard syslog (UDP 514) transmits in plaintext. An attacker with network access can intercept log data, inject false entries to cover tracks, or drop legitimate logs. TLS over TCP (port 6514, per RFC 5425) provides encryption and integrity.

**TLS mode:** We use `AuthMode="anon"` -- the server presents its certificate, clients verify the server's identity using the CA cert, but clients do not need their own certificates. Traffic is still encrypted. This simplifies deployment since you do not need to distribute client certs to every machine.

**Log organization:**

```
/var/log/remote/
+-- web01.yourlab.local/
|   +-- sshd.log
|   +-- sudo.log
|   +-- systemd.log
+-- db01.yourlab.local/
|   +-- mysql.log
|   +-- sshd.log
+-- win-dc01/
    +-- Security.log
    +-- System.log
```

---

# Prerequisites

- Linux VM with Docker and Docker Compose installed
- OpenSSL for certificate generation
- Firewall access on TCP 6514
- Root or sudo access

---

# Step 1: Create Directory Structure and Certificates

All server files live under `/etc/rsyslog-server/`. The setup script creates the TLS certificate authority, server certificate, and proper permissions.

```bash
sudo mkdir -p /etc/rsyslog-server/
cd /etc/rsyslog-server/
sudo chown "$USER:$USER" .

mkdir -p tls/{ca,server,clients}
cd tls
```

## Generate Certificate Authority

```bash
# Generate CA private key (4096-bit)
openssl genrsa -out ca/ca.key 4096

# Generate CA certificate (valid 10 years)
openssl req -new -x509 -days 3650 \
    -key ca/ca.key -out ca/ca.crt \
    -subj "/CN=YOURLAB Syslog CA/O=InfoSec Club/OU=Security Operations"
```

## Generate Server Certificate

```bash
# Server private key (2048-bit)
openssl genrsa -out server/server.key 2048

# Certificate signing request
openssl req -new \
    -key server/server.key \
    -out server/server.csr \
    -subj "/CN=syslog.yourlab.local/O=InfoSec Club"

# Sign with CA (valid 2 years)
openssl x509 -req -days 730 \
    -in server/server.csr \
    -CA ca/ca.crt \
    -CAkey ca/ca.key \
    -CAcreateserial \
    -out server/server.crt
```

## Set Permissions

Private keys must be `600`. Certificate `.crt` files must be `644` (clients need to read them). All parent directories of `.crt` files must be executable so processes can traverse into them.

```bash
cd /etc/rsyslog-server/

# Lock everything down first
chmod -R 600 .

# Make .crt files readable
chmod -R 644 ./tls/**/*.crt

# Make parent directories of .crt files traversable
# Each .crt needs its parent dirs to have +x
find . -name '*.crt' -exec sh -c '
    dir="$(dirname "{}")"
    while [ "$dir" != "." ] && [ "$dir" != "/" ]; do
        chmod +x "$dir"
        dir="$(dirname "$dir")"
    done
' \;
chmod +x .
```

Transfer ownership to root:

```bash
sudo chown -R root:root /etc/rsyslog-server/
```

**Important:** The CA certificate that clients need is at `/etc/rsyslog-server/tls/ca/ca.crt`. You will distribute this file to every client.

---

# Step 2: Rsyslog Configuration

Create the rsyslog configuration file at `/etc/rsyslog-server/config/rsyslog.conf`:

```bash
sudo mkdir -p /etc/rsyslog-server/config
```

Write the following to `/etc/rsyslog-server/config/rsyslog.conf`:

```
# Global Settings
global(
    workDirectory="/var/spool/rsyslog"
    maxMessageSize="64k"
)

# Modules
module(
    load="imtcp"
    StreamDriver.Name="gtls"
    StreamDriver.Mode="1"
    StreamDriver.AuthMode="anon"
)

# TLS Configuration
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/rsyslog-tls/ca/ca.crt"
    DefaultNetstreamDriverCertFile="/etc/rsyslog-tls/server/server.crt"
    DefaultNetstreamDriverKeyFile="/etc/rsyslog-tls/server/server.key"
)

# TLS Listener on port 6514
input(
    type="imtcp"
    port="6514"
    StreamDriver.Name="gtls"
    StreamDriver.Mode="1"
    StreamDriver.AuthMode="anon"
)

# Plain TCP on 1514 for testing. Disable in production!
input(
    type="imtcp"
    port="1514"
    StreamDriver.Name="ptcp"
    StreamDriver.Mode="0"
    StreamDriver.AuthMode="anon"
)

# Template: Organize by hostname and program
template(
    name="RemoteLogs"
    type="string"
    string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
)

# Template: JSON format for SIEM
template(
    name="JsonFormat"
    type="string"
    string="{\"timestamp\":\"%timegenerated:::date-rfc3339%\",
        \"host\":\"%HOSTNAME%\",
        \"program\":\"%programname%\",
        \"severity\":\"%syslogseverity-text%\",
        \"message\":\"%msg:::json%\"}\n"
)

# Rules: Store remote logs by hostname/program
if $fromhost-ip != "127.0.0.1" then {
    action(type="omfile" dynaFile="RemoteLogs")
    stop
}

# Local logging
*.* /var/log/messages
```

**Note on paths inside the container:** The Docker volumes map `/etc/rsyslog-server/tls` on the host to `/etc/rsyslog-tls` inside the container. The config file references container-internal paths (`/etc/rsyslog-tls/...`), not host paths.

---

# Step 3: Docker Compose

Create `/etc/rsyslog-server/docker-compose.yml`:

```yaml
services:
    rsyslog:
        image: rsyslog/rsyslog:latest
        container_name: rsyslog-server
        restart: unless-stopped
        ports:
            - 6514:6514
            - 1514:1514
        command: >
            /bin/sh -c
            'apt-get update &&
             apt-get -y install rsyslog-gnutls &&
             rsyslogd -n -f /etc/rsyslog.conf'
        volumes:
            - type: bind
              source: ./config/rsyslog.conf
              target: /etc/rsyslog.conf
              read_only: true
            - type: bind
              source: ./tls
              target: /etc/rsyslog-tls
              read_only: true
            - type: bind
              source: /var/log/remote
              target: /var/log/remote
            - type: bind
              source: rsyslog-spool
              target: /var/spool/rsyslog
        environment:
            - TZ=America/Los_Angeles
        healthcheck:
            test: ["CMD", "nc", "-z", "localhost", "6514"]
            interval: 30s
            timeout: 10s
            retries: 3

volumes:
    rsyslog-spool:
```

**Key details:**

- The `command` override installs `rsyslog-gnutls` at container start (the base image does not include it).
- The `rsyslog/rsyslog:latest` image is the official rsyslog Docker image.
- Logs are written to `/var/log/remote` on the host via bind mount. Change this path if you prefer a different location (e.g., `./logs`).
- Port 1514 is the plain TCP testing port. Remove it after confirming TLS works.

---

# Step 4: Deploy

```bash
cd /etc/rsyslog-server/

# Start the container
sudo docker compose up -d

# Verify it's running
sudo docker compose ps

# Watch container logs for startup errors
sudo docker compose logs -f rsyslog

# Check ports are listening
ss -tlnp | grep -E '6514|1514'
```

## Open Firewall

```bash
# UFW (Ubuntu/Debian)
sudo ufw allow 6514/tcp comment "Rsyslog TLS"

# firewalld (RHEL/CentOS/Fedora)
sudo firewall-cmd --permanent --add-port=6514/tcp
sudo firewall-cmd --reload

# iptables (direct)
sudo iptables -A INPUT -p tcp --dport 6514 -j ACCEPT
```

---

# Step 5: Quick Test (Plain TCP)

Before configuring TLS on clients, verify basic connectivity using the plain TCP port:

```bash
# From any Linux client, send a test message
logger -n <server-ip> -P 1514 -T "Test from $(hostname)"

# On the server, check for the message
sudo ls -la /var/log/remote/
sudo docker compose logs rsyslog | tail -20
```

If messages appear, the server is working. Proceed to configure TLS clients.

---

# Hardening (Post-Testing)

## Disable Plain TCP Listener

After all clients are confirmed working over TLS, remove the plain TCP listener:

1. Edit `/etc/rsyslog-server/config/rsyslog.conf` and remove or comment out:

```
# Plain TCP on 1514 for testing. Disable in production!
input(
    type="imtcp"
    port="1514"
    StreamDriver.Name="ptcp"
    StreamDriver.Mode="0"
    StreamDriver.AuthMode="anon"
)
```

2. Edit `/etc/rsyslog-server/docker-compose.yml` and remove the `1514:1514` port mapping.

3. Restart:

```bash
cd /etc/rsyslog-server/
sudo docker compose down
sudo docker compose up -d
```

## Firewall Rules

Restrict TCP 6514 to known client IPs/subnets only:

```bash
# UFW example: allow only from 10.0.0.0/24
sudo ufw delete allow 6514/tcp
sudo ufw allow from 10.0.0.0/24 to any port 6514 proto tcp
```

## Log Rotation

Set up log rotation to prevent disk exhaustion:

```bash
sudo tee /etc/logrotate.d/rsyslog-remote > /dev/null << 'EOF'
/var/log/remote/*/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    postrotate
        docker exec rsyslog-server kill -HUP 1
    endscript
}
EOF
```

## Certificate Management

| Consideration | Recommendation |
|---|---|
| Key strength | 2048-bit minimum for server, 4096-bit for CA |
| Validity period | 1-2 years for server cert, renew before expiry |
| Storage | Private keys must be `600`, never world-readable |
| CA key security | Keep CA private key offline or restricted after signing |

---

# Troubleshooting

## Common Issues

| Symptom | Likely Cause | Solution |
|---|---|---|
| Connection refused on 6514 | Firewall blocking port | Check `ufw`/`firewalld` rules, verify `ss -tlnp` shows 6514 |
| Container won't start | rsyslog-gnutls install fails | Check internet access from container, inspect `docker compose logs` |
| TLS handshake failed | Certificate mismatch | Verify CA cert, server cert, and key are all from the same CA |
| Permission denied on key files | Wrong file permissions | Keys must be `600`, certs `644`, parent dirs need `+x` |
| No logs in `/var/log/remote/` | Bind mount issue | Check Docker volume mounts, verify container can write to path |
| Logs arrive but no subdirectories | Template not applied | Check `rsyslog.conf` rules and template, verify `$fromhost-ip` filter |

## Check Docker Logs

```bash
sudo docker compose logs -f rsyslog
sudo docker compose logs rsyslog | grep -i -E 'tls|error|fail'
```

## Verify TLS is Working

From any machine with openssl:

```bash
openssl s_client \
    -connect <server-ip>:6514 \
    -CAfile /etc/rsyslog-tls/ca.crt

# Should show "Verify return code: 0 (ok)"
```

## Verify Connections

```bash
# On the server, check active TLS connections
sudo ss -tnp | grep 6514
```

---

# Quick Reference

## File Layout on Server

```
/etc/rsyslog-server/
+-- config/
|   +-- rsyslog.conf
+-- tls/
|   +-- ca/
|   |   +-- ca.crt         <-- Distribute to ALL clients
|   |   +-- ca.key
|   +-- server/
|   |   +-- server.crt
|   |   +-- server.key
|   +-- clients/
+-- docker-compose.yml
```

## Ports

| Port | Protocol | Purpose | When to Use |
|---|---|---|---|
| 6514 | TCP/TLS | Production syslog receiver | Always |
| 1514 | TCP/plain | Testing only | Remove after testing |

## Essential Commands

```bash
# Start / stop / restart
sudo docker compose up -d
sudo docker compose down
sudo docker compose restart

# View logs
sudo docker compose logs -f rsyslog
sudo ls -la /var/log/remote/
sudo tail -f /var/log/remote/*/syslog

# Check health
sudo docker compose ps
ss -tlnp | grep 6514
```
