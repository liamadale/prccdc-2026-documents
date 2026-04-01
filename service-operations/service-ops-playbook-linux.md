---
title: "Linux Service Operations Playbook"
subtitle: "Procedures for maintaining scored service uptime, troubleshooting failures, and recovering from misconfigurations on Linux systems."
date: today
format:
  pdf:
    toc: true
    toc-depth: 3
    number-sections: true
    colorlinks: true
    fontsize: 11pt
    geometry:
      - margin=1in
    include-in-header:
      text: |
        \usepackage{fvextra}
        \DefineVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}
        \usepackage{xurl}
        \usepackage{etoolbox}
        \renewcommand{\arraystretch}{1.3}
        \AtBeginEnvironment{longtable}{\footnotesize\setlength{\tabcolsep}{3pt}}
tbl-colwidths: auto
---

## Backup & Restore

### Configuration Backup

**Before editing any configuration file, always make a backup copy first.**

```bash
# Backup a single config file
cp /etc/<service>/config.conf /etc/<service>/config.conf.bak.$(date +%Y%m%d-%H%M)

# Backup all of /etc/
tar czf /root/backups/etc-$(date +%Y%m%d-%H%M).tar.gz /etc/

# Backup a web root
tar czf /root/backups/webroot-$(date +%Y%m%d-%H%M).tar.gz /var/www/

# Backup a specific service's config directory
tar czf /root/backups/apache-$(date +%Y%m%d-%H%M).tar.gz /etc/apache2/ /etc/httpd/ 2>/dev/null
tar czf /root/backups/nginx-$(date +%Y%m%d-%H%M).tar.gz /etc/nginx/
```

Create the backup directory if it does not exist:

```bash
mkdir -p /root/backups
```

### Database Backup & Restore

**MySQL / MariaDB:**

```bash
# Backup all databases
mysqldump --all-databases -u root -p > /root/backups/mysql-all-$(date +%Y%m%d-%H%M).sql

# Backup a specific database
mysqldump -u root -p <dbname> > /root/backups/mysql-<dbname>-$(date +%Y%m%d-%H%M).sql

# Restore all databases
mysql -u root -p < /root/backups/mysql-all-YYYYMMDD-HHMM.sql

# Restore a specific database
mysql -u root -p <dbname> < /root/backups/mysql-<dbname>-YYYYMMDD-HHMM.sql
```

**PostgreSQL:**

```bash
# Backup all databases
pg_dumpall -U postgres > /root/backups/pgdump-all-$(date +%Y%m%d-%H%M).sql

# Backup a specific database
pg_dump -U postgres <dbname> > /root/backups/pgdump-<dbname>-$(date +%Y%m%d-%H%M).sql

# Restore all databases
psql -U postgres < /root/backups/pgdump-all-YYYYMMDD-HHMM.sql

# Restore a specific database
psql -U postgres <dbname> < /root/backups/pgdump-<dbname>-YYYYMMDD-HHMM.sql
```

### Rollback Procedures

```bash
# Restore a single config file from .bak
cp /etc/<service>/config.conf.bak.YYYYMMDD-HHMM /etc/<service>/config.conf
systemctl restart <service>

# Restore /etc/ from tar backup
tar xzf /root/backups/etc-YYYYMMDD-HHMM.tar.gz -C /
systemctl restart <affected-service>

# Restore a web root
rm -rf /var/www/html/*
tar xzf /root/backups/webroot-YYYYMMDD-HHMM.tar.gz -C /
systemctl restart apache2   # or nginx, httpd
```

**Verification after any rollback:**

```bash
systemctl status <service>
# Check scoring dashboard to confirm service passes
```

## Distro-Agnostic Service Management

### Service Control Across Distributions

| Action | systemd (modern) | SysVinit (legacy) |
|------------|----------------------|---------------------------------------------|
| Start service | `systemctl start <svc>` | `service <svc> start` or `/etc/init.d/<svc> start` |
| Stop service | `systemctl stop <svc>` | `service <svc> stop` |
| Restart service | `systemctl restart <svc>` | `service <svc> restart` |
| Check status | `systemctl status <svc>` | `service <svc> status` |
| Enable at boot | `systemctl enable <svc>` | `update-rc.d <svc> defaults` (Debian) / `chkconfig <svc> on` (RHEL) |
| List all services | `systemctl list-units --type=service` | `service --status-all` |

