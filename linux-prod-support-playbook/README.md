# Linux Production Support Playbook

A practical, Git-friendly reference for an experienced application developer who needs to troubleshoot production Linux servers.

> **Assumption:** This guide is for **Linux command-line administration** (Linux CLI), not a product named **Lynx**. If your team means an internal Lynx application or platform, replace this pack with that platform's own command reference and access procedures.

## How to use this repo

1. Follow [`01-seven-day-learning-plan.md`](01-seven-day-learning-plan.md) for one week.
2. Keep [`02-command-cheatsheet.md`](02-command-cheatsheet.md) open during support shifts.
3. Use [`03-production-incident-runbook.md`](03-production-incident-runbook.md) to investigate incidents consistently.
4. Complete the exercises in [`04-hands-on-labs.md`](04-hands-on-labs.md) on a disposable Linux VM or approved non-production host.

## Learning outcome

By the end of the week, you should be able to safely gather evidence and troubleshoot common production issues involving:

- Files, directories, ownership, permissions, ACLs, disk space, and inodes
- Users, groups, sudo access, SSH basics, and account state
- Processes, ports, services, systemd, and application logs
- DNS, routing, TCP connectivity, HTTP endpoints, and TLS certificates
- CPU, memory, disk I/O, scheduled jobs, and package information
- Docker and Kubernetes basics where those technologies are part of your environment

The goal is not to memorize hundreds of commands. It is to identify the failing layer, collect useful evidence, understand the impact of a command, and make only approved changes.

## Important production safety rules

- Start with read-only inspection. Do not use `sudo` unless it is necessary and authorized.
- Confirm the host, environment, account, and current directory before acting:

  ```bash
  hostname
  whoami
  id
  pwd
  date -Is
  ```

- Never run an unfamiliar command copied from chat, a ticket, or the internet without understanding every argument.
- Do not use `chmod -R 777`, broad `chown -R`, `rm -rf` with wildcards, or recursive commands against `/` or `/var` as a shortcut.
- Do not restart services, kill processes, edit access rules, change firewall settings, install packages, or modify accounts without the required approval/change process.
- Prefer `SIGTERM` and a controlled service restart over `kill -9`; use force only as a last resort under an approved procedure.
- Do not paste passwords, tokens, private keys, connection strings, session cookies, or unredacted customer data into tickets or chat.
- Treat command output as potentially sensitive. Redact hostnames, IPs, usernames, paths, and identifiers where required by policy.
- Commands and options vary by distribution and tooling. Check `cat /etc/os-release`, `command -v <command>`, and the local `man <command>` page.

## A reliable troubleshooting sequence

1. **Scope:** What is failing, for whom, since when, and in which environment/region?
2. **Confirm target:** Verify hostname, account, time, and environment before running commands.
3. **Check impact:** Is this one instance, one node, one region, or all users?
4. **Inspect recent changes:** Deployment, configuration, certificate, secret, permission, network, database, or infrastructure changes.
5. **Trace the request path:** DNS → network/TCP → TLS → load balancer/reverse proxy → application process → dependency/database.
6. **Collect evidence:** Exact timestamps, error messages, service state, relevant log lines, resource metrics, and correlation/request IDs.
7. **Change safely:** Follow the runbook, approvals, rollback plan, and communications process.
8. **Verify and document:** Confirm recovery from the user/request perspective, record what changed, and add prevention actions.

## Recommended local practice environment

Use a disposable Ubuntu or RHEL-compatible VM/container. Create test files and users only in that lab. Do not practise account deletion, permission changes, service restarts, firewall changes, or cleanup commands on production.

## Quick links

- Learning plan: [`01-seven-day-learning-plan.md`](01-seven-day-learning-plan.md)
- Command reference: [`02-command-cheatsheet.md`](02-command-cheatsheet.md)
- Incident runbook: [`03-production-incident-runbook.md`](03-production-incident-runbook.md)
- Hands-on labs: [`04-hands-on-labs.md`](04-hands-on-labs.md)