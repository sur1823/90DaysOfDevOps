# Day 05 – Linux Troubleshooting Runbook
## Target Service: `nginx`
```
> Date:  27th May 2026
> System:  AlmaLinux release 10.1 (Heliotrope Lion) [6.12.0-124.56.5.el10_1.x86_64]
> nginx version:  nginx version: nginx/1.26.3 
> Goal:  Baseline health snapshot → log trace → escalation plan.
```
---

## 1. Environment Baseline

### CMD 1 — Kernel + OS identity

```bash
$ uname -a
$ cat /etc/os-release
```
<img width="717" height="88" alt="image" src="https://github.com/user-attachments/assets/514453e4-d121-4d73-8163-42577037e9f7" />

---

### CMD 2 — Filesystem sanity check

```bash
$ mkdir -p /tmp/runbook-demo
$ cp /etc/hosts /tmp/runbook-demo/hosts-copy && ls -lh /tmp/runbook-demo

```

<img width="1089" height="99" alt="image" src="https://github.com/user-attachments/assets/ee410b0b-da80-4ad0-b912-e71c28cc0e5c" />


**Observed:** `/tmp` is writable, permissions are `644` (owner rw, others r). File copy
succeeded — basic I/O is healthy. The hosts file is 409 bytes, which matches a minimal
VM with just a few entries (localhost, hostname).

---

## 2. Snapshot: CPU & Memory

### CMD 3 — Memory overview

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.9Gi       270Mi       3.7Gi       4.2Mi       163Mi       3.6Gi
Swap:             0B          0B          0B
```

<img width="993" height="97" alt="image" src="https://github.com/user-attachments/assets/22ff3393-1ad7-4040-a8ba-e5dcd2987308" />

**Observed:** Memory is healthy — only 407 MiB `used` out of 3.5 GiB total.
`buff/cache` at 156 MiB is the kernel caching hot files; this is normal and reclaimed
on demand. `Available` memory is pretty good, which shows true available memory and important parameter to consider in case a new application is need to run. 
No `swap` utilization as of now. 

⚠️ **If no swap is configured.** If nginx + app processes exhaust RAM, the OOM killer will force-terminate processes with no safety net.

---

### CMD 4 — CPU and process snapshot

<img width="965" height="146" alt="image" src="https://github.com/user-attachments/assets/023d082d-4792-4303-9207-641107fd7ab4" />

<img width="973" height="168" alt="image" src="https://github.com/user-attachments/assets/9b3c2e06-cc82-4728-a9e3-4616b4f32d4b" />

```bash
$ top -bn1 | head -5
$ ps -o pid,pcpu,pmem,vsz,rss,stat,comm -p $(pgrep -d',' nginx)

VSZ (Virtual Size) : Virtual memory allocated to process.
RSS (Resident Set Size) : Actual RAM currently used. (real memory consumption.)
STAT : S=Sleeping, s=session leader
```
**Observed:** Load average 0.00 — system is idle. `wa = 0%` confirms no disk I/O
bottleneck. nginx master (`Ss`) and 4 workers (`S` = sleeping, waiting for connections)
are all in normal idle state. RSS ~5.08 MiB per worker is expected for a lightly loaded
nginx instance.

---

## 3. Snapshot: Disk & IO

### CMD 5 — Disk space

```bash
$ df -h
```

<img width="1223" height="242" alt="image" src="https://github.com/user-attachments/assets/db5f4d63-3a6c-4559-baba-61d721acff85" />

**Observed:** Root filesystem is at 10% — comfortable. nginx access logs on a busy server can grow
hundreds of MBs per day without logrotate. ⚠️ Alert threshold: 80%. Action threshold: 90%.
Good Practice is to have seperate dedicated filesystem for nginx logs.

---

### CMD 6 — Log directory size + IO stats

```bash
$ du -sh /var/log

```

```bash
$ vmstat 1 3

```

<img width="552" height="142" alt="image" src="https://github.com/user-attachments/assets/36bd32ec-b220-432c-b4b6-dc3942904828" />
<img width="1052" height="145" alt="image" src="https://github.com/user-attachments/assets/dc668c08-9f12-47b6-bb9b-5d929c2191db" />


**Observed:** `/var/log` is only 16M — fresh system with minimal log history.
`vmstat` shows clean IO: `bi` spikes on first sample (initial disk reads), then drops
to 0. `wa = 1 → 1 → 0` confirms disk settled quickly. No swap activity (`si/so = 0`).
`r = 0–1` — run queue is empty, no CPU saturation.

---

## 4. Snapshot: Network

### CMD 7 — Listening ports

<img width="1598" height="114" alt="image" src="https://github.com/user-attachments/assets/a293e528-5f3f-4c62-b621-a0e9131c77d8" />

**Observed:** nginx is bound to port 80 on both IPv4 (`0.0.0.0`) and IPv6 (`[::]`).
PID 1051 matches the master process from `ps`. Backlog queue is 511 — nginx default.
If port 80 is absent here, nginx is not actually serving despite what `systemctl` may say.

---

### CMD 8 — HTTP response check

<img width="521" height="176" alt="image" src="https://github.com/user-attachments/assets/5b91c272-106d-473e-95fb-3785af73ec07" />

```bash
$ curl -I http://localhost

