# Linux Production Support Command Cheat Sheet

Use this as a fast lookup. Replace placeholders such as `<service>`, `<host>`, `<port>`, `<user>`, and `<path>` with approved values. Commands marked **change-capable** can alter production state; use only with authorization.

## 1. Confirm where you are

```bash
hostname
hostname -f
whoami
id
pwd
date -Is
uname -a
cat /etc/os-release
uptime
```

## 2. Navigation and files

| Need | Command |
|---|---|
| List including hidden files | `ls -lah <path>` |
| Show current directory | `pwd` |
| Identify file type | `file <path>` |
| File metadata and timestamps | `stat <path>` |
| Read a file safely | `less <file>` |
| First/last lines | `head -n 50 <file>` / `tail -n 100 <file>` |
| Follow a rotating log | `tail -F <logfile>` |
| Copy while preserving attributes | `cp -a <src> <dest>` |
| Compare two files/configs | `diff -u <old-file> <new-file>` |
| Verify file checksum | `sha256sum <file>` |
| Copy over SSH (approved path only) | `scp <file> <user>@<host>:<destination>` |
| Preview an rsync transfer | `rsync -avhn -- <src>/ <dest>/` (`-n` is dry-run) |
| Find by name | `find <path> -type f -name '*.log' -print` |
| Find recently modified files | `find <path> -type f -mmin -60 -print` |
| Find large files (GNU find) | `find <path> -xdev -type f -size +500M -print` |
| Search text | `grep -RIn -- 'pattern' <path>` |
| Count lines | `wc -l <file>` |
| Show symlink target | `readlink -f <link>` |
| Inspect each directory in a path | `namei -l <path>` |
| List archive contents without extracting | `tar -tf <archive.tar.gz>` |

**Tip:** Quote paths and patterns. Use `--` before user-controlled filenames where supported so a filename beginning with `-` is not treated as an option.

## 3. Permissions, ownership, ACLs

```bash
ls -ld <path>
stat <path>
namei -l <path>
getfacl <path>                    # ACL inspection
```

Common permission values:

| Mode | Meaning | Typical use |
|---|---|---|
| `600` | owner read/write | Private file |
| `640` | owner read/write, group read | Group-readable config/data file |
| `644` | owner read/write, everyone else read | Non-secret text/config file |
| `700` | owner full access only | Private directory/script |
| `750` | owner full, group read/execute | Restricted app directory |
| `755` | owner full, others read/execute | Publicly traversable directory or executable |
| `2770` | group inheritance on directory plus owner/group full | Shared group directory (review ACLs too) |

**Change-capable examples — lab or approved change only:**

```bash
chmod 640 <file>
chmod 750 <directory>
chown <user>:<group> <path>
chgrp <group> <path>
setfacl -m u:<user>:r-- <file>
setfacl -m g:<group>:r-x <directory>
umask
```

Do not copy a mode from another server without checking the intended owner, group, directory traversal, service identity, ACL, and security requirements. Avoid recursive permission changes unless a reviewed runbook explicitly requires them.

## 4. Users, groups, accounts, sudo

**Read-only inspection:**

```bash
whoami
id <user>
groups <user>
getent passwd <user>
getent group <group>
sudo -l
passwd -S <user>                 # status format varies by distribution
chage -l <user>
last -a | head
```

**Change-capable examples — lab or approved identity-management workflow only:**

```bash
sudo groupadd <group>
sudo useradd -m -s /bin/bash <user>
sudo passwd <user>
sudo usermod -aG <group> <user>   # -a is important: append to existing groups
sudo gpasswd -d <user> <group>    # remove one supplementary group
sudo passwd -l <user>             # lock password authentication
sudo passwd -u <user>             # unlock; verify policy first
sudo usermod -s /usr/sbin/nologin <user>  # path varies by distribution
sudo chage -E YYYY-MM-DD <user>   # set account expiry; confirm policy
sudo visudo                     # safely validate sudoers edits; authorized admins only
```

Notes:

- In enterprise environments, access is often controlled through LDAP/Active Directory/SSO/IAM. Use the approved identity workflow rather than creating local accounts.
- A user normally needs to log out and back in to receive newly added supplementary group membership.
- `usermod -G` without `-a` can replace existing supplementary groups. Prefer `-aG` when appending.
- Never edit `/etc/sudoers` directly with a normal editor. Use `visudo` and the organization's approved role model.
- Account deletion (especially `userdel -r`) is destructive. Check ownership, running processes, scheduled jobs, data retention, and approval first.

## 5. Processes and services

```bash
ps -ef | grep -i '[a]ppname'
pgrep -af '<pattern>'
top
ps -eo pid,ppid,stat,%cpu,%mem,etime,cmd --sort=-%cpu | head
systemctl status <service> --no-pager
systemctl is-active <service>
systemctl is-enabled <service>
systemctl list-units --type=service --state=failed
systemctl cat <service>
systemctl show <service> -p User -p Group -p ExecStart -p Restart
```

**Change-capable — only with approval:**