**Detect init system:**

```bash
# If PID 1 is systemd, use systemctl
ps -p 1 -o comm=
# Output: "systemd" or "init"
```

**Package managers by distro:**

| Distro | Package Manager | Install Command | Update/Refresh |
|------------|----------------|--------------------------|----------------------|
| Debian, Ubuntu | apt | `apt-get install -y <pkg>` | `apt-get update` |
| RHEL 7, CentOS 7 | yum | `yum install -y <pkg>` | `yum check-update` |
| RHEL 8+, CentOS 8+, Fedora | dnf | `dnf install -y <pkg>` | `dnf check-update` |
| SUSE, openSUSE | zypper | `zypper install -y <pkg>` | `zypper refresh` |
| Arch | pacman | `pacman -S --noconfirm <pkg>` | `pacman -Sy` |

### Log Locations Across Distributions

| Log Type | Debian/Ubuntu | RHEL/CentOS | SUSE |
|----------------|--------------------------|--------------------------|--------------------------|
| System log | `/var/log/syslog` | `/var/log/messages` | `/var/log/messages` |
| Auth log | `/var/log/auth.log` | `/var/log/secure` | `/var/log/secure` |
| Kernel log | `/var/log/kern.log` | `/var/log/messages` | `/var/log/messages` |
| Apache logs | `/var/log/apache2/` | `/var/log/httpd/` | `/var/log/apache2/` |
| Nginx logs | `/var/log/nginx/` | `/var/log/nginx/` | `/var/log/nginx/` |
| MySQL logs | `/var/log/mysql/` | `/var/log/mysqld.log` | `/var/log/mysql/` |
| PostgreSQL logs | `/var/log/postgresql/` | `/var/lib/pgsql/data/log/` | `/var/lib/pgsql/data/log/` |
| Mail logs | `/var/log/mail.log` | `/var/log/maillog` | `/var/log/mail` |
| Boot log | `/var/log/boot.log` | `/var/log/boot.log` | `/var/log/boot.msg` |
| journald (all) | `journalctl` | `journalctl` | `journalctl` |

**Quick log check for any service:**

```bash
# systemd journal (most reliable on modern systems)
journalctl -u <service> --no-pager -n 50

# Tail the relevant log file
tail -50 /var/log/<service>/<logfile>
```

## Scored Service Verification & Troubleshooting

### SSH

**How the scorer tests it:** Connects to port 22 with a valid username and password. If login succeeds, the check passes.

**Verify it yourself:**

```bash
# From another host
ssh <scored-user>@<host-ip>

# Check the service is listening
ss -tlnp | grep :22
systemctl status sshd
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | sshd not running | `systemctl start sshd && systemctl enable sshd` |
| Connection refused | sshd listening on wrong port | Check `Port` in `/etc/ssh/sshd_config`, set to `22`, restart |
| Connection timeout | Firewall blocking port 22 | `iptables -I INPUT -p tcp --dport 22 -j ACCEPT` |
| Authentication failure | Wrong password | Reset password: `passwd <username>` |
| Authentication failure | PasswordAuthentication disabled | Set `PasswordAuthentication yes` in `/etc/ssh/sshd_config`, restart |
| Authentication failure | User not allowed | Check `AllowUsers` / `DenyUsers` in sshd_config |
| Authentication failure | PAM module blocking login | Check `/var/log/auth.log` or `/var/log/secure`, check `/etc/pam.d/sshd` |
| Host key error | Host key changed | Client-side issue, not a scoring problem |

**Config file:** `/etc/ssh/sshd_config`

**Restart:** `systemctl restart sshd`

### HTTP / HTTPS (Apache)

**How the scorer tests it:** HTTP GET to the host's IP on port 80 or 443. Checks that specific content (text or JavaScript) appears on the page.

**Verify it yourself:**

```bash
# Check service is running
systemctl status apache2    # Debian/Ubuntu
systemctl status httpd      # RHEL/CentOS

# Check it's listening
ss -tlnp | grep -E ':80|:443'