```

**Observed:** `HTTP/1.1 200 OK` — nginx is alive and serving requests. `Content-Length: 5760`
is the welcome page. `Server: nginx/1.26.3` confirms version.
ℹ️ In production, consider hiding the version with `server_tokens off` in `nginx.conf`
to reduce information disclosure.

---

## 5. Logs Reviewed

### CMD 9 — nginx access log (last 50 lines)

```bash
$ tail -n 50 /var/log/nginx/access.log

```

**Observed:** Mostly clean traffic. `200` responses are healthy. ⚠️ One `404` hit 
on `/favicon.ico` from `192.168.0.102` — Browser automatically requested favicon, nginx could not find favicon.ico, so returned 404 Not Found.

---

### CMD 10 — nginx error log

```bash
$ tail -n 50 /var/log/nginx/error.log

```

```bash
$ journalctl -u nginx -n 50 --no-pager

```

**Observed:** No error logs found. journalctl confirms
nginx started cleanly with a valid config. No crash or upstream errors.

---

## 6. Quick Findings Summary

| Area | Status | Observation |
|---|---|---|
| CPU | ✅ Healthy | Load 0.00, `wa = 0%`, workers idle |
| Memory | ⚠️ Watch | Only 407 MiB used — and **no swap** used |
| Disk | ⚠️ Watch | 10% used, ~16 GiB free — monitor log growth |
| nginx process | ✅ Healthy | Master + 4 workers running, normal RSS |
| Port 80 | ✅ Healthy | Listening on IPv4 + IPv6, `curl` returns 200 |
| Access log | ⚠️ Note | nothing critical |
| Error log | ✅ Healthy | no `[error]`, `[crit]` or `[emerg]` |
| Config | ✅ Healthy | `nginx -t` passes — config syntax valid |

---

## 7. If This Worsens — Next Steps

### Step 1 · nginx stops serving (5xx or no response)

```bash
# Is the process still alive?
pgrep -la nginx

# What did systemd last see?
journalctl -u nginx -n 100 --no-pager

# Test config before reloading (never reload with a broken config)
nginx -t

# Graceful reload (zero downtime) — use this first
sudo systemctl reload nginx

# Full restart only if reload fails
sudo systemctl restart nginx

# Watch live after restart
journalctl -u nginx -f
```

### Step 2 · CPU or memory spikes (nginx workers pegged)

```bash
# Identify which worker is the culprit
ps aux --sort=-%cpu | grep nginx

# Watch it live every second
watch -n 1 "ps -o pid,pcpu,pmem,vsz,rss,stat,comm -p \$(pgrep -d',' nginx)"

# Attach strace to see what it's stuck on (safe, read-only)
sudo strace -p <worker_PID> -c -f -e trace=network,file

# Check if upstream (app server / DB) is slow — worker may be waiting
tail -f /var/log/nginx/access.log | grep -v " 200 "

# Increase worker verbosity temporarily
# In /etc/nginx/nginx.conf: error_log /var/log/nginx/error.log debug;
sudo nginx -s reload
```

### Step 3 · Disk fills up (`df -h` shows >85%)

```bash
# Find what's eating space
du -sh /var/log/nginx/*
du -sh /var/log/* | sort -rh | head -10

# Rotate nginx logs immediately (without restart)
sudo mv /var/log/nginx/access.log /var/log/nginx/access.log.bak
sudo kill -USR1 $(cat /var/run/nginx.pid)   # nginx reopens log files on USR1
ls -lh /var/log/nginx/

# Vacuum journal logs
sudo journalctl --vacuum-size=100M

# Verify logrotate is scheduled
cat /etc/logrotate.d/nginx
sudo systemctl status logrotate.timer
```

---

## Commands Quick Reference Card

```bash
# ── nginx health ──────────────────────────────────────────
nginx -v                                    # version
nginx -t                                    # test config syntax
systemctl status nginx                      # service status
pgrep -la nginx                             # PIDs + args

# ── process snapshot ─────────────────────────────────────
ps -o pid,pcpu,pmem,vsz,rss,stat,comm -p $(pgrep -d',' nginx)
top -bn1 | head -10
free -h

# ── disk & io ────────────────────────────────────────────
df -h
du -sh /var/log/nginx/*
vmstat 1 3

# ── network ──────────────────────────────────────────────
ss -tulpn | grep nginx
curl -I http://localhost
curl -I http://localhost/api/health

# ── logs ─────────────────────────────────────────────────
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
journalctl -u nginx -f
journalctl -u nginx --since "1 hour ago" --no-pager

# ── reload / restart ────────────────────────────────────
sudo nginx -t && sudo systemctl reload nginx   # safe reload
sudo systemctl restart nginx                   # hard restart
```

---

