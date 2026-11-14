# 7-Day Linux Production Support Learning Plan

**Target learner:** Experienced application developer (around 10 years) moving into production support.

**Suggested commitment:** 2–3 focused hours each day. Spend roughly 30% reading, 50% practising in a lab, and 20% explaining what you found as though you were updating an incident ticket.

## Day 1 — Linux orientation, shell, files, and text processing

**Outcome:** Navigate a server, inspect files safely, find configuration/log files, and extract useful evidence.

Learn and practise:

- Host and OS details: `hostname`, `uname -a`, `cat /etc/os-release`, `uptime`, `date -Is`, `whoami`, `id`.
- Navigation: `pwd`, `ls -lah`, `cd`, `tree` (if installed), absolute vs relative paths.
- Files: `touch`, `file`, `stat`, `cat`, `less`, `head`, `tail -n`, `tail -F`, `cp -a`, `mv`, `mkdir -p`, `ln -s`, `readlink -f`.
- Search: `find`, `grep -RIn`, `locate` (if installed), `xargs` (understand quoting first).
- Text tools: `cut`, `sort`, `uniq -c`, `wc -l`, `awk`, `sed` for inspection, and `jq` for JSON.
- Archives and transfer: `tar -tf`, `tar -czf`, `tar -xzf`, `gzip`, `zgrep`, `sha256sum`, `scp`, and `rsync` (understand `--dry-run` before transfer).

**Lab:** Create a sample application log with INFO/WARN/ERROR lines. Find the most recent errors, count error categories, inspect a JSON config with `jq`, and archive the lab logs.

**Production-support drill:** A user reports that a configuration file is missing. Show how you would verify the path, check parent-directory traversal permissions, search likely locations, and record file metadata without changing anything.

**Checkpoint:** Explain the difference between `cat`, `less`, `tail -f`, and `tail -F`; explain why `find` paths and quoting matter.

## Day 2 — Users, groups, ownership, permissions, ACLs, and sudo

**Outcome:** Diagnose access-denied issues without over-granting privileges.

Learn and practise:

- Identity: `whoami`, `id`, `groups`, `getent passwd <user>`, `getent group <group>`, `last`.
- File access: `ls -l`, `stat`, `namei -l <path>`, `getfacl <path>`.
- Permission model: owner/group/other; read/write/execute; directory execute (traverse); octal modes such as `640`, `644`, `750`, `755`.
- Ownership: `chown`, `chgrp`; permissions: `chmod`; default permissions: `umask`.
- ACLs: `getfacl`, `setfacl`; default directory ACLs; `setfacl --restore` where supported.
- Group collaboration: set-group-ID bit on a shared directory (`chmod 2770` in a lab) and default ACLs.
- Sudo: `sudo -l`, `visudo`, `/etc/sudoers.d/` (inspect only unless authorized).
- Account state: `passwd -S`, `chage -l`; account creation/locking/group membership commands in the lab only.

**Lab:** Create two lab users and a shared group. Give the group access to one shared directory while keeping another directory private. Diagnose an access issue with `namei -l` and `getfacl` before changing anything.

**Production-support drill:** An application process cannot read a mounted configuration file. Check the service account, supplementary groups, each directory in the path, ACLs, mount state, and whether a deployment changed ownership.

**Checkpoint:** Explain why `chmod 644` on a file does not make its parent directories traversable; explain why `usermod -aG` is safer than `usermod -G` when adding a group.

## Day 3 — Processes, ports, systemd services, and logs

**Outcome:** Determine whether an application is running, why it exited, and which process owns a port.

Learn and practise:

- Processes: `ps aux`, `ps -ef`, `pgrep -af`, `top`, `htop`, `pstree`, `pidof`.
- Signals: `kill -TERM <pid>`, `kill -HUP <pid>` only when the program documents it; understand `kill -9` as a last resort.
- Open files/sockets: `lsof -p <pid>`, `lsof -i`, `fuser -v <port>/tcp`.
- Ports: `ss -lntup`, `ss -s`.
- systemd: `systemctl status`, `is-active`, `is-enabled`, `list-units`, `list-timers`, `cat`, `show`, `journalctl`.
- Logs: `journalctl -u <service> --since '30 minutes ago'`, `journalctl -p err`, `tail -F`, `grep`.
- Understand the difference between application logs, stdout/stderr captured by systemd, and OS logs.

**Lab:** Start a test service (or use a lab web server), inspect status and logs, identify its listening port, and explain the last failure shown in the journal.

**Production-support drill:** A health check returns 503. Check service state, recent logs, listening sockets, process health, and whether the application is listening on the expected interface/port. Do not restart it until you have captured evidence and obtained approval.

**Checkpoint:** Explain `systemctl reload` vs `restart` vs `daemon-reload`; explain why a process can be running while the application is still unhealthy.

## Day 4 — Networking, DNS, HTTP, and TLS

**Outcome:** Separate DNS, routing, TCP, TLS, proxy, and application failures.

Learn and practise:

- Interfaces/routes: `ip -br addr`, `ip link`, `ip route`, `ip neigh`.
- Listening connections: `ss -lntup`, `ss -tan`.
- DNS: `getent hosts`, `dig`, `nslookup` (if installed), `/etc/resolv.conf` inspection.
- Connectivity: `ping` (where allowed), `nc -vz <host> <port>`, `traceroute`/`tracepath` if installed.
- HTTP: `curl -I`, `curl -v`, `curl --connect-timeout`, `curl --max-time`, `curl -sS -o /dev/null -w`.
- TLS: `openssl s_client -connect <host>:443 -servername <host>`, `curl -v` certificate output, certificate expiry checks with `openssl x509` when given a certificate file.
- Proxy: inspect `env | grep -i proxy` carefully; never publish credentials embedded in proxy URLs.

