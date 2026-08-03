---
title: "Linux Administration"
weight: 12
---

## Going Deeper with Linux

The [Linux Basics]({{< relref "04-linux-basics" >}}) section introduced essential commands and concepts. This section dives deeper into the structures and tools that make Linux administration possible: the filesystem hierarchy, permission model, process control, systemd, and power-user commands.

---

## The Filesystem Hierarchy

Linux organises everything into a single tree rooted at `/`. There are no drive letters — devices, processes, and configuration are all mounted into this tree.

### Standard Directories

| Directory | Purpose | Examples |
|-----------|---------|----------|
| `/` | Root of the entire filesystem | Everything branches from here |
| `/etc` | System-wide configuration files | `/etc/nginx/nginx.conf`, `/etc/hosts`, `/etc/passwd` |
| `/var` | Variable data (changes at runtime) | `/var/log/`, `/var/lib/docker/`, `/var/mail/` |
| `/home` | User home directories | `/home/alice/`, `/home/bob/` |
| `/usr` | User system resources (programs, libraries) | `/usr/bin/`, `/usr/lib/`, `/usr/share/` |
| `/tmp` | Temporary files (cleared on reboot) | Scratch space for any process |
| `/bin` | Essential command binaries | `ls`, `cp`, `cat` (often symlinked to `/usr/bin`) |
| `/sbin` | System administration binaries | `fdisk`, `iptables`, `systemctl` |
| `/opt` | Optional/third-party software | `/opt/datadog-agent/`, `/opt/terraform/` |
| `/dev` | Device files | `/dev/sda` (disk), `/dev/null`, `/dev/random` |
| `/proc` | Virtual filesystem — running processes | `/proc/cpuinfo`, `/proc/meminfo`, `/proc/1/status` |
| `/sys` | Virtual filesystem — kernel/hardware info | `/sys/class/net/`, `/sys/block/` |
| `/mnt`, `/media` | Mount points for external filesystems | USB drives, network shares |
| `/root` | Home directory for the root user | Not under `/home` — root is special |

### Key Principles

- **Everything is a file** — devices (`/dev/sda`), processes (`/proc/PID`), and even kernel parameters (`/sys/`) are exposed as files
- **Configuration is text** — almost all config lives in `/etc` as plain-text files you can read, edit, and version-control
- **Logs accumulate in `/var/log`** — `syslog`, `auth.log`, `kern.log`, application logs
- **`/tmp` is ephemeral** — never store anything important there; it's wiped on reboot and often mounted as tmpfs (RAM)

```bash
# See disk usage per top-level directory
du -sh /* 2>/dev/null | sort -rh | head -15

# Find the largest files in /var/log
find /var/log -type f -exec du -h {} + | sort -rh | head -10

# Check what's mounted where
df -hT
```

---

## File Permissions In Depth

### The Permission Model

Every file and directory has three sets of permissions for three entities:

```text
   Owner   Group   Others
    rwx     rwx     rwx
    |||     |||     |||
    |||     |||     ||└─ execute (enter directory / run file)
    |||     |||     |└── write (modify file / create files in dir)
    |||     |||     └─── read (view contents / list directory)
    (same pattern for owner and group)
```

### Numeric (Octal) Representation

Each permission is a bit: read=4, write=2, execute=1. Sum them per entity.

| Octal | Binary | Permission |
|-------|--------|-----------|
| 7 | 111 | rwx |
| 6 | 110 | rw- |
| 5 | 101 | r-x |
| 4 | 100 | r-- |
| 3 | 011 | -wx |
| 2 | 010 | -w- |
| 1 | 001 | --x |
| 0 | 000 | --- |

### Common Permission Patterns

| Octal | Symbolic | Use Case |
|-------|----------|----------|
| `755` | `rwxr-xr-x` | Executables, scripts, public directories |
| `644` | `rw-r--r--` | Regular files (configs, documents) |
| `700` | `rwx------` | Private directories (`.ssh/`) |
| `600` | `rw-------` | Private files (SSH keys, secrets) |
| `775` | `rwxrwxr-x` | Shared group directories |
| `664` | `rw-rw-r--` | Shared group files |

### chmod — Change Permissions

```bash
# Numeric mode
chmod 755 script.sh         # Owner: rwx, Group: r-x, Others: r-x
chmod 600 id_ed25519        # Owner: rw-, others: nothing

# Symbolic mode
chmod u+x script.sh         # Add execute for owner
chmod g-w file.txt          # Remove write for group
chmod o= file.txt           # Remove all permissions for others
chmod a+r file.txt          # Add read for all (a = all)

# Recursive
chmod -R 755 /var/www/html/ # Apply to directory and all contents
```

### chown — Change Ownership

