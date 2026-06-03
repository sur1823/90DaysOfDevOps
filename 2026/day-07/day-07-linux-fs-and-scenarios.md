# Day 07 – Linux File System Hierarchy & Scenario-Based Practice

> **Target OS:** Red Hat Enterprise Linux (RHEL 9 / 10)  
> **Standard:** Filesystem Hierarchy Standard (FHS 3.0)  
---

## Section 1 – Linux File System Hierarchy

Think of the Linux filesystem as a single upside-down tree. `/` is the root — every file, device, and directory on the system branches from here, regardless of how many physical disks or partitions exist.

---

### `/` — Root Directory

**What it contains:**  
The top-level anchor of the entire Linux filesystem. No data lives directly here — it is the parent of all other directories. Every path on the system begins with `/`.

**`ls -l /` (live output, selected):**
```
lrwxrwxrwx   1 root root    7  bin -> usr/bin
drwxr-xr-x   2 root root 4096  boot
drwxr-xr-x   6 root root 2280  dev
drwxr-xr-x  72 root root 4096  etc
drwxr-xr-x   4 root root 4096  home
lrwxrwxrwx   1 root root    7  lib -> usr/lib
```

**Notable entries:**
- `bin -> usr/bin` — symlink; on RHEL 7+ and modern distros, `/bin` is merged into `/usr/bin` (the "UsrMerge")
- `etc/` — all system-wide configuration files live here

**⚠️ RHEL note:** RHEL 7+ performs the UsrMerge — `/bin`, `/sbin`, `/lib`, `/lib64` are all symlinks into `/usr/`. This makes the system more consistent and easier to snapshot with tools like OSTree (used by RHEL 9 CoreOS).

**I would use this when...** I need to explore the full system layout, verify mount points, or recover a system by inspecting which top-level directories exist or are mounted.

---

### `/bin` → `/usr/bin` — Essential User Binaries

**What it contains:**  
Core command-line programs available to **all users** — `ls`, `cp`, `mv`, `cat`, `bash`, `grep`, `chmod`, `ping`, etc. On RHEL 7+, `/bin` is a symlink to `/usr/bin` (UsrMerge).

**`ls -l /bin` (live output):**
```
lrwxrwxrwx 1 root root 7  /bin -> usr/bin
```

**`ls -l /usr/bin` (selected):**
```
-rwxr-xr-x 1 root root  55744  [
-rwxr-xr-x 1 root root  14720  addpart
-rwxr-xr-x 1 root root  43552  afm2pl
lrwxrwxrwx 1 root root     26  addr2line -> x86_64-linux-gnu-addr2line
```

**Notable entries:**
- `[` — the test command used in shell `if [ condition ]` statements
- `addpart` — utility to inform the kernel about new disk partitions

**⚠️ RHEL note:** On RHEL, you'll commonly find `dnf`, `rpm`, `firewall-cmd`, `systemctl` in `/usr/bin`. Package manager is `dnf` (not `apt`).

**I would use this when...** I want to check if a tool is installed (`which curl`, `ls /usr/bin | grep python`) or verify the exact binary being called in a script.

---

### `/sbin` → `/usr/sbin` — System Administration Binaries

**What it contains:**  
Commands for system administration — `fdisk`, `fsck`, `ip`, `iptables`, `reboot`, `useradd`, `groupadd`. Traditionally root-only. On RHEL 7+, symlinked to `/usr/sbin`.

**`ls -l /sbin` (live output):**
```
lrwxrwxrwx 1 root root 8  /sbin -> usr/sbin
```

**`ls -l /usr/sbin` (selected):**
```
-rwxr-xr-x 1 root root  55191  adduser
-rwxr-xr-x 1 root root  60992  agetty
-rwxr-xr-x 1 root root  35144  badblocks
lrwxrwxrwx 1 root root      7  addgroup -> adduser
```

**Notable entries:**
- `agetty` — manages terminal login prompts (used by systemd for TTYs)
- `badblocks` — scans a disk for bad sectors

**⚠️ RHEL note:** Key RHEL admin tools here include `dracut` (initramfs builder), `grub2-install`, `semanage` (SELinux management), `subscription-manager`.

**I would use this when...** I need to run privileged admin commands like resizing a partition (`fdisk`), managing users (`useradd`), or restarting network services (`ip link`).

---

### `/usr` — Unix System Resources

**What it contains:**  
The largest directory on most Linux systems. Contains the bulk of installed programs, libraries, headers, documentation, and shared data. All user-installed packages land here. Divided into `bin/`, `sbin/`, `lib/`, `local/`, `share/`, `include/`, `libexec/`.

