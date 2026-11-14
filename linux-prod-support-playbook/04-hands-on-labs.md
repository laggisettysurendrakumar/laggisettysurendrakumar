# Hands-on Linux Labs

Run these only on a disposable Linux VM or approved non-production environment. Do not use production systems to practise account changes, permission changes, process termination, disk filling, or service restarts.

## Lab 1 — Files and logs

```bash
mkdir -p ~/prod-support-lab/logs ~/prod-support-lab/config
cat > ~/prod-support-lab/logs/app.log <<'LOG'
2026-11-14T10:00:00Z INFO Application started
2026-11-14T10:01:00Z WARN Retry attempt 1
2026-11-14T10:02:00Z ERROR Database timeout requestId=abc-001
2026-11-14T10:03:00Z ERROR Database timeout requestId=abc-002
2026-11-14T10:04:00Z INFO Health check passed
LOG
printf '{"service":"orders","environment":"lab","enabled":true}\n' > ~/prod-support-lab/config/app.json
```

Tasks:

1. Use `file`, `stat`, `less`, and `tail` on the log.
2. Find lines containing `ERROR` and count them.
3. Use `jq` to read the `service` and `enabled` values if `jq` is installed.
4. Archive the lab directory and list the archive contents without extracting.
5. Explain why none of these commands required `sudo`.

## Lab 2 — Permissions and ACLs

```bash
mkdir -p ~/prod-support-lab/shared
printf 'internal lab data\n' > ~/prod-support-lab/shared/sample.txt
chmod 640 ~/prod-support-lab/shared/sample.txt
chmod 750 ~/prod-support-lab/shared
ls -ld ~/prod-support-lab/shared
ls -l ~/prod-support-lab/shared/sample.txt
namei -l ~/prod-support-lab/shared/sample.txt
```

Tasks:

1. Interpret owner, group, and other permissions.
2. Explain whether a user outside the owner/group can list the directory or read the file.
3. If ACL tools are installed, inspect with `getfacl`.
4. Do not change permissions to `777`. Write down the minimum permission a service identity would need for the intended operation.

## Lab 3 — Users and groups (optional; requires a disposable VM and admin rights)

Only perform this lab in a disposable VM where you are authorized to administer local accounts.

```bash
sudo groupadd supportlab
sudo useradd -m -s /bin/bash supportuser
sudo usermod -aG supportlab supportuser
id supportuser
getent group supportlab
```

Tasks:

1. Explain the output of `id`.
2. Explain why `-aG` is used.
3. Log in as the test user (or start a fresh login session) to verify group membership.
4. Clean up only after confirming this is the test VM and the test account owns no needed files. Account deletion is intentionally not shown here; follow your distro's man page and lab procedure.

## Lab 4 — Process and service investigation

Use an existing lab service or start a temporary HTTP server in the lab:

```bash
python3 -m http.server 8088 --bind 127.0.0.1
```

In another terminal:

```bash
pgrep -af 'http.server'
ss -lntup | grep ':8088'
curl -I --max-time 5 http://127.0.0.1:8088/
```

Tasks:

1. Identify the process ID and listening address.
2. Explain why binding to `127.0.0.1` makes it local to the host.
3. Stop the test server with Ctrl+C in the terminal where it is running.

## Lab 5 — Networking layers

Use a public endpoint only if permitted by your network policy, or use an internal lab endpoint.

```bash
getent hosts example.com
nc -vz example.com 443
curl -I --connect-timeout 5 --max-time 10 https://example.com
```

Tasks:

1. Record DNS, TCP, TLS/HTTP observations separately.
2. Repeat with an intentionally invalid hostname and compare the failure.
3. Explain why a successful DNS lookup does not prove the web application is healthy.

## Lab 6 — Disk and inode investigation

Tasks:

1. Inspect `df -hT` and `df -ih` on the lab VM.
2. Compare `du -xhd1 /var` with the filesystem usage shown by `df` (if you have permission).
3. Learn the meaning of a deleted-but-open file using documentation or a controlled lab process.
4. Do not deliberately fill the VM disk unless it is a disposable VM with a dedicated small test volume.

## Lab 7 — Write a mini incident report

Scenario: A web application health check is returning 503, but the host is reachable.

Write a report using the template in `03-production-incident-runbook.md`. Include:

- What you would inspect first
- Commands you would use and why
- Evidence that would distinguish host, process, port, TLS, application, and dependency issues
- Which actions require approval
- How you would verify recovery

## Lab completion checklist

- [ ] I can confirm the host, user, OS, time, and working directory.
- [ ] I can find files and extract evidence from logs without editing them.
- [ ] I can explain owner/group/other permissions and directory traversal.
- [ ] I can inspect users, groups, ACLs, and sudo rights.
- [ ] I can inspect a process, systemd service, listening port, and journal logs.
- [ ] I can isolate DNS, TCP, TLS, and HTTP failures.
- [ ] I can diagnose disk bytes vs inode exhaustion.
- [ ] I can write a clear incident update with facts, hypotheses, actions, and next steps.