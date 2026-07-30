---
title: "Linux Basics"
weight: 4
---

## Why Linux?

Linux runs ~96% of cloud servers, all Android phones, most IoT devices, and the world's top supercomputers. For software engineers, Linux fluency is non-negotiable.

---

## Essential Commands

### Navigation and Files

```bash
pwd                      # print working directory
ls -la                   # list all files with details
cd /path/to/dir          # change directory
cd ~                     # go to home directory
cd -                     # go to previous directory

mkdir -p path/to/dir     # create directories (with parents)
touch file.txt           # create empty file
cp -r source/ dest/      # copy (recursive)
mv old new               # move/rename
rm -r directory/         # remove (recursive)
```

### Viewing Files

```bash
cat file.txt             # print entire file
less file.txt            # paginated viewer (q to quit)
head -20 file.txt        # first 20 lines
tail -20 file.txt        # last 20 lines
tail -f /var/log/app.log # follow (live updates)
wc -l file.txt           # count lines
```

### Searching

```bash
find . -name "*.py"      # find files by name
grep -r "pattern" dir/   # search file contents
which command            # find command location
locate filename          # fast search (uses index)
```

---

## Users and Permissions

### User Management

```bash
whoami                   # current user
id                       # user ID, groups
sudo command             # run as root
su - username            # switch user
```

### File Permissions

```text
-rwxr-xr-- 1 alice devs 4096 Jan 1 12:00 script.sh
 │││ │││ │││
 │││ │││ └── Others: read only
 │││ └──── Group: read + execute
 └────── Owner: read + write + execute
```

```bash
chmod 755 script.sh      # rwxr-xr-x
chmod 644 file.txt       # rw-r--r--
chown alice:devs file    # change owner:group
```

---

## Package Management

| Distro Family | Tool | Commands |
|--------------|------|----------|
| Debian/Ubuntu | apt | `apt update`, `apt install pkg`, `apt remove pkg` |
| RHEL/CentOS/Amazon | yum/dnf | `yum install pkg`, `dnf update` |
| Alpine | apk | `apk add pkg`, `apk del pkg` |

```bash
# Debian/Ubuntu
sudo apt update                  # refresh package index
sudo apt install nginx           # install
sudo apt remove nginx            # remove
sudo apt upgrade                 # upgrade all packages

# RHEL/Amazon Linux
sudo yum install nginx
sudo yum update
```

---

## Service Management (systemd)

```bash
# Service control
systemctl start nginx            # start service
systemctl stop nginx             # stop service
systemctl restart nginx          # restart
systemctl status nginx           # check status
systemctl enable nginx           # start on boot
systemctl disable nginx          # don't start on boot

# View logs
journalctl -u nginx              # logs for a service
journalctl -u nginx --since today
journalctl -f                    # follow all logs
```

---

## Process Management

```bash
ps aux                           # all processes
ps aux | grep nginx              # find specific process
top                              # real-time process monitor
htop                             # better process monitor

kill PID                         # graceful stop (SIGTERM)
kill -9 PID                      # force kill (SIGKILL)
pkill -f "pattern"              # kill by name pattern
```

---

## Networking Commands

```bash
ip addr show                     # show IP addresses
ip route show                    # show routing table
ss -tlnp                         # listening TCP ports
curl -v http://localhost         # HTTP request
ping host                        # test connectivity
dig domain.com                   # DNS lookup
```

---

## Environment Variables

```bash
echo $PATH                       # view variable
export MY_VAR="value"            # set for current session
env                              # list all env vars

# Persistent (add to ~/.bashrc or ~/.profile)
echo 'export MY_VAR="value"' >> ~/.bashrc
source ~/.bashrc                 # reload
```

---

## SSH (Secure Shell)

```bash
# Connect to remote server
ssh user@hostname
ssh -i ~/.ssh/key.pem user@host  # with private key

# Copy files
scp file.txt user@host:/path/    # local → remote
scp user@host:/path/file.txt .   # remote → local

# SSH key generation
ssh-keygen -t ed25519 -C "email@example.com"
# Public key → ~/.ssh/id_ed25519.pub (share this)
# Private key → ~/.ssh/id_ed25519 (keep secret, chmod 600)
```

---

## Key Takeaways

1. **Learn the core commands** — ls, cd, grep, find, chmod, ps, systemctl
2. **Everything is a file** in Linux — config, devices, processes
3. **systemd** manages services — `systemctl` and `journalctl` are essential
4. **SSH** is how you access remote servers — use key-based auth, never passwords
5. **Package managers** vary by distro — apt (Debian), yum/dnf (RHEL), apk (Alpine)
6. **Environment variables** configure applications — `export` for the session, `.bashrc` for persistence