# Check content is being served
curl -s http://localhost/ | head -20
curl -sk https://localhost/ | head -20
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | Apache not running | `systemctl start apache2` (or `httpd`) |
| 403 Forbidden | Permissions on web root | `chmod -R 755 /var/www/html` and `chown -R www-data:www-data /var/www/html` (Debian) or `apache:apache` (RHEL) |
| 403 Forbidden | No index file | Create `index.html` in the web root |
| 500 Internal Server Error | Config syntax error | `apachectl configtest` or `httpd -t`, fix errors, restart |
| 500 Internal Server Error | Missing PHP or broken .htaccess | Check: `tail -20 /var/log/apache2/error.log` |
| Wrong content | Wrong DocumentRoot | Check `DocumentRoot` in site config, fix path |
| Wrong content | Wrong VirtualHost | `apachectl -S` to check active sites |
| HTTPS not working | Missing SSL module/cert | `a2enmod ssl` (Debian), check cert paths in ssl.conf |
| Firewall blocking | iptables dropping HTTP | Allow 80 and 443: `iptables -I INPUT -p tcp --dport 80 -j ACCEPT` |

**Config locations:**

- Debian/Ubuntu: `/etc/apache2/apache2.conf`, sites in `/etc/apache2/sites-enabled/`
- RHEL/CentOS: `/etc/httpd/conf/httpd.conf`, sites in `/etc/httpd/conf.d/`
- Web root (default): `/var/www/html/`

**Restart:** `systemctl restart apache2` or `systemctl restart httpd`

### HTTP / HTTPS (Nginx)

**How the scorer tests it:** Same as Apache -- HTTP GET checking for specific content.

**Verify it yourself:**

```bash
systemctl status nginx
ss -tlnp | grep -E ':80|:443'
curl -s http://localhost/ | head -20
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | Nginx not running | `systemctl start nginx` |
| 403 Forbidden | Permissions on web root | `chmod -R 755 /usr/share/nginx/html && chown -R nginx:nginx /usr/share/nginx/html` |
| 502 Bad Gateway | Backend (PHP-FPM, app server) is down | Check backend: `systemctl status php*-fpm`, restart it |
| Config error on restart | Syntax error in nginx.conf | `nginx -t` to find the error, fix, then restart |
| Wrong content | Wrong `root` directive | Check `root` in `/etc/nginx/sites-enabled/` or `/etc/nginx/conf.d/` |
| HTTPS not working | Missing cert or misconfigured ssl block | Check `ssl_certificate` and `ssl_certificate_key` paths |

**Config locations:**

- Main config: `/etc/nginx/nginx.conf`
- Sites: `/etc/nginx/sites-enabled/` or `/etc/nginx/conf.d/`
- Web root (default): `/usr/share/nginx/html/` or `/var/www/html/`

**Restart:** `systemctl restart nginx`

### DNS (BIND)

**How the scorer tests it:** Sends a DNS query for a specific record (A, MX, PTR, etc.) to the host's IP. If the correct answer is returned, the check passes.

**Verify it yourself:**

```bash
# Check service is running
systemctl status named      # RHEL/CentOS
systemctl status bind9      # Debian/Ubuntu

# Check it's listening
ss -tulnp | grep :53

# Test a query
dig @localhost <domain> A
dig @<host-ip> <domain> A
nslookup <domain> <host-ip>
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | BIND not running | `systemctl start named` (or `bind9`) |
| REFUSED response | Query from unauthorized IP | Add scorer IP/subnet to `allow-query` in `named.conf` |
| SERVFAIL response | Zone file syntax error | `named-checkzone <zone> <zonefile>`, fix errors |
| SERVFAIL response | Missing zone file | Check `named.conf` zone declarations, verify files exist |
| Wrong answer | Incorrect record in zone file | Edit zone file, increment serial number, restart |
| Timeout | Firewall blocking UDP/TCP 53 | `iptables -I INPUT -p udp --dport 53 -j ACCEPT && iptables -I INPUT -p tcp --dport 53 -j ACCEPT` |

**Config locations:**

- RHEL/CentOS: `/etc/named.conf`, zones in `/var/named/`
- Debian/Ubuntu: `/etc/bind/named.conf`, zones in `/etc/bind/` or `/var/cache/bind/`

**After editing a zone file:** Increment the serial number in the SOA record, then restart.

**Restart:** `systemctl restart named` or `systemctl restart bind9`

### SMTP (Postfix)