**`ls -l /usr` (live output):**
```
drwxr-xr-x  2 root root 20480  bin
drwxr-xr-x  2 root root  4096  etc
drwxr-xr-x 50 root root  4096  include
drwxr-xr-x 49 root root  4096  lib
drwxr-xr-x  2 root root  4096  lib64
drwxr-xr-x  6 root root  4096  libexec
drwxr-xr-x 10 root root  4096  local
drwxr-xr-x  2 root root  4096  sbin
```

**Notable entries:**
- `include/` — C/C++ header files (`.h`) used when compiling software from source
- `libexec/` — internal helper executables called by other programs, not meant for direct use

**⚠️ RHEL note:** RHEL treats `/usr` as **read-only** in immutable variants (RHEL CoreOS). All package-managed content lives here; never manually drop files into `/usr/bin` on a managed system — use `/usr/local/` instead.

**I would use this when...** I'm tracing where a package installed its files (`rpm -ql <package>`), looking for development headers, or understanding why a binary is being found at a certain path.

---

### `/usr/local` — Locally Installed Software

**What it contains:**  
Software installed **manually** by the system administrator — compiled from source or installed outside the package manager (`dnf`/`rpm`). Mirrors the `/usr` structure with its own `bin/`, `sbin/`, `lib/`, `etc/`, `share/`. Has higher `PATH` priority than `/usr/bin`.

**`ls -l /usr/local` (live output):**
```
drwxr-xr-x 2 root root 4096  bin
drwxr-xr-x 2 root root 4096  etc
drwxr-xr-x 3 root root 4096  lib
lrwxrwxrwx 1 root root    9  man -> share/man
drwxr-xr-x 2 root root 4096  sbin
drwxr-xr-x 8 root root 4096  share
drwxr-xr-x 2 root root 4096  src
```

**Notable entries:**
- `bin/` — custom-compiled executables not managed by `dnf`
- `src/` — source code for locally compiled programs

**⚠️ RHEL note:** On RHEL, always prefer `/usr/local/` for custom installs. This keeps them safe from `dnf` upgrades overwriting your work. Tools like Node.js custom builds, Python virtualenvs, or in-house scripts go here.

**I would use this when...** I compile a newer version of a tool from source (`./configure --prefix=/usr/local && make install`) and want it isolated from the OS-managed binaries.

---

### `/etc` — Configuration Files

**What it contains:**  
**All system-wide configuration files and scripts.** No binaries. Every service, tool, and user setting has a file or directory here — `/etc/passwd`, `/etc/fstab`, `/etc/hostname`, `/etc/ssh/`, `/etc/cron.d/`, `/etc/systemd/`, `/etc/selinux/`.

**`ls -l /etc` (selected, live output):**
```
-rw-r--r-- 1 root root  2319  bash.bashrc
drwxr-xr-x 2 root root  4096  cron.d
drwxr-xr-x 2 root root  4096  cron.daily
-rw-r--r-- 1 root root    37  fstab
-rw-r--r-- 1 root root   597  group
-rw-r----- 1 root shadow 510  gshadow
-rw-r--r-- 1 root root     3  hostname
-rw-r--r-- 1 root root    65  hosts
-rw-r--r-- 1 root root  1111  passwd
drwxr-xr-x 2 root root  4096  selinux
-rw-r----- 1 root shadow 609  shadow
drwxr-xr-x 6 root root  4096  systemd
```

**Notable entries:**
- `passwd` — user account database (usernames, UIDs, home dirs, shells) — **readable by all**
- `shadow` — hashed passwords — **root-only** (`-rw-r-----`)
- `selinux/` — SELinux policy configuration (RHEL-specific, critical for security)
- `fstab` — filesystem mount table; defines what gets mounted at boot

**⚠️ RHEL-specific files to know:**
| File | Purpose |
|------|---------|
| `/etc/redhat-release` | RHEL version string |
| `/etc/dnf/dnf.conf` | DNF package manager config |
| `/etc/selinux/config` | SELinux mode (enforcing/permissive/disabled) |
| `/etc/firewalld/` | Firewall zones and rules |
| `/etc/yum.repos.d/` | Package repository definitions |
| `/etc/systemd/system/` | Custom systemd unit files |
| `/etc/sudoers.d/` | Sudo privilege configs |
| `/etc/ssh/sshd_config` | SSH server configuration |

**I would use this when...** I need to configure a service (edit `/etc/ssh/sshd_config`), add a user (`/etc/passwd`), change SELinux mode (`/etc/selinux/config`), or define a new mount point (`/etc/fstab`).

