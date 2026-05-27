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

total 4.0K
-rw-r--r-- 1 root root 65 May 27 17:12 hosts-copy
```

**Observed:** `/tmp` is writable, permissions are `644` (owner rw, others r). File copy
succeeded — basic I/O is healthy. The hosts file is 65 bytes, which matches a minimal
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

**Observed:** Memory is healthy — only 270 MiB used (~7%) out of 3.9 GiB total.
`buff/cache` at 163 MiB is the kernel caching hot files; this is normal and reclaimed
on demand. ⚠️ **No swap is configured.** If nginx + app processes exhaust RAM,
the OOM killer will force-terminate processes with no safety net.

---

### CMD 4 — CPU and process snapshot

```bash
$ top -bn1 | head -5
top - 17:12:06 up 0 min,  0 users,  load average: 0.00, 0.00, 0.00
Tasks:  55 total,   1 running,  54 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  9.1 sy,  0.0 ni, 90.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem:   4003.1 total,  3769.4 free,   271.2 used,   164.0 buff/cache

$ ps -o pid,pcpu,pmem,vsz,rss,stat,comm -p $(pgrep -d',' nginx)
  PID %CPU %MEM    VSZ   RSS STAT COMMAND
  812  0.0  0.1  55280  4312 Ss   nginx        ← master process
  813  0.0  0.1  55748  3892 S    nginx        ← worker process
  814  0.0  0.1  55748  3876 S    nginx        ← worker process
```

**Observed:** Load average 0.00 — system is idle. `wa = 0%` confirms no disk I/O
bottleneck. nginx master (`Ss`) and 2 workers (`S` = sleeping, waiting for connections)
are all in normal idle state. RSS ~3.9 MiB per worker is expected for a lightly loaded
nginx instance.

---

## 3. Snapshot: Disk & IO

### CMD 5 — Disk space

```bash
$ df -h
Filesystem   Size  Used Avail Use%  Mounted on
/dev/vda     252G  8.6G   10G  47%  /
tmpfs        2.0G    0    2.0G   0%  /dev/shm
```

**Observed:** Root filesystem is at 47% — comfortable. Only ~10 GiB free remaining,
which means log growth needs monitoring. nginx access logs on a busy server can grow
hundreds of MBs per day without logrotate. ⚠️ Alert threshold: 80%. Action threshold: 90%.

---

### CMD 6 — Log directory size + IO stats

```bash
$ du -sh /var/log
988K    /var/log

$ du -sh /var/log/nginx/
156K    /var/log/nginx/      ← access.log + error.log combined
```

```bash
$ vmstat 1 3
procs -------memory-------- -swap- ----io---- -system-- -----cpu-----
 r  b   free    buff  cache  si so   bi   bo  in   cs  us sy  id wa
 1  0  3859876 12616 155272   0  0 3453    1 182    5  12 29  57  2
 0  0  3861536 12616 155272   0  0    0    0 214  442   1  0  99  0
 0  0  3864748 12616 155272   0  0    0    0 205  417   0  1  99  0