```bash
sudo systemctl reload <service>
sudo systemctl restart <service>
sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl enable <service>
sudo systemctl daemon-reload   # rereads unit definitions; does not restart the service
kill -TERM <pid>               # graceful signal; verify impact first
```

Do not assume `reload` is supported. Check the service's documentation/unit file. A restart can interrupt requests or cause a wider outage.

## 6. Logs and evidence

```bash
journalctl -u <service> --since '30 minutes ago' --no-pager
journalctl -u <service> --since '2026-11-14 18:00:00' --until '2026-11-14 19:00:00' --no-pager
journalctl -p err --since today --no-pager
journalctl -k --since today --no-pager
tail -F <application-log>
grep -nEi 'error|exception|timeout|refused|denied' <logfile>
zgrep -nEi 'error|exception' <rotated-log.gz>
```

For JSON logs, if `jq` is installed:

```bash
jq -c 'select(.level == "Error")' <app.jsonl>
```

Be alert for secrets and personal/customer data in logs. Capture only the minimum evidence needed and redact before sharing.

## 7. Networking, DNS, HTTP, TLS

```bash
ip -br addr
ip route
ip neigh
ss -lntup
ss -tan
getent hosts <hostname>
dig <hostname>
nc -vz <hostname> <port>
curl -I --connect-timeout 5 --max-time 10 https://<hostname>/health
curl -sS -o /dev/null -w 'http=%{http_code} connect=%{time_connect}s total=%{time_total}s\n' --connect-timeout 5 --max-time 10 https://<hostname>/health
openssl s_client -connect <hostname>:443 -servername <hostname> </dev/null
```

If a local health endpoint is documented and safe to call:

```bash
curl -sS -i --max-time 5 http://127.0.0.1:<port>/health
```

Interpretation:

- DNS lookup failure: investigate resolver, DNS record, search domain, and recent DNS changes.
- Connection refused: target was reachable but no process accepted the connection (or a device actively rejected it).
- Timeout: packets or responses may be dropped, filtered, or stuck; check route/firewall/load balancer and target health.
- TLS error: check certificate chain, expiry, hostname/SNI, trust store, and system time.
- HTTP 401/403: authentication/authorization or policy issue; do not bypass controls.
- HTTP 5xx: investigate proxy/gateway and application/dependency logs.

`ping` is not proof that an application port works; ICMP may be blocked while TCP works.

## 8. Disk, inodes, CPU, memory, I/O

```bash
df -hT
df -ih
du -xhd1 /var 2>/dev/null | sort -h
lsblk -f
findmnt
free -h
uptime
vmstat 1 5
iostat -xz 1 5                 # if sysstat is installed
dmesg -T | tail -n 100
journalctl --disk-usage
sudo lsof +L1                 # deleted files still held open; needs privilege for full view
```

Do not delete logs or application files just because a filesystem is full. Identify the mount, owner, file purpose, retention policy, log rotation, backup status, open file handles, and approval path first.

## 9. Scheduled jobs and package inventory

```bash
crontab -l
sudo crontab -l -u <user>       # only if authorized
systemctl list-timers --all
atq
```

Package inventory (read-only):

```bash
dpkg -l                         # Debian/Ubuntu
apt list --installed            # Debian/Ubuntu
rpm -qa                         # RHEL-compatible
dnf list installed               # RHEL-compatible
```

Installing/upgrading packages on production is a controlled change. Do not use package-manager update commands as a first troubleshooting step.

## 10. Docker and Kubernetes (only if used by your team)

Docker inspection:

```bash
docker ps
docker ps -a
docker logs --since 30m <container>
docker inspect <container>
docker stats --no-stream
```

Kubernetes inspection:

```bash
kubectl config current-context
kubectl get namespaces
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --since=30m
kubectl logs <pod> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Always verify cluster context and namespace before running commands. `kubectl exec`, `delete`, `rollout restart`, `scale`, `apply`, and `patch` can change state; use only with approval.

## 11. Shell safety and useful patterns

```bash
# Print each command and stop a script on errors; use in scripts you own
set -Eeuo pipefail

# Search an exact string in recent logs
grep -nF -- 'exact message' <logfile>

# Count error messages by matching line
grep -i 'error' <logfile> | wc -l

# List files modified in the past hour (GNU find)
find <path> -type f -mmin -60 -print

# Show top 10 largest immediate subdirectories (GNU du)
du -xhd1 <path> 2>/dev/null | sort -h | tail -n 10
```

Use shell scripts with care: `set -e` and pipelines have edge cases; test in a lab and review exit handling before operational use. Avoid `eval` and avoid building commands from untrusted input.

## 12. Before executing a risky command

Ask yourself:

1. Am I on the right host/environment/cluster/context?
2. Is this command read-only or can it change state?
3. What is the expected impact, blast radius, and rollback?
4. Is this action permitted by the incident/change process?
5. Have I captured the evidence needed before the change?
6. Can I target one file/process/service instead of using a broad recursive action?
7. How will I verify recovery and communicate the result?