```bash
chown alice file.txt              # Change owner to alice
chown alice:developers file.txt   # Change owner and group
chown -R www-data:www-data /var/www/  # Recursive
chown :developers file.txt        # Change group only
```

### umask — Default Permission Mask

The umask subtracts permissions from newly created files and directories.

| umask | New File Gets | New Dir Gets | Typical Use |
|-------|--------------|-------------|-------------|
| `022` | `644` (rw-r--r--) | `755` (rwxr-xr-x) | Default on most systems |
| `027` | `640` (rw-r-----) | `750` (rwxr-x---) | Server environments |
| `077` | `600` (rw-------) | `700` (rwx------) | Strict privacy |

```bash
# Check current umask
umask

# Set temporarily
umask 027

# Set permanently (add to ~/.bashrc or /etc/profile)
echo "umask 027" >> ~/.bashrc
```

**How it works:** Default permissions are 666 for files, 777 for directories. The umask is *subtracted*: `666 - 022 = 644` for files, `777 - 022 = 755` for directories.

### Special Permissions

| Permission | Octal | Effect on File | Effect on Directory |
|-----------|-------|---------------|-------------------|
| Setuid | 4xxx | Runs as file owner (not caller) | — |
| Setgid | 2xxx | Runs as file group | New files inherit dir's group |
| Sticky bit | 1xxx | — | Only owner can delete their files |

```bash
# Setuid (passwd runs as root regardless of who calls it)
ls -la /usr/bin/passwd
# -rwsr-xr-x 1 root root ...

# Sticky bit on /tmp (prevents users deleting each other's files)
ls -ld /tmp
# drwxrwxrwt 1 root root ...

# Set sticky bit
chmod 1777 /shared/
chmod +t /shared/
```

---

## Users and Groups

### User Management

| File | Purpose |
|------|---------|
| `/etc/passwd` | User accounts (name, UID, home dir, shell) |
| `/etc/shadow` | Password hashes (root-readable only) |
| `/etc/group` | Group definitions and membership |

```bash
# View user info
id alice                        # UID, GID, groups
getent passwd alice             # Full passwd entry
groups alice                    # List groups

# Create and manage users
sudo useradd -m -s /bin/bash alice     # Create with home dir and bash shell
sudo usermod -aG docker alice          # Add to supplementary group
sudo passwd alice                       # Set/change password
sudo userdel -r alice                  # Remove user and home dir

# Create groups
sudo groupadd developers
sudo groupdel developers
```

### Important UIDs

| UID | User | Significance |
|-----|------|-------------|
| 0 | root | Superuser — bypasses all permission checks |
| 1-999 | System accounts | Services (www-data, postgres, nobody) |
| 1000+ | Regular users | Human accounts |

### The `sudo` Mechanism

`sudo` grants temporary root privileges to permitted users. Configuration lives in `/etc/sudoers` (edit with `visudo` only):

```bash
# Run single command as root
sudo apt update

# Open root shell
sudo -i

# Run as another user
sudo -u postgres psql

# Check what you're allowed to do
sudo -l
```

---

## Process Management

### Viewing Processes

```bash
# Snapshot of all processes
ps aux
# Columns: USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND

# Process tree (shows parent-child relationships)
ps auxf
pstree -p

# Real-time monitor
top                             # Basic — press q to quit
htop                            # Better UI, sortable columns
```

### Key `top`/`htop` Fields

| Field | Meaning |
|-------|---------|
| PID | Process ID |
| USER | Owner |
| %CPU | CPU usage percentage |
| %MEM | Memory usage percentage |
| VIRT | Virtual memory allocated |
| RES | Resident (physical) memory used |
| S | State: R=running, S=sleeping, Z=zombie, D=disk wait |
| TIME+ | Total CPU time consumed |
| COMMAND | Command that started the process |

### Signals and Killing Processes

```bash
# Graceful stop (SIGTERM — process can clean up)
kill PID
kill -15 PID                    # Same as above (15 = SIGTERM)

# Force kill (SIGKILL — immediate, no cleanup)
kill -9 PID

# Kill by name
pkill nginx                     # All processes named "nginx"
pkill -f "python app.py"       # Match full command line
killall nginx                   # All processes with exact name

# Common signals
kill -HUP PID                   # Reload configuration (many daemons support this)
kill -STOP PID                  # Pause process
kill -CONT PID                  # Resume paused process
```

| Signal | Number | Default Action | Common Use |
|--------|--------|---------------|-----------|
| SIGHUP | 1 | Terminate | Reload config (daemons) |
| SIGINT | 2 | Terminate | Ctrl+C |
| SIGTERM | 15 | Terminate | Graceful shutdown |
| SIGKILL | 9 | Terminate (cannot be caught) | Force kill |
| SIGSTOP | 19 | Stop | Pause (cannot be caught) |
| SIGCONT | 18 | Continue | Resume paused process |