---

### `/var` — Variable Data

**What it contains:**  
Files that **grow and change** at runtime — logs, mail spools, package caches, databases, print queues, lock files. The "var" stands for *variable*. This directory is often on a separate partition to prevent log bloat from filling the root filesystem.

**`ls -l /var` (live output):**
```
drwxr-xr-x  2 root root  4096  backups
drwxr-xr-x 11 root root  4096  cache
drwxr-xr-x 26 root root  4096  lib
drwxrwsr-x  2 root staff 4096  local
lrwxrwxrwx  1 root root     9  lock -> /run/lock
drwxr-xr-x  5 root root  4096  log
drwxrwsr-x  2 root mail  4096  mail
```

**Notable entries:**
- `log/` — all system and application logs (`messages`, `secure`, `audit/audit.log`)
- `cache/` — package manager cache (RHEL: `dnf` stores downloaded RPMs here)
- `lib/` — persistent app state data (databases, RPM database at `/var/lib/rpm/`)

**⚠️ RHEL-specific paths to know:**
| Path | Purpose |
|------|---------|
| `/var/log/messages` | General system log (RHEL default, unlike Ubuntu's syslog) |
| `/var/log/secure` | Authentication & sudo logs (RHEL) |
| `/var/log/audit/audit.log` | SELinux & auditd security events |
| `/var/lib/rpm/` | RPM package database |
| `/var/cache/dnf/` | Downloaded RPM packages cache |
| `/var/spool/cron/` | Per-user crontab files |

**I would use this when...** Troubleshooting a service failure by reading `/var/log/messages`, freeing disk space by clearing `/var/cache/dnf/`, or checking authentication failures in `/var/log/secure`.

---

### `/var/log` — System & Application Logs

**What it contains:**  
All log files generated by the kernel, services, and applications. On RHEL, these are text files written by `rsyslog` and binary journals written by `systemd-journald`.

**`ls -l /var/log` (live output):**
```
-rw-r--r-- 1 root root   20360  alternatives.log
drwxr-xr-x 2 root root    4096  apt
-rw-r--r-- 1 root root   61229  bootstrap.log
-rw-rw---- 1 root utmp       0  btmp
-rw-r--r-- 1 root root       0  faillog
drwxr-sr-x 2 root systemd-journal  4096  journal
-rw-rw-r-- 1 root utmp       0  lastlog
```

**Notable entries:**
- `journal/` — systemd binary journal (read with `journalctl`)
- `btmp` — failed login attempts (read with `lastb`)
- `lastlog` — last login per user (read with `lastlog`)

**⚠️ RHEL-specific logs:**
| Log File | Contains |
|----------|---------|
| `/var/log/messages` | General system events |
| `/var/log/secure` | SSH logins, sudo use, PAM events |
| `/var/log/audit/audit.log` | SELinux denials, file access audits |
| `/var/log/dnf.log` | Package install/remove history |
| `/var/log/cron` | Cron job execution records |
| `/var/log/boot.log` | System boot messages |

**I would use this when...** Debugging why a service failed at boot (`journalctl -xb`), investigating an unauthorized login (`/var/log/secure`), or tracing an SELinux denial (`ausearch -m avc`).

---

### `/tmp` — Temporary Files

**What it contains:**  
World-writable scratch space for temporary files created by running programs and users. Files here are **cleared on every reboot**. The sticky bit (`t` in permissions) prevents users from deleting each other's files.

**`ls -l /tmp` (live output):**
```
drwxr-xr-x 2 root root 4096  hsperfdata_root
drwxr-xr-x 3 root root 4096  node-compile-cache
drwxrwxrwx 3 root root 4096  phantomjs
-rw------- 1 root root 1612  rclone-mount-config.json
drwxr-xr-x 6 root root 4096  rclone-mounts
-rw-rw-rw- 1 root root    0  uv-ce9cd633bb00c47d.lock
```

**Notable entries:**
- `rclone-mount-config.json` — temp config file written by a running rclone process
- `hsperfdata_root` — JVM performance data directory

**⚠️ RHEL note:** RHEL uses `systemd-tmpfiles` to manage `/tmp`. On RHEL 9, `/tmp` may be a `tmpfs` (RAM-backed), making it faster but also size-limited. Check with `df -h /tmp`. Also note `/var/tmp` persists across reboots — use it for temp files that should survive a restart.

**I would use this when...** I need a quick scratch space during a script (`cp file.tar.gz /tmp && cd /tmp && tar xf file.tar.gz`), or when a build process needs intermediate file storage.

---

### `/home` — User Home Directories

**What it contains:**  
Personal directories for each regular (non-root) user. Each user gets `/home/<username>` as their starting directory. Contains personal files, dotfiles (`.bashrc`, `.bash_profile`, `.ssh/`), and user-specific application configs.

**`ls -l /home` (live output):**
```
drwxr-xr-x 7 root   root   4096  claude
drwxr-x--- 2 ubuntu ubuntu 4096  ubuntu
```

**Notable entries:**
- `claude/` — home directory for the `claude` user (world-readable in this case)
- `ubuntu/` — home for `ubuntu` user (restricted: `drwxr-x---`, owner + group only)

**⚠️ RHEL note:** In enterprise RHEL setups, `/home` is typically on a **separate partition** or **NFS mount** so user data survives OS reinstalls. Users managed by `sssd`/LDAP/Active Directory will also have home dirs created here on first login via `pam_mkhomedir`.

**I would use this when...** I need to edit a user's shell environment (`.bashrc`, `.bash_profile`), manage their SSH keys (`~/.ssh/authorized_keys`), or check disk usage per user (`du -sh /home/*`).

---

### `/root` — Root User's Home Directory

**What it contains:**  
The superuser's (root's) personal home directory. Kept at `/root` instead of `/home/root` so it remains accessible even if the `/home` partition fails to mount — critical for emergency recovery.