**How the scorer tests it:** Connects to port 25 and sends a test email (EHLO, MAIL FROM, RCPT TO, DATA). If the server accepts the message, the check passes.

**Verify it yourself:**

```bash
systemctl status postfix
ss -tlnp | grep :25

# Manual SMTP test
telnet localhost 25
# then type: EHLO test / MAIL FROM:<test@test.com> / RCPT TO:<user@domain> / DATA / . / QUIT

# Or with nc
echo -e "EHLO test\nMAIL FROM:<test@test.com>\nRCPT TO:<user@domain>\nDATA\nSubject: test\n\ntest\n.\nQUIT" | nc localhost 25
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | Postfix not running | `systemctl start postfix` |
| 550 relay denied | Postfix not configured to accept mail for this domain | Add domain to `mydestination` in `/etc/postfix/main.cf` |
| 550 user unknown | Recipient doesn't exist | Create the user: `useradd <username>` or add to virtual mailbox maps |
| Timeout | Firewall blocking port 25 | `iptables -I INPUT -p tcp --dport 25 -j ACCEPT` |
| Config error | Syntax error in main.cf | `postfix check` to find errors |

**Config file:** `/etc/postfix/main.cf`

**Restart:** `systemctl restart postfix`

### FTP

**How the scorer tests it:** Connects to port 21 with valid credentials, may attempt to upload or download a file.

**Verify it yourself:**

```bash
# Check service (vsftpd is most common)
systemctl status vsftpd
ss -tlnp | grep :21

# Test login
ftp <host-ip>
# Enter username and password
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | vsftpd not running | `systemctl start vsftpd` |
| 530 Login incorrect | Wrong credentials or local_enable=NO | Set `local_enable=YES` in `/etc/vsftpd.conf`, reset password |
| 500 OOPS: chroot | Home directory writable with chroot | `chmod 755 /home/<user>` or set `allow_writeable_chroot=YES` |
| Passive mode failure | Passive ports not configured/open | Set `pasv_min_port` and `pasv_max_port` in vsftpd.conf, open those ports in firewall |
| Timeout | Firewall blocking port 21 | `iptables -I INPUT -p tcp --dport 21 -j ACCEPT` |

**Config file:** `/etc/vsftpd.conf` or `/etc/vsftpd/vsftpd.conf`

**Restart:** `systemctl restart vsftpd`

### MySQL / MariaDB

**How the scorer tests it:** Connects to port 3306 with valid credentials, may query a specific database or table, checks for expected data.

**Verify it yourself:**

```bash
systemctl status mysql      # Debian/Ubuntu
systemctl status mariadb    # RHEL/CentOS or MariaDB installs
ss -tlnp | grep :3306

# Test login
mysql -u <scored-user> -p -h 127.0.0.1
# Run: SHOW DATABASES; USE <db>; SELECT * FROM <table> LIMIT 5;
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | MySQL not running | `systemctl start mysql` (or `mariadb`) |
| Access denied | Wrong password | Reset: `ALTER USER '<user>'@'%' IDENTIFIED BY '<newpass>'; FLUSH PRIVILEGES;` |
| Access denied | User restricted to localhost | `GRANT ALL ON *.* TO '<user>'@'%' IDENTIFIED BY '<pass>'; FLUSH PRIVILEGES;` |
| Connection refused on remote | bind-address = 127.0.0.1 | Change `bind-address` to `0.0.0.0` in `/etc/mysql/mysql.conf.d/mysqld.cnf` or `/etc/my.cnf`, restart |
| Database missing | Dropped or corrupted | Restore from backup: `mysql -u root -p < backup.sql` |
| Table missing or wrong data | Data deleted/modified | Restore from backup |
| Firewall blocking | iptables | `iptables -I INPUT -p tcp --dport 3306 -j ACCEPT` |

**Config locations:**

- Debian/Ubuntu: `/etc/mysql/mysql.conf.d/mysqld.cnf` or `/etc/mysql/my.cnf`
- RHEL/CentOS: `/etc/my.cnf` or `/etc/my.cnf.d/`

**Restart:** `systemctl restart mysql` or `systemctl restart mariadb`

### PostgreSQL

**How the scorer tests it:** Connects to port 5432 with valid credentials, may query specific data.

**Verify it yourself:**

```bash
systemctl status postgresql
ss -tlnp | grep :5432