### Job Control (bg/fg)

```bash
# Run command in background
long_running_task &

# Suspend current foreground process
# Press Ctrl+Z → [1]+  Stopped  long_running_task

# Resume in background
bg %1

# Bring back to foreground
fg %1

# List background jobs
jobs

# Prevent job from dying when terminal closes
nohup long_running_task &
disown %1                       # Detach from current shell
```

---

## systemd Basics

systemd is the init system and service manager on nearly all modern Linux distributions. It starts at PID 1 and manages everything else.

### Unit Types

| Unit Type | Extension | Purpose |
|-----------|-----------|---------|
| Service | `.service` | Daemons and long-running processes |
| Timer | `.timer` | Scheduled tasks (replaces cron) |
| Socket | `.socket` | Network socket activation |
| Mount | `.mount` | Filesystem mount points |
| Target | `.target` | Groups of units (like runlevels) |

### systemctl — Service Control

```bash
# Basic service management
systemctl start nginx            # Start now
systemctl stop nginx             # Stop now
systemctl restart nginx          # Stop + start
systemctl reload nginx           # Reload config without restart (if supported)
systemctl status nginx           # Show state, recent logs, PID

# Boot behaviour
systemctl enable nginx           # Start on boot
systemctl disable nginx          # Don't start on boot
systemctl enable --now nginx     # Enable AND start immediately

# Querying
systemctl is-active nginx        # Returns "active" or "inactive"
systemctl is-enabled nginx       # Returns "enabled" or "disabled"
systemctl list-units --type=service --state=running
systemctl list-timers            # Show scheduled timers
```

### journalctl — Reading Logs

systemd captures all service output in the journal:

```bash
journalctl -u nginx              # All logs for nginx
journalctl -u nginx --since today
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -f           # Follow (live tail)
journalctl -p err                # Only errors and above
journalctl -b                    # Since last boot
journalctl --disk-usage          # Check journal size
```

### Writing a Simple Unit File

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
# After creating/editing a unit file
sudo systemctl daemon-reload     # Re-read unit files
sudo systemctl enable --now myapp
```

---

## Essential Power Commands

### find — Search the Filesystem

```bash
# Find files by name
find /var/log -name "*.log"
find . -iname "readme*"          # Case-insensitive

# Find by type
find / -type d -name "config"    # Directories only
find . -type f -size +100M       # Files over 100MB

# Find by time
find . -mtime -7                 # Modified in last 7 days
find . -mmin -30                 # Modified in last 30 minutes

# Find and act
find /tmp -type f -mtime +30 -delete               # Delete old temp files
find . -name "*.py" -exec grep -l "TODO" {} \;     # Find files containing TODO
```

### grep — Search File Contents

```bash
grep "error" /var/log/syslog     # Basic search
grep -i "error" log.txt          # Case-insensitive
grep -r "TODO" src/              # Recursive through directories
grep -n "pattern" file.txt       # Show line numbers
grep -c "error" log.txt          # Count matches
grep -v "debug" log.txt          # Invert (exclude matches)
grep -E "err|warn|crit" log.txt  # Extended regex (OR)
grep -A3 -B1 "error" log.txt    # 3 lines after, 1 before match
```

### awk — Column Processing

```bash
# Print specific columns
ps aux | awk '{print $1, $2, $11}'           # User, PID, Command
df -h | awk '{print $1, $5}'                 # Device, usage%

# Filter rows
awk -F: '$3 >= 1000 {print $1}' /etc/passwd  # Users with UID >= 1000

# Sum a column
awk '{sum += $5} END {print sum}' data.txt

# Conditional formatting
awk '$3 > 80 {print "HIGH:", $0}' cpu_report.txt
```

### sed — Stream Editor

```bash
# Substitute (first occurrence per line)
sed 's/old/new/' file.txt

# Substitute all occurrences
sed 's/old/new/g' file.txt

# Edit in-place
sed -i 's/old/new/g' file.txt

# Delete lines
sed '/pattern/d' file.txt                    # Delete matching lines
sed '1,5d' file.txt                          # Delete lines 1-5

# Insert/append
sed '3i\New line here' file.txt              # Insert before line 3
sed '3a\New line here' file.txt              # Append after line 3
```

### tar — Archives

```bash
# Create archive
tar -czf archive.tar.gz directory/           # Gzip compressed
tar -cjf archive.tar.bz2 directory/          # Bzip2 compressed

# Extract
tar -xzf archive.tar.gz                      # Extract gzipped
tar -xzf archive.tar.gz -C /target/dir/      # Extract to specific dir

# List contents without extracting
tar -tzf archive.tar.gz