**`ls -l /root` (live output):**
```
total 0
(empty in this container environment)
```

**⚠️ RHEL note:** In practice, `/root` typically contains:
- `/root/.bash_history` — command history (forensically important)
- `/root/.ssh/` — SSH keys for root remote access
- `/root/anaconda-ks.cfg` — the kickstart file used during RHEL installation (very useful for reproducing builds)

**I would use this when...** I'm logged in as root and need to check command history (`cat /root/.bash_history`), set up root SSH keys, or review the original installation kickstart config at `/root/anaconda-ks.cfg`.

---

### `/lib` → `/usr/lib` — Shared Libraries

**What it contains:**  
Essential shared library files (`.so` files — Linux equivalent of Windows `.dll`) required by programs in `/bin` and `/sbin`. Also contains kernel modules (`/lib/modules/<kernel-version>/`). On RHEL 7+, symlinked to `/usr/lib`.

**`ls -l /lib` (live output):**
```
lrwxrwxrwx 1 root root 7  /lib -> usr/lib
```

**`ls -l /usr/lib` (selected):**
```
drwxr-xr-x  5 root root 4096  apt
drwxr-xr-x  2 root root 4096  binfmt.d
lrwxrwxrwx  1 root root   21  cpp -> /etc/alternatives/cpp
drwxr-xr-x  2 root root 4096  bfd-plugins
```

**⚠️ RHEL note:** Key paths:
- `/usr/lib/modules/<kernel>/` — loadable kernel modules (`.ko` files)
- `/usr/lib/systemd/system/` — default systemd unit files shipped by packages
- `/usr/lib64/` — 64-bit libraries (see next entry)

**I would use this when...** A program throws `error while loading shared libraries: libxyz.so.1: cannot open shared object file` — I'd search here and run `ldconfig` to refresh the linker cache.

---

### `/lib64` → `/usr/lib64` — 64-bit Shared Libraries

**What it contains:**  
64-bit shared libraries for the x86_64 architecture. Contains the critical dynamic linker `ld-linux-x86-64.so.2` that bootstraps every dynamically linked program. On RHEL 7+, symlinked to `/usr/lib64`.

**`ls -l /lib64` (live output):**
```
lrwxrwxrwx 1 root root 9  /lib64 -> usr/lib64
```

**⚠️ RHEL note:** RHEL supports multilib (running 32-bit software on 64-bit systems). You'll find both `/usr/lib/` (32-bit) and `/usr/lib64/` (64-bit) on the same system. When installing packages: `dnf install glibc.i686` installs 32-bit libs alongside 64-bit ones.

**I would use this when...** Debugging architecture-specific issues — for example, when a binary compiled for x86_64 can't find its runtime linker, I'd verify `/lib64/ld-linux-x86-64.so.2` exists.

---

### `/opt` — Optional / Third-Party Software

**What it contains:**  
Self-contained application packages installed outside the system package manager. Each application gets its own subdirectory (e.g., `/opt/google/`, `/opt/oracle/`). The application manages all its own files within that subdirectory.

**`ls -l /opt` (live output):**
```
drwxr-xr-x 3 root root 4096  google
drwxr-xr-x 6 root root 4096  pw-browsers
drwx------ 2 root root   39  rclone
```

