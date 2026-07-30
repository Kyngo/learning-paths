---
title: "File Operations and Permissions"
weight: 7
---

## File Operations

### Creating Files and Directories

```bash
touch file.txt                   # create empty file (or update timestamp)
mkdir mydir                      # create directory
mkdir -p path/to/nested/dir      # create parent dirs as needed
```

### Copying, Moving, Deleting

```bash
# Copy
cp source.txt dest.txt
cp -r source_dir/ dest_dir/      # recursive (directories)
cp -p file.txt backup/           # preserve permissions and timestamps

# Move / Rename
mv old_name.txt new_name.txt
mv file.txt /other/directory/

# Delete
rm file.txt
rm -r directory/                 # recursive
rm -f file.txt                   # force (no error if missing)
rm -rf directory/                # recursive + force (DANGEROUS)
```

### Safety with `rm`

```bash
# NEVER do this with unquoted or unset variables
rm -rf "$DIR/"                   # if DIR is empty → rm -rf /  !!!

# Safe pattern
[[ -n "$DIR" ]] || die "DIR is empty"
[[ "$DIR" != "/" ]] || die "Refusing to delete /"
rm -rf "${DIR:?ERROR: DIR not set}"
```

---

## find — The Swiss Army Knife

### Basic Searches

```bash
# By name
find . -name "*.py"                     # exact glob match
find . -iname "*.py"                    # case-insensitive
find . -name "*.py" -o -name "*.js"     # OR

# By type
find . -type f                          # regular files only
find . -type d                          # directories only
find . -type l                          # symlinks only

# By size
find . -size +100M                      # larger than 100MB
find . -size -1k                        # smaller than 1KB

# By time
find . -mtime -7                        # modified in last 7 days
find . -mtime +30                       # modified more than 30 days ago
find . -newer reference.txt             # newer than reference file
```

### Actions

```bash
# Execute command on each result
find . -name "*.log" -exec rm {} \;
find . -name "*.py" -exec grep -l "TODO" {} \;

# Batch execution (more efficient)
find . -name "*.log" -exec rm {} +

# Print with custom format
find . -type f -printf "%s %p\n"        # size and path

# Delete directly (faster than -exec rm)
find . -name "*.tmp" -delete
```

### Combining Conditions

```bash
# AND (implicit)
find . -name "*.log" -size +10M

# OR
find . \( -name "*.log" -o -name "*.tmp" \) -mtime +7

# NOT
find . -type f ! -name "*.py"

# Practical: find large old logs
find /var/log -name "*.log" -size +50M -mtime +30 -exec ls -lh {} \;
```

---

## Permissions

### Understanding Permission Notation

```text
-rwxr-xr-- 1 alice developers 4096 Jan 1 12:00 script.sh
│├─┤├─┤├─┤
│ │   │  └── Others: read only (r--)
│ │   └───── Group: read + execute (r-x)
│ └───────── Owner: read + write + execute (rwx)
└─────────── Type: - = file, d = directory, l = symlink
```

### Octal (Numeric) Permissions

| Octal | Binary | Permission |
|-------|--------|-----------|
| 0 | 000 | --- (none) |
| 1 | 001 | --x (execute) |
| 2 | 010 | -w- (write) |
| 3 | 011 | -wx (write + execute) |
| 4 | 100 | r-- (read) |
| 5 | 101 | r-x (read + execute) |
| 6 | 110 | rw- (read + write) |
| 7 | 111 | rwx (all) |

### Common Permission Patterns

| Octal | Meaning | Use Case |
|-------|---------|----------|
| `755` | rwxr-xr-x | Executable scripts, directories |
| `644` | rw-r--r-- | Regular files |
| `600` | rw------- | Private files (keys, secrets) |
| `700` | rwx------ | Private directories |
| `775` | rwxrwxr-x | Shared group directories |
| `664` | rw-rw-r-- | Shared group files |

### chmod — Change Permissions

```bash
# Numeric
chmod 755 script.sh
chmod 644 config.yml
chmod 600 ~/.ssh/id_rsa

# Symbolic
chmod +x script.sh              # add execute for all
chmod u+w file.txt              # add write for owner
chmod go-r file.txt             # remove read for group and others
chmod a+r file.txt              # add read for all
chmod u=rwx,go=rx script.sh     # set explicitly
```

### chown — Change Ownership

```bash
chown alice file.txt             # change owner
chown alice:developers file.txt  # change owner and group
chown -R alice:developers dir/   # recursive
chown :developers file.txt       # change group only
```

---

## Directory Permissions

Directories have different semantics for the same bits:

| Bit | On Files | On Directories |
|-----|----------|---------------|
| `r` | Read content | List contents (`ls`) |
| `w` | Modify content | Create/delete files inside |
| `x` | Execute | Enter directory (`cd`) |

A directory needs `x` to be traversable — without it, you can't `cd` into it or access files inside, even if you know their names.

---

## Special Permissions

| Bit | Octal | On Files | On Directories |
|-----|-------|----------|---------------|
| setuid | 4000 | Run as file owner | — |
| setgid | 2000 | Run as file group | New files inherit group |
| sticky | 1000 | — | Only owner can delete files |

```bash
# setuid (dangerous — use sparingly)
chmod u+s /usr/bin/passwd       # runs as root

# setgid on directory (useful for shared dirs)
chmod g+s /shared/project/      # new files get group ownership

# sticky bit (common on /tmp)
chmod +t /tmp                   # only file owner can delete
```

---

## Practical Patterns

```bash
# Fix permissions recursively
find . -type f -exec chmod 644 {} +    # files: rw-r--r--
find . -type d -exec chmod 755 {} +    # dirs: rwxr-xr-x

# Find world-writable files (security audit)
find / -type f -perm -o+w 2>/dev/null

# Find setuid binaries
find / -type f -perm -4000 2>/dev/null

# Secure a private key
chmod 600 ~/.ssh/id_rsa
chmod 700 ~/.ssh/
```

---

## Key Takeaways

1. **`find` is essential** — learn its flags for name, type, size, time, and actions
2. **Permission octals:** 755 (executables), 644 (files), 600 (secrets)
3. **Always quote paths** in `rm` commands — unset variables can be catastrophic
4. **Directories need `x`** to be traversable
5. **`find ... -exec {} +`** is more efficient than `\;` (batches arguments)
6. **Sticky bit on /tmp** prevents users from deleting each other's files