# Common flags: c=create, x=extract, z=gzip, j=bzip2, f=file, v=verbose, t=list
```

### curl and wget — HTTP from the Command Line

```bash
# curl — versatile HTTP client
curl https://api.example.com                  # GET request
curl -o file.zip https://example.com/file.zip # Download to file
curl -I https://example.com                   # Headers only
curl -X POST -d '{"key":"val"}' \
  -H "Content-Type: application/json" \
  https://api.example.com/endpoint

# wget — simple downloader
wget https://example.com/file.tar.gz          # Download file
wget -q -O - https://example.com/script.sh   # Output to stdout
wget -r -l 2 https://example.com/docs/       # Recursive download (2 levels)
```

---

## Environment Variables

Environment variables configure the shell and applications without hardcoding values.

### Scope and Persistence

| Scope | How to Set | Lifetime |
|-------|-----------|----------|
| Current command only | `VAR=value command` | One command |
| Current session | `export VAR=value` | Until terminal closes |
| User-permanent | Add `export` to `~/.bashrc` or `~/.zshrc` | Every login |
| System-wide | Add to `/etc/environment` or `/etc/profile.d/*.sh` | All users |

### Important System Variables

| Variable | Purpose | Example Value |
|----------|---------|---------------|
| `PATH` | Directories searched for commands | `/usr/local/bin:/usr/bin:/bin` |
| `HOME` | Current user's home directory | `/home/alice` |
| `USER` | Current username | `alice` |
| `SHELL` | Default shell | `/bin/bash` |
| `LANG` | System locale | `en_US.UTF-8` |
| `EDITOR` | Default text editor | `vim` or `nano` |
| `LD_LIBRARY_PATH` | Shared library search path | `/usr/local/lib` |

```bash
# View a variable
echo $PATH
printenv PATH

# Modify PATH (prepend)
export PATH="/opt/mytools/bin:$PATH"

# List all variables
env
printenv

# Unset a variable
unset MY_VAR
```

---

## Package Management Overview

Package managers handle installing, updating, and removing software. Each Linux family has its own.

| Family | Distros | Package Format | Manager | Lock File |
|--------|---------|---------------|---------|-----------|
| Debian | Ubuntu, Debian, Mint | `.deb` | `apt` (frontend for dpkg) | `/var/lib/dpkg/lock` |
| Red Hat | RHEL, CentOS, Fedora, Amazon Linux | `.rpm` | `dnf` / `yum` | `/var/run/yum.pid` |
| Alpine | Alpine Linux | `.apk` | `apk` | — |
| Arch | Arch, Manjaro | `.pkg.tar.zst` | `pacman` | `/var/lib/pacman/db.lck` |

### Common Operations Across Managers

| Operation | apt (Debian/Ubuntu) | dnf (RHEL/Fedora) | apk (Alpine) |
|-----------|--------------------|--------------------|--------------|
| Update index | `apt update` | `dnf check-update` | `apk update` |
| Install | `apt install pkg` | `dnf install pkg` | `apk add pkg` |
| Remove | `apt remove pkg` | `dnf remove pkg` | `apk del pkg` |
| Upgrade all | `apt upgrade` | `dnf upgrade` | `apk upgrade` |
| Search | `apt search term` | `dnf search term` | `apk search term` |
| Show info | `apt show pkg` | `dnf info pkg` | `apk info pkg` |
| List installed | `dpkg -l` | `rpm -qa` | `apk list --installed` |

For deeper coverage of package managers (including language-specific ones like pip, npm, and cargo), see the [Package Management]({{< relref "/paths/tools/06-package-management" >}}) section in the Tools path.

---

## Key Takeaways

1. **Understand the filesystem hierarchy** — `/etc` for config, `/var` for runtime data, `/tmp` for ephemeral scratch, `/home` for user files
2. **Master the permission model** — `chmod` sets permissions, `chown` sets ownership, `umask` controls defaults for new files
3. **Use numeric permissions for precision** — 755 for executables, 644 for files, 600 for secrets
4. **Know your signals** — SIGTERM (15) for graceful shutdown, SIGKILL (9) for force kill, SIGHUP (1) for config reload
5. **systemd manages modern Linux** — `systemctl` controls services, `journalctl` reads their logs
6. **Combine find, grep, awk, sed** — These four commands handle 90% of text processing and file discovery tasks
7. **Environment variables are configuration** — Set in `~/.bashrc` for persistence, `export` for the session, inline for one command
8. **Package managers vary by family** — apt (Debian), dnf (Red Hat), apk (Alpine) — same concepts, different syntax
9. **Everything is a file** — Devices, processes, kernel parameters all appear in the filesystem and can be read or written like files
10. **Prefer `sudo` over root login** — Run specific commands with elevated privileges rather than operating as root permanently