**Notable entries:**
- `google/` — Google software (e.g., Chrome, Cloud SDK)
- `pw-browsers/` — Playwright browser binaries (self-contained, not in `/usr/bin`)

**⚠️ RHEL note:** Common enterprise software installed under `/opt` on RHEL:
- `/opt/oracle/` — Oracle Database
- `/opt/splunkforwarder/` — Splunk Universal Forwarder
- `/opt/IBM/` — IBM products
- `/opt/rh/` — Red Hat Software Collections (SCL) — older versions of Python, GCC, etc.

**I would use this when...** Installing a vendor-provided application (like Oracle DB or a monitoring agent) that ships as a tarball with its own directory structure and shouldn't mix with OS-managed files.

---

### `/proc` — Process & Kernel Information (Virtual Filesystem)

**What it contains:**  
A **virtual filesystem** — nothing here exists on disk. The kernel generates it entirely in memory to expose live process information and kernel parameters. Every running process has a numbered directory matching its PID.

**`ls -l /proc` (selected, live output):**
```
dr-xr-xr-x  9 root root 0   1          ← PID 1 (systemd/init)
dr-xr-xr-x  9 root root 0   10
dr-xr-xr-x  9 root root 0   115
-r--r--r--  1 root root 0   cmdline    ← kernel boot command line
-r--r--r--  1 root root 0   cpuinfo    ← CPU details
-r--r--r--  1 root root 0   meminfo    ← memory statistics
-r--r--r--  1 root root 0   diskstats  ← disk I/O statistics
```

**Notable entries:**
- `1/` — directory for PID 1 (systemd on RHEL); contains cmdline, maps, status, fd/
- `cpuinfo` — real-time CPU info (`cat /proc/cpuinfo`)
- `meminfo` — real-time memory stats (`cat /proc/meminfo`)

**⚠️ RHEL useful `/proc` paths:**
| Path | Use |
|------|-----|
| `/proc/cpuinfo` | CPU model, cores, flags |
| `/proc/meminfo` | RAM and swap usage |
| `/proc/net/dev` | Network interface stats |
| `/proc/<pid>/cmdline` | Exact command that started a process |
| `/proc/<pid>/fd/` | Open file descriptors of a process |
| `/proc/sys/` | Tunable kernel parameters (via `sysctl`) |

**I would use this when...** I need to inspect a running process (`cat /proc/<pid>/cmdline`), check live memory (`cat /proc/meminfo`), or tune kernel networking parameters (`echo 1 > /proc/sys/net/ipv4/ip_forward`).

---

### `/sys` — System Hardware Interface (Virtual Filesystem)

**What it contains:**  
Another **virtual filesystem** (sysfs), exposing kernel objects, hardware devices, and driver parameters as a structured tree. More organized than `/proc`. Used by `udev` for device management. Many files here are **writable** — writing to them adjusts live kernel/driver behavior.

**`ls -l /sys` (live output):**
```
drwxr-xr-x  2 root root 0  block    ← block devices (disks)
drwxr-xr-x 23 root root 0  bus      ← hardware buses (PCI, USB, SCSI)
drwxr-xr-x 38 root root 0  class    ← device classes (net, input, block)
drwxr-xr-x  4 root root 0  dev
drwxr-xr-x 13 root root 0  devices  ← full hardware device tree
drwxr-xr-x  4 root root 0  firmware ← UEFI/BIOS interface
drwxr-xr-x 13 root root 0  kernel   ← kernel tuning parameters
drwxr-xr-x 86 root root 0  module   ← loaded kernel modules + params
```

**Notable entries:**
- `class/net/` — network interfaces (`/sys/class/net/eth0/`)
- `block/` — block devices and their queues (`/sys/block/sda/queue/`)

**⚠️ RHEL note:** Used heavily by `udev`, `tuned`, and `nmcli`. To change the CPU scaling governor: `echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`. The `tuned` daemon (RHEL-specific) automates many `/sys` tunings for workload profiles.

**I would use this when...** Inspecting hardware details (`cat /sys/class/net/eth0/address` for MAC address), tuning disk I/O schedulers, or debugging why a device isn't detected by the kernel.

---

### `/dev` — Device Files

**What it contains:**  
Special files that represent hardware and virtual devices. Not real files — reading/writing them communicates directly with kernel drivers. Three types: **character devices** (`c`), **block devices** (`b`), and **symlinks**.

