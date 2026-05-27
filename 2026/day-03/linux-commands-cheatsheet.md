## Day 03 – Linux Commands Practice

---

## Table of Contents
- [Process Management](#process-management)
- [File System](#file-system)
- [Networking & Troubleshooting](#networking--troubleshooting)
- [Text Processing](#text-processing)
- [System Info & Users](#system-info--users)
- [Git & Productivity](#git--productivity)

---

## Process Management

| Command | Description |
|---|---|
| `ps aux` | List all running processes with user, PID, CPU, mem |
| `top` | Live process monitor (press `q` to quit) |
| `htop` | Interactive process viewer — nicer than top |
| `kill <PID>` | Send SIGTERM to a process (graceful stop) |
| `kill -9 <PID>` | Send SIGKILL — force-terminate immediately |
| `pkill <name>` | Kill all processes matching a name |
| `jobs` | List background jobs in current shell |
| `bg / fg` | Resume a job in background / foreground |
| `nohup cmd &` | Run command that survives logout |
| `nice -n 10 cmd` | Start process with lower priority (0–19) |
| `renice 5 <PID>` | Change priority of a running process |
| `systemctl status svc` | Check status of a systemd service |
| `systemctl start\|stop svc` | Start or stop a service |
| `journalctl -u svc -f` | Follow logs for a systemd service |

> **Note :** Always try `kill` (SIGTERM) before `kill -9` — give the process a chance to clean up first.

---

## File System

| Command | Description |
|---|---|
| `ls -lah` | List files with sizes, permissions, hidden files |
| `cd -` | Go back to previous directory |
| `pwd` | Print current working directory |
| `find / -name '*.log'` | Recursive search by name from root |
| `find . -mtime -1` | Files modified in the last 24 hours |
| `du -sh *` | Disk usage of each item in current directory |
| `df -h` | Filesystem disk space usage, human-readable |
| `cp -r src/ dst/` | Copy directory recursively |
| `mv old new` | Move or rename a file or directory |
| `rm -rf dir/` | Force-delete directory (**no undo — be careful**) |
| `ln -s target link` | Create a symbolic link |
| `chmod 755 file` | Set rwx for owner, rx for group/others |
| `chown user:group file` | Change file ownership |
| `tar -czf out.tar.gz dir` | Create compressed archive |
| `tar -xzf file.tar.gz` | Extract a `.tar.gz` archive |

> **Note :** `find . -mtime -1` is incredibly useful for debugging — "what just changed on this box?"

---

## Networking & Troubleshooting

| Command | Description |
|---|---|
| `ip a` | Show all network interfaces and IP addresses |
| `ip route` | Show routing table |
| `ping -c 4 host` | Test reachability — sends 4 packets |
| `traceroute host` | Trace the route packets take to a host |
| `curl -I https://host` | Fetch HTTP response headers only |
| `wget -qO- url` | Download URL content to stdout |
| `ss -tulpn` | Show listening sockets with PID |
| `netstat -tulpn` | Same as ss — for older systems |
| `nmap -sV host` | Scan open ports and detect service versions |
| `dig domain` | DNS lookup — full response |
| `nslookup domain` | Quick DNS query |
| `tcpdump -i eth0` | Capture live packets on an interface |
| `iptables -L -n` | List firewall rules |
| `ufw status verbose` | Check UFW firewall status |

> **Troubleshoot sequence :** `ping` → `traceroute` → `dig` → `ss` → `tcpdump`. Start simple, go deeper only if needed.

---

## Text Processing

| Command | Description |
|---|---|
| `cat file` | Print entire file to stdout |
| `less file` | Paginate a file (`q` to quit, `/` to search) |
| `head -n 20 file` | Print first 20 lines |
| `tail -f file` | Stream new lines appended to a file (live logs) |
| `grep -rn 'pattern' .` | Recursive search with line numbers |
| `grep -v 'pattern'` | Invert match — lines NOT containing pattern |
| `awk '{print $2}' file` | Print second column of each line |
| `sed 's/old/new/g' file` | Replace all occurrences of old with new |
| `sort \| uniq -c` | Sort then count unique lines |
| `wc -l file` | Count lines in a file |
| `cut -d',' -f1` | Extract first field of a CSV |
| `diff file1 file2` | Show differences between two files |

---

## System Info & Users

| Command | Description |
|---|---|
| `uname -r` | Show kernel version |
| `uptime` | How long the system has been running |
| `free -h` | RAM and swap usage, human-readable |
| `lscpu` | CPU architecture and core info |
| `lsblk` | List block devices (disks, partitions) |
| `dmesg \| tail` | Recent kernel messages — hardware events |
| `who / w` | Who is logged in and what they're doing |
| `id` | Show current user's UID, GID, groups |
| `sudo su -` | Switch to root shell |
| `useradd -m user` | Create new user with home directory |
| `passwd user` | Set or change a user's password |
| `last` | Login history — recent sessions |
| `history \| grep cmd` | Search command history |

---

## Git & Productivity

| Command | Description |
|---|---|
| `git log --oneline` | Compact commit history |
| `git diff --staged` | Show staged changes before committing |
| `git stash` | Temporarily shelve uncommitted changes |
| `git bisect` | Binary search for the commit that broke things |
| `alias ll='ls -lah'` | Create a shorthand command alias |
| `env \| grep VAR` | Check environment variables |
| `crontab -e` | Edit scheduled cron jobs for current user |
| `screen / tmux` | Multiplexer — persist terminal sessions |
| `which cmd` | Show full path to a command binary |
| `type cmd` | Distinguish alias vs binary vs built-in |
| `xargs` | Build and execute commands from stdin |
| `tee file` | Write stdout to both terminal and file |

---