# Test login
psql -U <scored-user> -h 127.0.0.1 -d <dbname>
# Run: \l (list databases), \dt (list tables), SELECT * FROM <table> LIMIT 5;
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | PostgreSQL not running | `systemctl start postgresql` |
| Connection refused on remote | Only listening on localhost | Set `listen_addresses = '*'` in `postgresql.conf`, restart |
| Authentication failed | pg_hba.conf rejects connection | Add line: `host all all 0.0.0.0/0 md5` to `pg_hba.conf`, reload |
| Authentication failed | Wrong password | As postgres user: `psql -c "ALTER USER <user> PASSWORD '<newpass>';"` |
| Database missing | Dropped or corrupted | Restore: `psql -U postgres < backup.sql` |
| Firewall blocking | iptables | `iptables -I INPUT -p tcp --dport 5432 -j ACCEPT` |

**Config locations:**

- `postgresql.conf` and `pg_hba.conf` -- find them with: `find / -name postgresql.conf 2>/dev/null`
- Common paths: `/etc/postgresql/<version>/main/` (Debian) or `/var/lib/pgsql/data/` (RHEL)

**Restart:** `systemctl restart postgresql`

### Samba (SMB)

**How the scorer tests it:** Connects to port 445, authenticates, and accesses a specific share.

**Verify it yourself:**

```bash
systemctl status smbd
ss -tlnp | grep :445

# List shares
smbclient -L //localhost -U <user>

# Access a share
smbclient //localhost/<sharename> -U <user>
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | smbd not running | `systemctl start smbd` |
| NT_STATUS_LOGON_FAILURE | Wrong Samba password | `smbpasswd -a <user>` (sets Samba password, separate from Linux password) |
| NT_STATUS_BAD_NETWORK_NAME | Share not defined | Add share definition in `/etc/samba/smb.conf` |
| NT_STATUS_ACCESS_DENIED | Permissions on share directory | `chmod 755 /path/to/share && chown <user>:<group> /path/to/share` |
| Config error | Syntax error in smb.conf | `testparm` to validate config |
| Firewall blocking | iptables | `iptables -I INPUT -p tcp --dport 445 -j ACCEPT` |

**Config file:** `/etc/samba/smb.conf`

**Restart:** `systemctl restart smbd`

### NFS

**How the scorer tests it:** Mounts an NFS export from the host and verifies access to files.

**Verify it yourself:**

```bash
systemctl status nfs-server    # or nfs-kernel-server on Debian
ss -tlnp | grep :2049

# Show exports
showmount -e localhost

# Test mount from another host
mount -t nfs <host-ip>:/export/path /mnt/test
ls /mnt/test
umount /mnt/test
```

**Common failures and fixes:**

| Symptom | Likely Cause | Fix |
|------------|----------------|---------------------------------------------|
| Connection refused | NFS server not running | `systemctl start nfs-server` |
| Mount fails: access denied | Client IP not in exports | Add client to `/etc/exports`: `/export/path <client-ip>(rw,sync,no_subtree_check)` then `exportfs -ra` |
| Permission denied on files | Root squash or UID mismatch | Add `no_root_squash` to export options (temporary fix), or fix UID/GID |
| Stale file handle | Export was changed while mounted | Re-export: `exportfs -ra`, remount on client |
| Firewall blocking | iptables | `iptables -I INPUT -p tcp --dport 2049 -j ACCEPT` |

**Config file:** `/etc/exports`

**Apply changes:** `exportfs -ra`

**Restart:** `systemctl restart nfs-server`

## Package Manager Troubleshooting

### APT (Debian/Ubuntu)

**Broken dependencies:**

```bash
# Fix broken packages
apt-get -f install

# Reconfigure packages that failed to install
dpkg --configure -a

# Clear package cache and retry
apt-get clean
apt-get update
apt-get install -y <package>
```

**Locked dpkg:**

```bash
# Check if another process is using dpkg
ps aux | grep -E "apt|dpkg"

# If no apt/dpkg process is running, remove the lock
rm /var/lib/dpkg/lock-frontend
rm /var/lib/dpkg/lock
rm /var/cache/apt/archives/lock
dpkg --configure -a
```

**Failed upgrades:**

```bash
# Hold a package to prevent it from upgrading (if an upgrade breaks things)
apt-mark hold <package>