**`ls -l /dev` (selected, live output):**
```
crw------- 1 root root   5,   1  console     ← system console
lrwxrwxrwx 1 root root      13  fd -> /proc/self/fd
crw-rw-rw- 1 root root   1,   3  null        ← data sink (discard output)
crw-rw-rw- 1 root root   1,   8  random      ← random number generator
lrwxrwxrwx 1 root root      15  stderr -> /proc/self/fd/2
lrwxrwxrwx 1 root root      15  stdin  -> /proc/self/fd/0
lrwxrwxrwx 1 root root      15  stdout -> /proc/self/fd/1
crw-rw-rw- 1 root root   5,   0  tty         ← current terminal
crw------- 1 root root   4,   0  tty0        ← virtual terminal 0
crw------- 1 root root   1,   1  mem         ← physical memory access
```

**Notable entries:**
- `null` — the "black hole"; anything written here is discarded (`command > /dev/null`)
- `random` / `urandom` — cryptographically secure random data

**⚠️ RHEL common devices:**
| Device | Represents |
|--------|-----------|
| `/dev/sda`, `/dev/sdb` | SCSI/SATA disks |
| `/dev/nvme0n1` | NVMe SSD |
| `/dev/vda` | Virtual disk (KVM/QEMU — common in RHEL VMs) |
| `/dev/sr0` | CD/DVD-ROM |
| `/dev/tty1`–`tty6` | Virtual consoles (Ctrl+Alt+F1–F6) |
| `/dev/pts/0` | Pseudo-terminal (SSH session) |
| `/dev/mapper/` | LVM logical volumes and LUKS encrypted devices |
| `/dev/null` | Discard output |
| `/dev/zero` | Infinite stream of zero bytes |

**I would use this when...** Discarding noisy output (`2>/dev/null`), checking disk names before partitioning (`ls /dev/sd*`), generating test data (`dd if=/dev/urandom of=testfile bs=1M count=10`), or accessing a serial console.

---

### `/boot` — Boot Loader & Kernel Files

**What it contains:**  
Files required to **boot the system** before the root filesystem is fully mounted — the Linux kernel (`vmlinuz`), initial RAM disk (`initramfs`), and GRUB2 bootloader config and modules. **Modifying this incorrectly can make the system unbootable.**

**`ls -l /boot` (live output — empty in this container):**
```
total 0
(container environment — no bootloader needed)
```

**⚠️ RHEL actual `/boot` contents:**
```
-rw-r--r-- 1 root root  ~10MB  vmlinuz-5.14.0-362.el9.x86_64   ← kernel
-rw-r--r-- 1 root root  ~35MB  initramfs-5.14.0-362.el9.x86_64.img  ← initrd
drwx------ 6 root root   4096  efi/                              ← UEFI boot files
drwxr-xr-x 6 root root   4096  grub2/                           ← GRUB2 config
-rw-r--r-- 1 root root         config-5.14.0-362.el9.x86_64     ← kernel build config
-rw-r--r-- 1 root root         System.map-5.14.0-362.el9.x86_64 ← kernel symbol table
```

**⚠️ RHEL note:** On RHEL, `/boot` is typically a **separate 1GB partition** (ext4 or xfs). GRUB2 config is at `/boot/grub2/grub.cfg` (regenerate with `grub2-mkconfig`). After kernel updates via `dnf`, the old kernel is kept — use `grubby` to manage default boot entry.

**I would use this when...** Upgrading the kernel (`dnf update kernel`), recovering from a bad kernel update by booting an older kernel via GRUB menu, or rebuilding the initramfs after adding a driver (`dracut -f`).

---

### `/srv` — Service Data

**What it contains:**  
Data **served by services** running on the system. Web server content (`/srv/www`), FTP data (`/srv/ftp`). Keeps served data separate from configuration (`/etc`) and logs (`/var`). Often empty by default — it's a convention, not required.

**`ls -l /srv` (live output):**
```
total 0
(empty — no services configured in this environment)
```

**⚠️ RHEL note:** Not heavily used by default RHEL services (Apache uses `/var/www/html` instead), but the FHS recommends it. Some organizations use it for NFS exports or FTP roots.

**I would use this when...** Setting up an FTP server and storing the shareable files in `/srv/ftp/`, or organizing a web hosting environment with site data in `/srv/www/<sitename>/`.

---

### `/mnt` — Temporary Manual Mount Points

**What it contains:**  
A conventional directory for **manually and temporarily mounting** filesystems — external drives, NFS shares, ISO images, additional disks. System admins use it for ad-hoc mounts.

**`ls -l /mnt` (live output):**
```
drwxr-xr-x 2 root   root   4096  project
drwxr-xr-x 4 root   root   4096  skills
drwxr-xr-x 1 claude ubuntu    0  transcripts
drwxr-xr-x 5 root   root   4096  user-data
```

