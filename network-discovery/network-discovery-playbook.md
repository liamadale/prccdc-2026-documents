---
title: "Network Discovery Playbook"
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

# Network Discovery Playbook

Procedures for rapidly mapping the competition environment: what hosts exist, what services they run, and how they communicate.

---

## Host Discovery

Goal: identify every live host on your assigned subnet(s) as fast as possible.

### Nmap Ping Sweep (fastest if available)

```bash
# Discover live hosts on a /24 subnet (no port scan, just host detection)
nmap -sn 10.0.0.0/24

# Scan multiple subnets
nmap -sn 10.0.0.0/24 10.0.1.0/24 10.0.2.0/24

# Output to a file for later reference
nmap -sn 10.0.0.0/24 -oN /root/host-discovery.txt
```

Nmap `-sn` sends ICMP echo, TCP SYN to port 443, TCP ACK to port 80, and ICMP timestamp. It will find hosts even if ICMP is blocked.

### ARP Table Inspection

If you are already on a host in the subnet, the ARP table shows hosts that have recently communicated:

```bash
# Linux
arp -a
ip neigh show

# Windows
arp -a
```

ARP only shows hosts on the local broadcast domain that the machine has recently talked to. To populate the ARP table, run a ping sweep first.

### Ping Sweep (no tools required)

Linux:

```bash
# Quick /24 ping sweep using bash
for i in $(seq 1 254); do
  ping -c 1 -W 1 10.0.0.$i &
done 2>/dev/null | grep "bytes from"
```

Windows (PowerShell):

```powershell
1..254 | ForEach-Object {
  Test-Connection -ComputerName "10.0.0.$_" -Count 1 -Quiet -ErrorAction SilentlyContinue |
    Where-Object { $_ -eq $true } |
    ForEach-Object { "10.0.0.$using:_is alive" }
}
```

Windows (cmd):

```cmd
for /L %i in (1,1,254) do @ping -n 1 -w 500 10.0.0.%i | find "Reply"
```

**Note:** Some hosts may have ICMP disabled. If a host does not respond to ping, it may still be alive. Follow up with port scans.

### Verification

- Compare discovered hosts against the competition packet's expected host count.
- Record every discovered IP on the host inventory sheet immediately.

---

## Port & Service Scanning

### Quick Scan (scored services only)

Scan only the ports that are commonly scored. This finishes in seconds:

```bash
nmap -p 21,22,25,53,80,443,445,1433,3306,3389,5432 -sV --open 10.0.0.0/24 -oN /root/quick-scan.txt
```

### Targeted Host Scan (all common ports)

Once you know a host is alive, get a full picture:

```bash
# Top 1000 ports with service detection
nmap -sV -sC -O <target-ip> -oN /root/scan-<hostname>.txt

# All 65535 ports (takes several minutes per host -- run in background)
nmap -p- -sV <target-ip> -oN /root/fullscan-<hostname>.txt &
```

### Interpreting Results

| nmap State | Meaning |
|---|---|
| open | Service is listening and accepting connections |
| closed | Port responds but no service is listening |
| filtered | No response -- firewall is dropping packets |

Key fields in nmap output:

- **PORT:** port number and protocol
- **STATE:** open/closed/filtered
- **SERVICE:** nmap's guess at the service name
- **VERSION:** detected software and version (from `-sV`)

### Verification

After scanning, confirm the results match reality:

```bash
# Verify a specific port is open
nc -zv <target-ip> <port>

# Verify HTTP is serving content
curl -s -o /dev/null -w "%{http_code}" http://<target-ip>/
```

Record all discovered services on the host inventory sheet.

---

## Network Topology Mapping

### Step 1: Identify Subnets and Gateways

From each host you have access to:

```bash
# Linux -- show subnets and gateways
ip addr show
ip route show
# Look for "default via X.X.X.X" -- that's the gateway

# Windows
ipconfig /all
route print
# Look for "Default Gateway" and network masks
```

### Step 2: Identify VLANs (if applicable)

```bash
# Linux -- check for VLAN interfaces
ip -d link show
cat /proc/net/vlan/config  # if 8021q module is loaded

# Windows
Get-NetAdapter | Select-Object Name,InterfaceDescription,VlanID
```

### Step 3: Build the Diagram

Use the following table to organize data before drawing:

| Subnet | CIDR | Gateway | VLAN | Hosts in Subnet |
|---|---|---|---|---|
| | /24 | | | |

Draw the diagram with:

1. Internet/scoring engine at the top.
2. Firewall/router below it with its IP.
3. Subnets branching down, each labeled with CIDR.
4. Hosts within each subnet, labeled with IP, OS, and scored services.
5. Arrows between hosts that have service dependencies.

### Step 4: Verify Connectivity

Confirm that routing works as expected:

```bash
# Can hosts in different subnets reach each other?
ping <host-in-other-subnet>
traceroute <host-in-other-subnet>    # Linux
tracert <host-in-other-subnet>       # Windows
```

---

## Cross-Host Communication Map

Identify which hosts depend on other hosts for service delivery.

### Common Dependencies

| Client Service | Depends On | Port | Example |
|---|---|---|---|
| Web application | Database server | 3306, 5432, 1433 | Apache on Host A queries MySQL on Host B |
| All hosts | DNS server | 53 | Name resolution for internal services |
| Domain-joined Windows | Domain controller | 88, 389, 445, 636 | Authentication, GPO, DNS |
| Mail server | DNS server | 53 | MX record lookups |
| Clients | DHCP server | 67/68 | IP address assignment |
| File share clients | SMB/NFS server | 445, 2049 | Shared file access |