# Downgrade to a specific version
apt-get install <package>=<version>
```

**Repository issues (no internet in competition):**

If `apt-get update` fails because repositories are unreachable, skip the update. Packages already in the local cache (`/var/cache/apt/archives/`) can still be installed:

```bash
dpkg -i /var/cache/apt/archives/<package>*.deb
```

### YUM / DNF (RHEL/CentOS/Fedora)

**Broken dependencies:**

```bash
# Check for dependency problems
yum check
# or
dnf check

# Clean cache and retry
yum clean all && yum makecache
# or
dnf clean all && dnf makecache
```

**Repo issues:**

```bash
# List configured repos
yum repolist all
# or
dnf repolist all

# Disable a broken repo temporarily
yum install --disablerepo=<reponame> <package>
# or
dnf install --disablerepo=<reponame> <package>
```

**Failed upgrades / downgrades:**

```bash
# Downgrade a package
yum downgrade <package>
# or
dnf downgrade <package>

# Lock a package version (yum-plugin-versionlock)
yum install yum-plugin-versionlock
yum versionlock add <package>
```

**RPM-level fixes:**

```bash
# Rebuild RPM database if corrupted
rpm --rebuilddb

# Force-install an RPM if dependency resolution is stuck
rpm -ivh --nodeps /path/to/<package>.rpm
```

## Service Dependency Mapping

Understanding dependencies prevents cascading failures. If a dependency goes down, everything above it in the chain fails.

### Common Dependency Chains

```
DNS Server
  +-- All hosts (name resolution)
       +-- Web Server -> Database Server (web app queries DB)
       +-- Mail Server -> DNS (MX lookups)
       +-- Domain Controller -> DNS (AD requires DNS)
            +-- All domain-joined Windows hosts (auth, GPO)

Database Server
  +-- Web Application (data backend)

DHCP Server
  +-- All hosts using dynamic IPs (no IP = no connectivity)
```

### Dependency Quick Reference

| Service | Depends On | Impact if Dependency Fails |
|------------|----------------------|---------------------------------------------|
| Web application | Database, DNS | App errors, timeouts, blank pages |
| Mail (Postfix) | DNS | Cannot resolve recipient domains |
| Domain-joined Windows | AD Domain Controller, DNS | Cannot authenticate, GPO fails |
| All hosts | DNS | Name resolution fails (use IPs as fallback) |
| All hosts | Gateway/router | No cross-subnet or external connectivity |
| NFS/SMB clients | NFS/SMB server | Mounted shares become unavailable |

### Priority Order for Restoration

When multiple services are down, restore in this order:

1. **Network connectivity** (gateway, routing, firewall)
2. **DNS** (everything depends on it)
3. **Active Directory / Domain Controller** (Windows hosts depend on it)
4. **Database servers** (web apps depend on them)
5. **Individual scored services** (web, mail, FTP, etc.)

## Change Management Discipline

### Copy Before Edit

**Every time you edit a config file:**

```bash
cp /etc/<service>/config.conf /etc/<service>/config.conf.bak.$(date +%Y%m%d-%H%M)
```

This takes 2 seconds and can save 20 minutes of recovery.

### Test Before Commit

Many services have a config-test command. Use it before restarting:

| Service | Config Test Command |
|------------|---------------------------------------------|
| Apache | `apachectl configtest` or `httpd -t` |
| Nginx | `nginx -t` |
| BIND (named) | `named-checkconf` and `named-checkzone <zone> <file>` |
| Postfix | `postfix check` |
| Samba | `testparm` |
| sshd | `sshd -t` |

### Verify After Every Change

After restarting a service, confirm it is scoring:

```bash
# 1. Check service is running
systemctl status <service>

# 2. Check it's listening on the expected port
ss -tlnp | grep :<port>

# 3. Perform the same check the scorer does
#    SSH: ssh <user>@<ip>
#    HTTP: curl http://<ip>/
#    DNS: dig @<ip> <record>
#    SMTP: nc <ip> 25
#    MySQL: mysql -u <user> -p -h <ip>

# 4. Check the scoring dashboard
```

### One Change at a Time

- Make one change, test, confirm scoring.
- If scoring breaks, you know exactly what caused it.
- If you make five changes and scoring breaks, you don't know which one is responsible.