**Notable entries:**
- `skills/` — mounted skill/resource directory (this environment uses it as a mount point)
- `user-data/` — user uploads and outputs mounted here

**⚠️ RHEL note:** Common RHEL usage:
```bash
mount /dev/sdb1 /mnt            # Mount a second disk temporarily
mount -t nfs server:/share /mnt # Mount an NFS share
mount -o loop rhel9.iso /mnt    # Mount an ISO image
```
For **permanent** mounts, add the entry to `/etc/fstab` instead.

**I would use this when...** I need to temporarily access a USB drive, mount an NFS export to copy files, or inspect a disk before partitioning it.

---

### `/media` — Auto-Mounted Removable Media

**What it contains:**  
Auto-mount location for removable media — USB sticks, CDs, DVDs. On desktop Linux, `udisks2` automatically mounts removable devices here when plugged in and creates a named subdirectory.

**`ls -l /media` (live output):**
```
total 0
(empty — no removable media attached)
```

**⚠️ RHEL note:** On RHEL **servers**, `/media` is typically always empty — no desktop auto-mount daemon runs. On RHEL **Workstation** or desktops with GNOME, removable drives appear as `/media/<username>/<label>/`. For servers, use `/mnt` with a manual `mount` command.

**I would use this when...** On a RHEL workstation, I plug in a USB drive and it auto-appears at `/media/username/USB_LABEL/` — I'd navigate here to copy files off it.

---

### `/run` — Runtime Data (tmpfs)

**What it contains:**  
Runtime data for the **current boot session** — PID files, Unix sockets, lock files. Stored entirely in **memory (tmpfs)**, so it's fast and wiped clean every reboot. Replaced older `/var/run` and `/var/lock` (now symlinks to `/run`).

**`ls -l /run` (live output):**
```
-rw-r--r-- 1 root root    0  adduser
drwxr-xr-x 3 root root 4096  dbus           ← D-Bus communication socket
drwxrwxrwt 3 root root 4096  lock           ← lock files
drwxr-xr-x 2 root root 4096  log
drwxr-xr-x 2 root root 4096  sendsigs.omit.d
lrwxrwxrwx 1 root root    8  shm -> /dev/shm
drwxr-xr-x 9 root root 4096  systemd        ← systemd runtime state
drwxr-xr-x 2 root root 4096  user
```

**Notable entries:**
- `dbus/` — D-Bus system bus socket (IPC between system services)
- `systemd/` — systemd runtime state, notify sockets, sessions

**⚠️ RHEL note:** Critical runtime files:
| Path | Purpose |
|------|---------|
| `/run/systemd/` | Systemd sessions, service state |
| `/run/sshd.pid` | SSH daemon PID |
| `/run/httpd/httpd.pid` | Apache PID |
| `/run/NetworkManager/` | NetworkManager connection state |
| `/run/lock/` | Advisory lock files |
| `/var/run` | Symlink → `/run` (backwards compat) |

**I would use this when...** Checking if a service is running via its PID file (`cat /run/sshd.pid`), examining active D-Bus sockets, or verifying systemd has a service's socket registered.

---

### `/usr/share` — Architecture-Independent Shared Data

**What it contains:**  
Read-only, architecture-independent data files used by programs — man pages (`/usr/share/man/`), documentation (`/usr/share/doc/`), locale/language files (`/usr/share/locale/`), icons, timezone data (`/usr/share/zoneinfo/`).

**`ls -l /usr/share` (selected, live output):**
```
drwxr-xr-x  3 root root 4096  GConf
drwxr-xr-x  2 root root 4096  aclocal
drwxr-xr-x  5 root root 4096  alsa
drwxr-xr-x  2 root root 4096  applications
drwxr-xr-x  2 root root 4096  ImageMagick-6
```

**Notable entries:**
- `man/` — all manual pages (`man ls` reads from here)
- `zoneinfo/` — timezone database (`/usr/share/zoneinfo/Asia/Kolkata`)
- `doc/` — package documentation and changelogs

**⚠️ RHEL note:** Timezone configuration on RHEL: `timedatectl set-timezone Asia/Kolkata` copies from `/usr/share/zoneinfo/` to `/etc/localtime`. Man pages here are read by `man` command — install `man-pages` package if they're missing on a minimal RHEL install.

**I would use this when...** Reading documentation (`man 5 fstab`), setting the system timezone by locating the right file in `/usr/share/zoneinfo/`, or checking what version of a package is installed via its changelog in `/usr/share/doc/`.

