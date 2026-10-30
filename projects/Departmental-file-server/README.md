```markdown
# Project: Departmental File Server

---

## 1. Problem Statement

A small company runs three departments — **engineering**, **sales**, and **hr** — that share files over the network but have no enforced separation. Employees accidentally edit each other's files, a single team can fill the disk, and there is no way to give a manager cross-department read access without making them a full member. The company needs a **centralized file server** with isolated department shares, enforced collaboration within each team, per-department quotas, and a public read-only share for company-wide notices.

**Business requirements:**

| Requirement | Detail |
|---|---|
| Isolated department shares | engineering, sales, hr — each writable only by its own group |
| Intra-team collaboration | Files created by one member editable by all members of the same group |
| Public share | Readable by all staff, writable only by IT admins |
| Per-department quota | 2 GB hard limit per department share |
| Cross-department read access | HR manager can read sales share without joining the sales group |
| SELinux | Enforcing mode — correct NFS booleans and file contexts |
| Firewall | Only NFS-related services allowed from the client subnet |
| Client access | NFS mounts from RHEL 10 clients |

## 2. Architecture

```
                ┌──────────────┐       ┌──────────────┐
                │  client01    │       │  client02    │
                │ (engineering)│       │ (sales/hr)   │
                └──────┬───────┘       └──────┬───────┘
                       │ NFSv4                 │ NFSv4
                       ▼                       ▼
                ┌─────────────────────────────────────┐
                │           srv01 (file server)       │
                │                                     │
                │  /srv/shares  ← XFS, gquota         │
                │   ├── engineering  2770 eng   2G    │
                │   ├── sales        2770 sales 2G    │
                │   ├── hr           2770 hr    2G    │
                │   └── public       2775 itadmins    │
                │                                     │
                │  ACL: u:dave:rx on sales            │
                │  SELinux: public_content_rw_t       │
                │  Firewall: nfs, mountd, rpc-bind    │
                └─────────────────────────────────────┘
```

**Environment:** 3 VMs (RHEL 10) on KVM/VirtualBox, one subnet `192.168.56.0/24`. Server: 2 vCPU, 2 GB RAM, one additional disk (`/dev/sdb`) dedicated to `/srv/shares`.

**Groups:** `engineering`, `sales`, `hr`, `itadmins`
**Users:** `alice` (engineering), `bob` (sales), `carol` (hr), `dave` (itadmins + hr manager)

## 3. Build Steps

### 3.1 Server — Dedicated XFS Filesystem with Quotas

```bash
# Identify the new disk
lsblk                        # /dev/sdb found

# Format XFS and mount with group quota support
mkfs.xfs /dev/sdb
mkdir -p /srv/shares
vim /etc/fstab               # add:
# /dev/sdb  /srv/shares  xfs  defaults,gquota  0 0
mount -a
mount | grep /srv/shares     # confirm gquota is active
```

**Why gquota at mount time:** XFS quotas cannot be enabled on an already-mounted filesystem without a remount that includes the option — and even then, a reboot is often required. Setting it in fstab before first mount avoids the trap.

### 3.2 Server — Groups, Users, and Directory Structure

```bash
groupadd engineering
groupadd sales
groupadd hr
groupadd itadmins

useradd -G engineering alice
useradd -G sales       bob
useradd -G hr          carol
useradd -G itadmins,hr dave

for d in engineering sales hr public; do
  mkdir -p /srv/shares/$d
done

chown root:engineering /srv/shares/engineering
chown root:sales       /srv/shares/sales
chown root:hr          /srv/shares/hr
chown root:itadmins    /srv/shares/public

chmod 2770 /srv/shares/engineering
chmod 2770 /srv/shares/sales
chmod 2770 /srv/shares/hr
chmod 2775 /srv/shares/public
```

**Why `2770`:** the leading `2` (setgid bit) makes new files and subdirectories inherit the parent directory's group. Without it, Alice's files would be owned by group `alice` and Bob — even though both are in `engineering` — could not edit them. This is the single most important setting in the project; the `770` provides the isolation boundary, the `2` provides collaboration.

**Why `2775` for public:** all users can read and traverse, but only `itadmins` (group owner with write) can modify. The setgid bit keeps new files manageable by admins.

### 3.3 Server — ACL for the HR Manager

`dave` is the HR manager and must read `/srv/shares/sales`, but must **not** be a member of the `sales` group (membership would grant write access via the 2770 bit). The solution is a named-user POSIX ACL that overrides the group class for one user:

```bash
setfacl -m u:dave:rx /srv/shares/sales
setfacl -d -m u:dave:rx /srv/shares/sales    # default ACL propagates to new files
getfacl /srv/shares/sales                    # verify user:dave:r-x is present
```

The `getfacl` output will show `user:dave:r-x` alongside `group::---` — a clean demonstration that named-user ACL entries take precedence over traditional group bits.

### 3.4 Server — XFS Group Quotas

```bash
xfs_quota -x -c 'limit -g bhard=2g engineering' /srv/shares
xfs_quota -x -c 'limit -g bhard=2g sales'       /srv/shares
xfs_quota -x -c 'limit -g bhard=2g hr'          /srv/shares

xfs_quota -x -c 'report -gh' /srv/shares        # confirm limits
```

### 3.5 Server — NFS Exports

```bash
dnf install -y nfs-utils
systemctl enable --now nfs-server