**Lab:** For a lab endpoint, test DNS resolution, TCP port reachability, TLS negotiation, and HTTP status separately. Compare a direct local health endpoint with the external URL.

**Production-support drill:** One service cannot call another service. Write down tests that distinguish DNS failure, route/firewall failure, refused connection, timeout, TLS hostname/expiry problem, and HTTP 401/403/5xx.

**Checkpoint:** Explain the difference between connection refused and connection timed out; explain why `ping` failing does not prove the application port is unavailable.

## Day 5 — Disk, inodes, CPU, memory, and system health

**Outcome:** Identify common host resource exhaustion symptoms and collect evidence before cleanup.

Learn and practise:

- Disk space: `df -hT`, `du -xhd1 <path>` (GNU `du`), `lsblk -f`, `findmnt`.
- Inodes: `df -ih`; many tiny files can exhaust inodes while disk bytes remain available.
- Large files: `find <path> -xdev -type f -size +500M -printf '%s %p\n'` (GNU find).
- Deleted-but-open files: `sudo lsof +L1` (read-only inspection; requires privilege for full visibility).
- CPU/load: `uptime`, `top`, `ps -eo pid,ppid,stat,%cpu,%mem,etime,cmd --sort=-%cpu`.
- Memory: `free -h`, `vmstat 1 5`, `ps --sort=-%mem`.
- I/O/kernel: `iostat -xz 1 5` (sysstat package), `dmesg -T`.
- systemd journal footprint: `journalctl --disk-usage`.

**Lab:** Fill a lab filesystem or create many small files in a disposable volume. Observe how `df`, `du`, and `df -i` differ. Do not run space-filling tests on production.

**Production-support drill:** Disk usage is 100%. Find the affected mount, largest directories, large or recently changed logs, inode usage, and deleted-open files. Propose a safe cleanup/rotation plan without deleting anything.

**Checkpoint:** Explain why `du` totals may not match `df`; explain what happens when a deleted file is still held open by a process.

## Day 6 — Scheduling, packages, security basics, containers, and Kubernetes

**Outcome:** Inspect recurring jobs and understand deployment/runtime context.

Learn and practise:

- Cron: `crontab -l`, `crontab -l -u <user>` (authorized), `/etc/cron.*` inspection, `systemctl list-timers`.
- Package inventory: Debian/Ubuntu `dpkg -l`, `apt list --installed`; RHEL family `rpm -qa`, `dnf list installed`.
- SSH basics: `ssh -v`, `ssh-keygen -lf <public-key>`, inspect `sshd` logs; never copy private keys or weaken SSH security to bypass access problems.
- Security evidence: `journalctl`, `last`, `getent`, `sudo -l`; use your organization's endpoint/security tooling as the source of truth.
- Environment: `env` (redact output), `printenv <NAME>`, `ulimit -a`, `timedatectl`.
- Docker where used: `docker ps`, `docker logs --since`, `docker inspect`, `docker stats`; treat `docker exec` as an interactive change-capable action.
- Kubernetes where used: `kubectl config current-context`, `kubectl get pods -n <namespace>`, `kubectl describe pod`, `kubectl logs --since=30m`, `kubectl logs --previous`, `kubectl get events --sort-by=.lastTimestamp`.

**Lab:** Inspect a cron/timer definition in a lab. If your team uses containers, inspect one container's status/logs. If your team uses Kubernetes, identify a pod restart reason from `describe` and previous logs.

**Production-support drill:** A background job did not run. Check schedule, timezone, account, permissions, environment variables, prior execution logs, and dependencies.

**Checkpoint:** Explain why container logs may differ from host logs; verify the Kubernetes context and namespace before any cluster command.

## Day 7 — Incident simulations and support readiness

**Outcome:** Run a structured investigation and communicate clearly under pressure.

Complete at least three simulations:

1. **Application down:** Service status, recent logs, process, port, health endpoint, recent deployment/config changes.
2. **Permission denied:** Effective user, groups, path traversal, ownership, mode bits, ACL, mounted volume, service unit identity.
3. **High disk / no space:** Mount, bytes, inodes, top directories, deleted-open files, log rotation, approved cleanup.
4. **Dependency timeout:** DNS, route, TCP, TLS, proxy, remote service health, request/correlation ID.
5. **Scheduled job failure:** Timer/cron, execution user, environment, logs, exit code, dependency status.

For each simulation, write a short ticket update:

- Impact and scope
- First observed time and timezone
- Host/service/environment
- Evidence and commands used
- Most likely failing layer (and confidence)
- What has not yet been ruled out
- Action taken or approval required
- Verification and next update

**Final self-assessment:** You are ready for supervised support when you can investigate the above scenarios without guessing, avoid risky actions, explain each command, preserve evidence, and escalate with a concise problem statement.

## Daily habit after the first week

Spend 15 minutes reviewing one real or simulated incident. Add only validated, environment-approved commands to the repository. Record distribution-specific differences and link to the team's official runbooks. Never store secrets, internal host credentials, or customer data in Git.