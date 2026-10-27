# Project 01: Centralized Log Server


## 1. Problem Statement

A small company runs 4 servers (web, database, mail, and a backup server). When a service fails, engineers must log into each machine individually and search logs — slow and error-prone. The company needs a **centralized log server** that collects logs from all servers in real time, stored on dedicated resilient storage with enforced rotation to prevent disk exhaustion.

**Business requirements:**

| Requirement 		| Detail |
|--------------------|-----------------|
| Collect logs from 4 clients | Over TCP (reliable, encrypted tunnel available) |
| Dedicated storage | Separate LVM volume, 5 GB, ext4 |
| Log rotation | Keep 7 days; compressed archives |
| Access control | Log analysts group can read, others denied |
| SELinux | Enforcing mode — no permissive workarounds |
| Firewall | Only TCP 514 allowed from client subnet |

## 2. Architecture

```
        ┌─────────────┐      ┌─────────────┐
        │  web-srv    │      │  db-srv     │
        └──────┬──────┘      └──────┬──────┘
               │ rsyslog → TCP 514  │
        ┌──────┴──────┐      ┌──────┴──────┐
        │ mail-srv    │─────▶│  LOG-SRV    │
        └─────────────┘      │ (this proj) │
                             ├─────────────┤
                             │ LVM: /log   │  5 GB ext4
                             │ Rotation    │  7 days, gzip
                             └─────────────┘
```

**Environment:** 5 VMs (RHEL 9.x) on KVM/VirtualBox, one subnet `192.168.56.0/24`. Server: 2 vCPU, 2 GB RAM.

## 3. Build Steps

### 3.1 Server — Dedicated Storage (LVM)

```bash
# Identify the new disk
lsblk                        # /dev/sdb found

# Create PV, VG, LV
pvcreate /dev/sdb
vgcreate logvg /dev/sdb
lvcreate -L 5G -n loglv logvg

# Filesystem, mount point, persistence
mkfs.ext4 /dev/logvg/loglv
mkdir /log
vim /etc/fstab               # add: /dev/logvg/loglv /log ext4 defaults 0 0
mount -a                     # verify: df -h /log
```

**Why LVM:** the company may need more space later — `lvextend` + `resize2fs` grows it online without downtime.

### 3.2 Server — rsyslog Configuration

Edit `/etc/rsyslog.d/10-remote.conf`:

```bash
# Load TCP server module
module(load="imtcp")

# Listen on 514
input(type="imtcp" port="514")

# Store per-client, per-service logs
template(name="RemoteLogs" type="string"
    string="/log/%HOSTNAME%/%PROGRAMNAME%.log")

if ($fromhost-ip != "127.0.0.1") then {
    action(type="omfile" dynaFile="RemoteLogs")
    stop
}
```

```bash
systemctl restart rsyslog
systemctl enable rsyslog
```

### 3.3 Firewall + SELinux

```bash
firewall-cmd --permanent --add-port=514/tcp
firewall-cmd --reload
firewall-cmd --list-ports

getenforce                   # must say: Enforcing

# rsyslog is allowed to listen on 514 under the rsyslogd_port_t type:
semanage port -a -t syslogd_port_t -p tcp 514
# (or use: setsebool -P nis_enabled on — NOT used here; explicit port is cleaner)
```

### 3.4 Clients — Forward Logs

Edit `/etc/rsyslog.d/00-forward.conf` on each client:

```bash
*.* @@192.168.56.10:514     # @@ = TCP, @ = UDP
```

```bash
systemctl restart rsyslog
logger -p local0.info "test message from $(hostname)"
```

### 3.5 Log Rotation (server)

Create `/etc/logrotate.d/remote`:

```bash
/log/*/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
    create 0640 root loganalysts
}
```

```bash
logrotate -d /etc/logrotate.d/remote    # dry run, verify no errors
```

### 3.6 Access Control

```bash
groupadd loganalysts
useradd -G loganalysts analyst1
chown -R root:loganalysts /log
chmod -R 640 /log/*/*
```

## 4. Verification (Proof It Works)

```bash
# On server — logs arriving:
ls /log/web-srv/
cat /log/web-srv/sshd.log

# From client:
logger "VERIFICATION TEST 2026-10-27"
# On server:
grep VERIFICATION /log/db-srv/*.log
```

Expected output (screenshot in `/screenshots/`):
- `systemctl status rsyslog` → `active (running)`
- `getenforce` → `Enforcing`
- Client log file present with test message

## 5. Troubleshooting Log (real issues I hit)

| Issue | Symptom | Root cause | Fix |
|---|---|---|---|
| Clients' logs not arriving | No `/log/*/` dirs created | firewalld blocking 514 | `firewall-cmd --add-port=514/tcp` |
| rsyslog denied port 514 | AVC denials in `/var/log/audit/audit.log` | SELinux: port not in `syslogd_port_t` | `semanage port -a -t syslogd_port_t -p tcp 514` |
| Log files unreadable by analysts | Permission denied | Default umask 022, group not set | `chmod 640` + `create` directive in logrotate |

## 6. What This Demonstrates (RHCSA Skill Map)

| Exam Objective | Where Used |
|---|---|
| Manage storage / LVM | §3.1 |
| Manage and configure services (systemd) | §3.2 |
| Firewalld configuration | §3.3 |
| SELinux management (enforcing, port labels) | §3.3, §5 |
| Users, groups, permissions, ACLs | §3.6 |
| Schedule tasks (logrotate via cron/cron.daily) | §3.5 |
| Analyze and store logs | Entire project |

## 7. Possible Extensions

- TLS encryption for log transport (cert-based, port 6514)
- Hourly rotation for high-volume hosts
- Ship critical alerts to a monitoring dashboard (Grafana + Loki)