cat > /etc/exports <<'EOF'
/srv/shares/engineering  192.168.56.0/24(rw,sync,root_squash)
/srv/shares/sales        192.168.56.0/24(rw,sync,root_squash)
/srv/shares/hr           192.168.56.0/24(rw,sync,root_squash)
/srv/shares/public       192.168.56.0/24(ro,sync,root_squash)
EOF

exportfs -rav
exportfs -v                  # verify
```

**Why `ro` on public:** even if a non-admin client somehow guessed the path, the export itself blocks writes. Isolation is enforced at two layers — filesystem (`2775` + group ownership) and export (`ro`).

### 3.6 Server — Firewall

NFS needs **three** services, not just `nfs`:

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=mountd
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --reload
firewall-cmd --list-services
```

### 3.7 Server — SELinux

```bash
getenforce                   # must say: Enforcing

setsebool -P nfs_export_all_rw on
setsebool -P nfs_export_all_ro on

semanage fcontext -a -t public_content_rw_t '/srv/shares(/.*)?'
restorecon -Rv /srv/shares
```

### 3.8 Clients — Mount and Verify

```bash
# client01 (engineering)
dnf install -y nfs-utils
mount -t nfs 192.168.56.10:/srv/shares/engineering /mnt/eng

# client02 (sales/hr)
mount -t nfs 192.168.56.10:/srv/shares/sales   /mnt/sales
mount -t nfs 192.168.56.10:/srv/shares/public  /mnt/public
```

Add equivalent lines to each client's `/etc/fstab` for persistence.

## 4. Verification (Proof It Works)

| # | Test | Where | Expected |
|---|------|-------|----------|
| 1 | setgid inheritance | on srv01: `touch /srv/shares/engineering/a.txt; ls -l` | file group is `engineering` |
| 2 | Cross-dept isolation | as `alice`: `ls /srv/shares/sales` | `Permission denied` |
| 3 | Intra-team collaboration | `bob` creates a file, second sales user edits it | both succeed |
| 4 | Public read | as `alice`: `cat /mnt/public/notice.txt` | succeeds |
| 5 | Public write denied | as `alice`: `touch /mnt/public/x` | `Read-only file system` or `Permission denied` |
| 6 | Public write as admin | as `dave`: `touch /mnt/public/x` | succeeds |
| 7 | ACL read for HR manager | as `dave`: `ls /srv/shares/sales` | succeeds |
| 8 | ACL does not grant write | as `dave`: `touch /srv/shares/sales/z` | `Permission denied` |
| 9 | Quota enforced | as `bob`: `dd if=/dev/zero of=/srv/shares/sales/big bs=1M count=100` in a loop | `Disk quota exceeded` around 2 GB |

**Quota test detail:** `bob` will hit a *file too large* error before the quota if his base file-size limit is small. Run the `dd` in a loop of 100 MB files until the group reaches 2 GB — that produces the clean `Disk quota exceeded` message.

Expected output (screenshots in `/screenshots/`):
- `mount | grep /srv/shares` → shows `gquota`
- `getfacl /srv/shares/sales` → shows `user:dave:r-x`
- `exportfs -v` → all four shares listed
- `firewall-cmd --list-services` → includes `nfs mountd rpc-bind`
- `xfs_quota -x -c 'report -gh' /srv/shares` → limits shown at 2 GB
- Quota test → `Disk quota exceeded`

## 5. Troubleshooting Log (real issues I hit)

| Issue | Symptom | Root cause | Fix |
|---|---|---|---|
| Teammates can't edit each other's files | Files created with personal group | setgid bit missing on directory | `chmod 2770 <dir>` — verify `s` in `ls -ld` |
| Client hangs on mount, `access denied` | Mount times out or is denied | Firewall missing `mountd` or `rpc-bind` | `firewall-cmd --add-service=mountd --add-service=rpc-bind --reload` |
| Writes fail despite 2770 | Permission denied on write | SELinux `nfs_export_all_rw` boolean off | `setsebool -P nfs_export_all_rw on` |
| `xfs_quota: Function not implemented` | Cannot set limits | `gquota` not present at mount time | Confirm `gquota` in fstab **and** in `mount` output; remount does not enable quotas — reboot |
| `dave` cannot read sales after `setfacl` | ACL appears set but no access | ACL applied to wrong path, or client mount is stale | `getfacl /srv/shares/sales` on the **server**; re-check after client remount |

## 6. What This Demonstrates (RHCSA Skill Map)

| Exam Objective | Where Used |
|---|---|
| Manage storage (XFS, quotas) | §3.1, §3.4 |
| Create and configure file systems | §3.1 |
| Create, delete, and modify users and groups | §3.2 |
| Manage file permissions and ownership (setgid) | §3.2, §3.3 |
| Manage POSIX ACLs | §3.3 |
| Configure and use NFS | §3.5, §3.8 |
| Configure firewalld | §3.6 |
| Manage SELinux (booleans, contexts) | §3.7, §5 |
| Schedule and persist mounts via fstab | §3.1, §3.8 |

## 7. Possible Extensions

- **Samba** — expose the same shares to Windows clients via `smb.conf`, mirroring the permission model
- **autofs** — replace static client mounts with automount so shares mount on demand
- **LUKS** — encrypt `/srv/shares` at rest while retaining the quota setup
- **Bridging into Project 03** — the department shares become the backup targets for the automated backup & rotation project
```