---

## Quick Reference Table

| Directory | Type | Purpose | RHEL Key Files |
|-----------|------|---------|----------------|
| `/` | Real | Filesystem root | `fstab`, `hostname`, top-level tree |
| `/bin` → `/usr/bin` | Symlink | User commands | `bash`, `ls`, `dnf`, `systemctl` |
| `/sbin` → `/usr/sbin` | Symlink | Admin commands | `fdisk`, `useradd`, `dracut`, `semanage` |
| `/usr` | Real | Programs, libs, docs | Bulk of installed software |
| `/usr/local` | Real | Manually installed software | Custom builds, local scripts |
| `/etc` | Real | System configuration | `passwd`, `fstab`, `selinux/config`, `dnf/` |
| `/var` | Real | Variable runtime data | `log/messages`, `lib/rpm/`, `cache/dnf/` |
| `/var/log` | Real | Log files | `messages`, `secure`, `audit/audit.log` |
| `/tmp` | Real/tmpfs | Temporary scratch space | Cleared on reboot |
| `/home` | Real | User home directories | `.bashrc`, `.ssh/authorized_keys` |
| `/root` | Real | Root's home | `anaconda-ks.cfg`, `.bash_history` |
| `/lib` → `/usr/lib` | Symlink | Shared libraries (.so) | `modules/`, `systemd/system/` |
| `/lib64` → `/usr/lib64` | Symlink | 64-bit libraries | `ld-linux-x86-64.so.2` |
| `/opt` | Real | Third-party apps | Oracle, Splunk, Red Hat SCL (`/opt/rh/`) |
| `/proc` | Virtual | Process & kernel info | `cpuinfo`, `meminfo`, `PID dirs`, `sys/` |
| `/sys` | Virtual | Hardware & driver interface | `class/net/`, `block/`, `module/` |
| `/dev` | Virtual | Device files | `sda`, `nvme0n1`, `vda`, `null`, `tty` |
| `/boot` | Real | Kernel & bootloader | `vmlinuz`, `initramfs`, `grub2/grub.cfg` |
| `/srv` | Real | Service-served data | Web/FTP content (often empty on servers) |
| `/mnt` | Real | Manual mount points | Temp disk/NFS mounts |
| `/media` | Real | Auto-mounted removable media | USB drives (desktop only) |
| `/run` | tmpfs | Runtime state (current boot) | PID files, sockets, systemd state |
| `/usr/share` | Real | Arch-independent data | `man/`, `zoneinfo/`, `doc/` |

---

## Key RHEL-Specific Observations

### 1. The UsrMerge
On RHEL 7+, `/bin`, `/sbin`, `/lib`, and `/lib64` are **symlinks** into `/usr/`. This enables:
- Easier atomic system snapshots (OSTree / rpm-ostree on RHEL CoreOS)
- Simpler `PATH` management
- Consistent behavior between early boot and normal operation

### 2. SELinux Integration
RHEL's **SELinux** (Security-Enhanced Linux) assigns a **security context** to every file. You can see these with `ls -lZ`:
```
-rw-r--r--. root root system_u:object_r:etc_t:s0  /etc/hostname
```
The `.` after permissions means SELinux is enforcing on this file. Directories like `/etc/selinux/` and logs in `/var/log/audit/` are critical for SELinux management.

### 3. Package Manager Footprint
Everything installed via `dnf install` lands in:
- Binaries → `/usr/bin/` or `/usr/sbin/`
- Libraries → `/usr/lib64/`
- Config → `/etc/<package>/`
- Data → `/usr/share/<package>/`
- Logs → `/var/log/<package>/`
- State → `/var/lib/<package>/`
- RPM database → `/var/lib/rpm/`

### 4. Virtual Filesystems — Zero Disk Usage
`/proc`, `/sys`, and `/dev` have **0 bytes on disk**. They are kernel interfaces mounted over empty directories:
```
proc      /proc  proc   defaults  0 0
sysfs     /sys   sysfs  defaults  0 0
devtmpfs  /dev   devtmpfs defaults 0 0
```

### 5. Separate Partitions (RHEL Best Practice)
In enterprise RHEL deployments, these are typically separate partitions:

| Partition | Reason |
|-----------|--------|
| `/boot` | Survive root filesystem issues |
| `/home` | Survive OS reinstall; apply separate quotas |
| `/var` | Prevent log bloat from filling `/` |
| `/var/log` | Isolate log growth |
| `/tmp` | Security isolation; `noexec,nosuid` mount options |
| `/opt` | Third-party software isolation |

---
