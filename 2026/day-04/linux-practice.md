# Day 04 – Linux Practice: Processes and Services

> **Goal:** Hands-on practice with process inspection, service management, and log analysis.

**System:** Almalinux 10  6.12.0-124.56.5.el10_1.x86_64 <!-- fill in: uname -r output -->

**Date:** 27th May 2026<!-- fill in -->

**Service inspected:** `nginx` 

---

## 1. Process Checks

### Command 1 — Top CPU-consuming processes

```bash
$ ps aux --sort=-%cpu | head -15
```

**Output:**

<img width="1352" height="356" alt="image" src="https://github.com/user-attachments/assets/b6b4ffe9-ed01-4c32-ad72-8ce0ee568aea" />



**What I observed:**
- PID 1 is always `systemd` — the root of all processes
- Kernel threads appear in `[brackets]` — they live entirely in kernel space
- `STAT` column: `S` = sleeping, `R` = running, `I` = idle kernel thread

---


### Command 2 — Find a specific process by name

```bash
$ pgrep -la nginx
```

**Output:**

<img width="469" height="136" alt="image" src="https://github.com/user-attachments/assets/bd12fa2c-c334-40e2-b313-0635354b18f6" />

```

**Observation :**
- `pgrep -l` prints the PID and process name
- `-a` shows the full command including arguments
- If blank, the service isn't running — check with `systemctl status nginx`
```
---

## 2. Service Checks

### Command 3 — Inspect the SSH service

```bash
$ systemctl status nginx
```

**Output:**

<img width="1046" height="442" alt="image" src="https://github.com/user-attachments/assets/1e2240c1-359a-4d1e-8828-d0de2be9b02e" />

**What I observed:**
- `Active: active (running)` — service is healthy
- `Main PID` matches what `pgrep -la nginx` returned : 1532
- `enabled` means it will auto-start on reboot

**Key status values to know:**
| Status | Meaning |
|---|---|
| `active (running)` | Process is alive and running |
| `active (exited)` | Ran once and exited cleanly (e.g. one-shot services) |
| `inactive (dead)` | Not running |
| `failed` | Crashed or exited with an error |

---

### Command 4 — List all running services

```bash
$ systemctl list-units --type=service --state=running
```

**Output:**

<img width="970" height="504" alt="image" src="https://github.com/user-attachments/assets/f63dfbfa-580f-4723-8bd5-564e8246072e" />



**What I observed:**
- Total running services: 15<!-- fill in count -->
- Notable ones: nginx, sshd, crond <!-- list 2-3 you recognize -->

---

## 3. Log Checks

### Command 5 — Tail nginx service logs

```bash
$ journalctl -u nginx --since "1 hour ago" --no-pager | tail -20
```

**Output:**

<img width="1055" height="111" alt="image" src="https://github.com/user-attachments/assets/efd94d2b-0023-4fb9-bf57-1332331649cf" />

**What I observed:**
- nginx service is starting, the conf /etc/nginx/nginx.conf is checked for syntax and confirmed as OK, finally starting the service.
---

### Command 6 — Check for system-wide errors

```bash
$ journalctl -p err --since "1 hour ago" --no-pager
```
- `-p err` filters by priority: `emerg > alert > crit > err > warning > notice > info > debug`

**Output:**

<img width="1238" height="88" alt="image" src="https://github.com/user-attachments/assets/528f239c-d83d-4ea9-9c05-40dc975d8904" />

**What I observed:**
- `-- Informational messages --` These are mostly informational kernel/SELinux messages, not critical failures.


---

## 4. Mini Troubleshooting Flow

> **Scenario:** Nginx is not responding. Walk through diagnosing it.

```bash
# Step 1 — is the process even running?
$ pgrep -la nginx

# Step 2 — what does systemd think?
$ systemctl status nginx

# Step 3 — did it crash recently?
$ journalctl -u nginx --since "30 min ago" --no-pager

# Step 4 — is it listening on port?
$ ss -tulpn | grep 80 OR ss -tulpn | grep -i nginx

# Step 5 — if it's stopped, restart it
$ sudo systemctl restart nginx

# Step 6 — confirm it came back
$ systemctl status nginx
```

---

## Key Takeaways

| Area | What I learned |
|---|---|
| Processes | `ps aux` shows every process; `STAT=R` means actively using CPU |
| Services | `systemctl status` is the first stop for any service issue |
| Logs | `journalctl -u <service>` scopes logs to one unit — much cleaner than `tail /var/log/syslog` |
| Troubleshooting | The flow: process alive? → service status? → recent logs? → port listening? → restart |

---

## Commands Reference Card

```bash
# Processes
ps aux --sort=-%cpu | head -15     # top CPU users
pgrep -la <name>                   # find process by name with PID

# Services
systemctl status <service>         # inspect a service
systemctl list-units --type=service --state=running  # all running services
sudo systemctl restart <service>   # restart a service
sudo systemctl enable <service>    # enable at boot

# Logs
journalctl -u <service> -f                    # follow live logs
journalctl -u <service> --since "1 hour ago"  # recent logs for a service
journalctl -p err --since "today"             # all errors today
```

---