### Discovery Methods

From each host, identify what it connects to:

```bash
# Linux -- show established connections
ss -tnp | grep ESTAB
netstat -tnp | grep ESTABLISHED

# Linux -- show DNS server in use
cat /etc/resolv.conf

# Windows -- show established connections
netstat -ano | findstr ESTABLISHED

# Windows -- show DNS server in use
Get-DnsClientServerAddress
ipconfig /all | findstr "DNS Servers"
```

Check application configuration files for remote host references:

```bash
# Web application database connection strings
grep -ri "host\|server\|connect" /var/www/ /etc/apache2/ /etc/nginx/ /etc/httpd/ 2>/dev/null
grep -ri "mysql\|postgres\|sqlserver\|mongodb" /var/www/ 2>/dev/null
```

### Record the Map

| Source Host | Destination Host | Port | Purpose |
|---|---|---|---|
| | | | |

**Critical insight:** If a dependency host goes down, all services that depend on it may fail scoring. Prioritize dependency hosts (DNS, AD, database servers) for hardening and monitoring.

---

## Identifying Host Roles

Map discovered services to standard server roles.

### Role Identification Table

| Open Ports | Likely Role |
|---|---|
| 53 | DNS server |
| 22 | SSH server (Linux/Unix host) |
| 3389 | Windows host with RDP |
| 80, 443 | Web server |
| 25, 587 | Mail server (SMTP) |
| 88, 389, 636, 445, 3268 | Active Directory Domain Controller |
| 3306 | MySQL / MariaDB database server |
| 5432 | PostgreSQL database server |
| 1433 | Microsoft SQL Server |
| 21 | FTP server |
| 445 (without 88) | File server (SMB) |
| 2049 | NFS server |
| 67 | DHCP server |
| 110, 143, 993, 995 | Mail server (POP3/IMAP) |

### Confirming Roles

After identifying likely roles from port scans, confirm by logging in:

```bash
# Linux -- check what's actually running
systemctl list-units --type=service --state=running
ps aux

# Windows
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-WindowsFeature | Where-Object {$_.Installed}
```

### Role Assignment Table

Fill this in as you identify each host:

| IP Address | Hostname | OS | Identified Role(s) | Scored Services | Assigned To |
|---|---|---|---|---|---|
| | | | | | |

---

## When You Can't Use Nmap

Nmap may not be installed, or you may not be able to install it. Use these native alternatives.

### Host Discovery Without Nmap

**Bash (any Linux):**

```bash
# Ping sweep
for i in $(seq 1 254); do
  (ping -c 1 -W 1 10.0.0.$i &>/dev/null && echo "10.0.0.$i is alive") &
done; wait

# /dev/tcp port check (bash built-in, no tools needed)
for ip in 10.0.0.{1..254}; do
  (echo >/dev/tcp/$ip/22 2>/dev/null && echo "$ip:22 open") &
done; wait
```

**PowerShell:**

```powershell
# Ping sweep
1..254 | ForEach-Object -Parallel {
  if (Test-Connection -ComputerName "10.0.0.$_" -Count 1 -Quiet) {
    "10.0.0.$_ is alive"
  }
} -ThrottleLimit 50
```

**Note:** The `-Parallel` parameter requires PowerShell 7+. For older PowerShell:

```powershell
1..254 | ForEach-Object {
  $ip = "10.0.0.$_"
  if (Test-Connection -ComputerName $ip -Count 1 -Quiet -ErrorAction SilentlyContinue) {
    Write-Output "$ip is alive"
  }
}
```

### Port Scanning Without Nmap

**Bash /dev/tcp (no external tools):**

```bash
# Scan common ports on a single host
for port in 21 22 25 53 80 443 445 1433 3306 3389 5432; do
  (echo >/dev/tcp/10.0.0.5/$port 2>/dev/null && echo "Port $port open") &
done; wait
```

**Netcat (nc):**

```bash
# Scan a port range
nc -zv 10.0.0.5 1-1024 2>&1 | grep "open\|succeeded"
```

**PowerShell Test-NetConnection:**

```powershell
# Test specific ports on a host
$ports = @(21,22,25,53,80,443,445,1433,3306,3389,5432)
foreach ($port in $ports) {
  $result = Test-NetConnection -ComputerName 10.0.0.5 -Port $port -WarningAction SilentlyContinue
  if ($result.TcpTestSucceeded) {
    Write-Output "Port $port open"
  }
}
```

**PowerShell .NET sockets (fastest, works on all PS versions):**

```powershell
$ports = @(21,22,25,53,80,443,445,1433,3306,3389,5432)
foreach ($port in $ports) {
  $tcp = New-Object System.Net.Sockets.TcpClient
  try {
    $tcp.Connect("10.0.0.5", $port)
    Write-Output "Port $port open"
    $tcp.Close()
  } catch {}
}
```

**curl (check HTTP/HTTPS):**

```bash
curl -s -o /dev/null -w "%{http_code}" http://10.0.0.5/
curl -sk -o /dev/null -w "%{http_code}" https://10.0.0.5/
```

### Service Version Detection Without Nmap

```bash
# Grab service banners with netcat
echo "" | nc -w 3 10.0.0.5 22    # SSH banner
echo "" | nc -w 3 10.0.0.5 25    # SMTP banner
echo "" | nc -w 3 10.0.0.5 21    # FTP banner

# HTTP server header
curl -sI http://10.0.0.5/ | grep -i "server:"
```