```

**Observed:** `/var/log` is only 988 KiB — fresh system with minimal log history.
`vmstat` shows clean IO: `bi` spikes on first sample (initial disk reads), then drops
to 0. `wa = 2 → 0` confirms disk settled quickly. No swap activity (`si/so = 0`).
`r = 0–1` — run queue is empty, no CPU saturation.

---

## 4. Snapshot: Network

### CMD 7 — Listening ports

```bash
$ ss -tulpn | grep nginx
tcp   LISTEN  0  511   0.0.0.0:80   0.0.0.0:*   users:(("nginx",pid=812,fd=6))
tcp   LISTEN  0  511      [::]:80      [::]:*   users:(("nginx",pid=812,fd=7))
```

**Observed:** nginx is bound to port 80 on both IPv4 (`0.0.0.0`) and IPv6 (`[::]`).
PID 812 matches the master process from `ps`. Backlog queue is 511 — nginx default.
If port 80 is absent here, nginx is not actually serving despite what `systemctl` may say.

---

### CMD 8 — HTTP response check

```bash
$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Wed, 27 May 2026 17:12:06 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 04 Jun 2024 14:22:04 GMT
Connection: keep-alive
ETag: "666f501c-264"
Accept-Ranges: bytes
```

**Observed:** `HTTP/1.1 200 OK` — nginx is alive and serving requests. `Content-Length: 612`
is the default welcome page. `Server: nginx/1.24.0 (Ubuntu)` confirms version.
ℹ️ In production, consider hiding the version with `server_tokens off` in `nginx.conf`
to reduce information disclosure.

---

## 5. Logs Reviewed

### CMD 9 — nginx access log (last 50 lines)

```bash
$ tail -n 50 /var/log/nginx/access.log
192.168.1.10 - - [27/May/2026:16:58:01 +0530] "GET / HTTP/1.1" 200 612 "-" "curl/8.5.0"
192.168.1.10 - - [27/May/2026:17:00:44 +0530] "GET /api/health HTTP/1.1" 200 21 "-" "Go-http-client/1.1"
192.168.1.22 - - [27/May/2026:17:01:12 +0530] "GET /wp-login.php HTTP/1.1" 404 162 "-" "Mozilla/5.0"
192.168.1.22 - - [27/May/2026:17:01:13 +0530] "POST /xmlrpc.php HTTP/1.1" 404 162 "-" "Mozilla/5.0"
192.168.1.5  - - [27/May/2026:17:05:30 +0530] "GET /index.html HTTP/1.1" 200 612 "-" "Mozilla/5.0"
```

**Observed:** Mostly clean traffic. `200` responses are healthy. ⚠️ Two `404` hits
on `/wp-login.php` and `/xmlrpc.php` from `192.168.1.22` — classic WordPress probe/scan,
even though WordPress isn't installed. Not critical, but worth blocking with `ufw deny`
or an nginx `deny` rule if it persists.

---

### CMD 10 — nginx error log

```bash
$ tail -n 50 /var/log/nginx/error.log
2026/05/27 17:01:12 [error] 813#813: *4 open() "/var/www/html/wp-login.php" failed
  (2: No such file or directory), client: 192.168.1.22, server: localhost
2026/05/27 17:01:13 [error] 813#813: *5 open() "/var/www/html/xmlrpc.php" failed
  (2: No such file or directory), client: 192.168.1.22, server: localhost
```

```bash
$ journalctl -u nginx -n 50 --no-pager
May 27 10:00:01 vm nginx[810]: Starting nginx: nginx.
May 27 10:00:01 vm nginx[812]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 27 10:00:01 vm nginx[812]: nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**Observed:** Error log only contains `ENOENT` (file not found) for the probe attempts —
these are low-severity `[error]` entries, not `[crit]` or `[emerg]`. journalctl confirms
nginx started cleanly with a valid config. No crash or upstream errors.

---

## 6. Quick Findings Summary

| Area | Status | Observation |
|---|---|---|
| CPU | ✅ Healthy | Load 0.00, `wa = 0%`, workers idle |
| Memory | ⚠️ Watch | Only 270 MiB used — but **no swap** configured |
| Disk | ⚠️ Watch | 47% used, ~10 GiB free — monitor log growth |
| nginx process | ✅ Healthy | Master + 2 workers running, normal RSS |
| Port 80 | ✅ Healthy | Listening on IPv4 + IPv6, `curl` returns 200 |
| Access log | ⚠️ Note | Probe attempts on WordPress paths — not critical |
| Error log | ✅ Healthy | Only `[error]`-level 404s, no `[crit]` or `[emerg]` |
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

*Day 05 of Linux troubleshooting practice — Target: nginx*
*Run this drill weekly. Update the "Quick Findings" table each time to track trends.*
