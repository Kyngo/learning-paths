---
title: "Scripting"
weight: 40
bookFlatSection: false
bookCollapseSection: true
---

Shell scripting is the art of automating tasks through command-line programs. It's the glue that connects tools, automates workflows, and manages systems. Every software engineer needs scripting fluency — it's how you interact with servers, CI/CD pipelines, and development environments.

## Prerequisites

- Basic command-line usage (navigating directories, running commands)
- IT Fundamentals (file systems, processes)

---

## 1. Shell Basics

### What is a Shell?

A shell is a command interpreter — it reads commands, executes them, and displays results. It's both an interactive environment and a scripting language.

| Shell | Description |
|-------|-------------|
| `sh` | POSIX shell — minimal, portable |
| `bash` | Bourne Again Shell — most common on Linux |
| `zsh` | Z Shell — default on macOS, extended features |
| `dash` | Debian Almquist Shell — fast, POSIX-compliant |
| `fish` | Friendly Interactive Shell — user-friendly, not POSIX |

### Script Structure

```bash
#!/bin/bash
# Shebang (^) tells the OS which interpreter to use

# Script description
# Author: name
# Date: 2024-01-01

set -euo pipefail  # Strict mode (explained in section 10)

# Your code here
echo "Hello, World!"
```

### Running Scripts

```bash
# Make executable and run
chmod +x script.sh
./script.sh

# Or run with interpreter explicitly
bash script.sh

# Source (runs in current shell — variables persist)
source script.sh
. script.sh  # equivalent
```

### Key Takeaway

Use `bash` for scripts unless you need POSIX portability (`sh`). Always include a shebang. Use `set -euo pipefail` for safety.

---

## 2. Variables, Quoting, and Expansion

### Variables

```bash
# Assignment (NO spaces around =)
name="Alice"
count=42
readonly PI=3.14159  # immutable

# Usage
echo "Hello, $name"
echo "Count is ${count}"  # braces for clarity/concatenation
echo "${name}_backup"     # without braces: $name_backup (wrong variable)
```

### Quoting Rules

| Quoting | Behavior | Example |
|---------|----------|---------|
| Double `"..."` | Expands variables, preserves spaces | `"Hello $name"` → `Hello Alice` |
| Single `'...'` | Literal — no expansion | `'Hello $name'` → `Hello $name` |
| None | Word splitting + glob expansion | Dangerous for filenames with spaces |
| `$'...'` | ANSI-C quoting (escape sequences) | `$'\t'` → tab character |

**Rule:** Always double-quote variables unless you specifically want word splitting.

```bash
# BAD — breaks on filenames with spaces
for file in $(ls *.txt); do ...

# GOOD
for file in *.txt; do ...

# BAD — word splitting
if [ $name = "Alice" ]; then ...

# GOOD — quoted
if [ "$name" = "Alice" ]; then ...
```

### Parameter Expansion

```bash
# Default values
${var:-default}    # use default if var is unset or empty
${var:=default}    # assign default if var is unset or empty
${var:+alternate}  # use alternate if var IS set

# String manipulation
${var#pattern}     # remove shortest prefix match
${var##pattern}    # remove longest prefix match
${var%pattern}     # remove shortest suffix match
${var%%pattern}    # remove longest suffix match
${var/old/new}     # replace first occurrence
${var//old/new}    # replace all occurrences
${#var}            # string length

# Examples
filepath="/home/user/docs/report.txt"
echo "${filepath##*/}"   # report.txt (basename)
echo "${filepath%/*}"    # /home/user/docs (dirname)
echo "${filepath%.txt}"  # /home/user/docs/report (remove extension)
```

### Command Substitution

```bash
# Modern syntax (preferred)
today=$(date +%Y-%m-%d)
file_count=$(find . -name "*.py" | wc -l)

# Legacy syntax (avoid — harder to nest)
today=`date +%Y-%m-%d`
```

### Arithmetic

```bash
# $(( )) for integer arithmetic
result=$((5 + 3))
((count++))
((total = price * quantity))

# For floating point, use bc or awk
result=$(echo "scale=2; 10 / 3" | bc)  # 3.33
```

### Key Takeaway

Quoting is the #1 source of shell scripting bugs. When in doubt, double-quote. Use parameter expansion for string manipulation — it's faster than calling external tools.

---

## 3. Control Structures

### Conditionals

```bash
# if/elif/else
if [[ "$status" == "active" ]]; then
    echo "Running"
elif [[ "$status" == "stopped" ]]; then
    echo "Stopped"
else
    echo "Unknown: $status"
fi

# Test operators
[[ -f "$file" ]]      # file exists and is regular file
[[ -d "$dir" ]]       # directory exists
[[ -z "$var" ]]       # string is empty
[[ -n "$var" ]]       # string is non-empty
[[ "$a" == "$b" ]]    # string equality
[[ "$a" != "$b" ]]    # string inequality
[[ "$a" =~ ^[0-9]+$ ]] # regex match
[[ $num -eq 5 ]]      # numeric equality
[[ $num -lt 10 ]]     # numeric less than
[[ $num -gt 0 ]]      # numeric greater than
```

### `[[ ]]` vs `[ ]`

| Feature | `[ ]` (test) | `[[ ]]` (bash) |
|---------|-------------|----------------|
| Word splitting | Yes (must quote) | No |
| Glob expansion | Yes | No |
| Regex | No | `=~` |
| Pattern matching | No | `==` with globs |
| Logical operators | `-a`, `-o` | `&&`, `\|\|` |
| Portability | POSIX | Bash/Zsh only |

**Use `[[ ]]`** in bash scripts. Use `[ ]` only for POSIX portability.

### Case Statement

```bash
case "$command" in
    start)
        start_service
        ;;
    stop)
        stop_service
        ;;
    restart)
        stop_service
        start_service
        ;;
    status|info)  # multiple patterns
        show_status
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}" >&2
        exit 1
        ;;
esac
```

### Loops

```bash
# For loop — iterate over list
for fruit in apple banana cherry; do
    echo "$fruit"
done

# C-style for loop
for ((i = 0; i < 10; i++)); do
    echo "$i"
done

# While loop — read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt

# Until loop — run until condition is true
until ping -c 1 server.example.com &>/dev/null; do
    echo "Waiting for server..."
    sleep 5
done
```

### Key Takeaway

Use `[[ ]]` for tests, `case` for multi-branch string matching, and `while read` for processing files line by line. Always quote variables in conditions.

---

## 4. Functions

```bash
# Function definition
greet() {
    local name="${1:?Error: name required}"  # required parameter
    local greeting="${2:-Hello}"             # optional with default
    echo "${greeting}, ${name}!"
}

# Call
greet "Alice"           # Hello, Alice!
greet "Bob" "Hey"       # Hey, Bob!

# Return values
# Functions return exit codes (0-255), not values
# Use echo + command substitution for "return values"
get_timestamp() {
    date +%s
}
ts=$(get_timestamp)

# Or use a global variable (less clean)
calculate() {
    local a=$1 b=$2
    RESULT=$((a + b))
}
calculate 3 4
echo "$RESULT"  # 7
```

### Local Variables

```bash
# Without local — variable leaks to caller
bad_function() {
    x=42  # modifies global scope!
}

# With local — scoped to function
good_function() {
    local x=42  # only visible inside this function
}
```

### Error Handling in Functions

```bash
# Return exit codes
validate_email() {
    local email="$1"
    if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        return 0  # success
    else
        return 1  # failure
    fi
}

if validate_email "$user_email"; then
    echo "Valid"
else
    echo "Invalid email: $user_email" >&2
    exit 1
fi
```

### Key Takeaway

Always use `local` for function variables. Return exit codes for success/failure. Use `echo` + command substitution for returning data. Document required parameters.

---

## 5. Text Processing

### grep — Search for Patterns

```bash
# Basic search
grep "error" logfile.txt

# Common flags
grep -i "error"          # case-insensitive
grep -r "TODO" src/      # recursive
grep -n "pattern" file   # show line numbers
grep -c "error" file     # count matches
grep -l "pattern" *.py   # list files with matches
grep -v "debug" file     # invert (exclude matches)
grep -E "err|warn" file  # extended regex (egrep)
grep -o "[0-9]\+" file   # only matching part

# Regex
grep -E "^[0-9]{4}-[0-9]{2}" access.log  # lines starting with date
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" file  # IP addresses (Perl regex)
```

### sed — Stream Editor

```bash
# Substitute (first occurrence per line)
sed 's/old/new/' file

# Substitute all occurrences
sed 's/old/new/g' file

# In-place editing
sed -i 's/old/new/g' file        # Linux
sed -i '' 's/old/new/g' file     # macOS

# Delete lines
sed '/pattern/d' file            # delete matching lines
sed '1,5d' file                  # delete lines 1-5

# Print specific lines
sed -n '10,20p' file             # print lines 10-20
sed -n '/START/,/END/p' file     # print between patterns

# Multiple operations
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file
```

### awk — Pattern Scanning and Processing

```bash
# Print specific columns
awk '{print $1, $3}' file        # columns 1 and 3
awk -F',' '{print $2}' data.csv  # CSV: second field

# Filter and process
awk '$3 > 100 {print $1, $3}' file  # where column 3 > 100
awk '/error/ {count++} END {print count}' log  # count error lines

# Built-in variables
# NR = line number, NF = number of fields, FS = field separator
awk '{print NR": "$0}' file      # number lines
awk 'NF > 3' file                # lines with more than 3 fields

# Aggregation
awk '{sum += $2} END {print "Total:", sum}' sales.txt
awk '{sum += $2; count++} END {print "Avg:", sum/count}' data.txt
```

### cut, sort, uniq

```bash
# cut — extract columns
cut -d',' -f1,3 data.csv         # fields 1 and 3, comma-delimited
cut -c1-10 file                  # characters 1-10

# sort
sort file                        # alphabetical
sort -n file                     # numeric
sort -k2 -t',' file              # sort by field 2, comma-delimited
sort -r file                     # reverse
sort -u file                     # unique (remove duplicates)

# uniq (requires sorted input)
sort file | uniq                  # remove duplicates
sort file | uniq -c              # count occurrences
sort file | uniq -d              # show only duplicates
```

### Key Takeaway

`grep` finds, `sed` transforms, `awk` processes structured data. These three tools handle 90% of text processing needs. Learn them well — they're available on every Unix system.

---

## 6. Pipes, Redirection, and Process Substitution

### Pipes

Connect stdout of one command to stdin of the next:

```mermaid
flowchart LR
    A["cat access.log"] -->|stdout → stdin| B["grep 'ERROR'"]
    B -->|stdout → stdin| C["awk '{print $4}'"]
    C -->|stdout → stdin| D["sort"]
    D -->|stdout → stdin| E["uniq -c"]
    E -->|stdout → stdin| F["sort -rn"]
    F -->|stdout → stdin| G["head -10"]
```

```bash
# Top 10 error sources
cat access.log | grep "ERROR" | awk '{print $4}' | sort | uniq -c | sort -rn | head -10
```

### File Descriptors and Redirection

```bash
# Standard streams
# 0 = stdin, 1 = stdout, 2 = stderr

# Redirect stdout to file
command > output.txt       # overwrite
command >> output.txt      # append

# Redirect stderr
command 2> errors.txt      # stderr to file
command 2>&1               # stderr to stdout
command &> all.txt         # both stdout and stderr to file

# Redirect stdin
command < input.txt

# Discard output
command > /dev/null 2>&1   # discard everything
command &> /dev/null       # same (bash shorthand)

# Separate stdout and stderr
command > output.txt 2> errors.txt
```

### Here Documents and Here Strings

```bash
# Here document — multi-line input
cat <<EOF
Hello, $name!
Today is $(date).
EOF

# Here string — single-line input
grep "pattern" <<< "$variable"

# No variable expansion (quoted delimiter)
cat <<'EOF'
This $variable is literal
EOF
```

### Process Substitution

Treat command output as a file:

```bash
# Compare two command outputs
diff <(sort file1.txt) <(sort file2.txt)

# Feed multiple inputs
paste <(cut -f1 data.txt) <(cut -f3 data.txt)

# Write to multiple destinations
command | tee >(grep "error" > errors.log) >(grep "warn" > warnings.log) > /dev/null
```

### Key Takeaway

Pipes are the Unix philosophy in action — small tools connected together. Understand file descriptors (0, 1, 2) to control where data flows. Process substitution eliminates temporary files.

---

## 7. File Operations and Permissions

### File Operations

```bash
# Create
touch file.txt
mkdir -p path/to/dir     # create parent dirs

# Copy
cp source dest
cp -r source_dir/ dest_dir/  # recursive

# Move/rename
mv old_name new_name
mv file.txt /other/dir/

# Delete
rm file.txt
rm -rf directory/        # recursive, force (DANGEROUS)

# Find files
find . -name "*.py"                    # by name
find . -type f -mtime -7              # modified in last 7 days
find . -size +100M                    # larger than 100MB
find . -name "*.log" -exec rm {} \;   # find and delete
find . -name "*.py" -exec grep -l "TODO" {} \;  # find files containing pattern
```

### Permissions

```text
-rwxr-xr-- 1 user group 4096 Jan 1 12:00 script.sh
│├─┤├─┤├─┤
│ │   │  └── Others: read only
│ │   └───── Group: read + execute
│ └───────── Owner: read + write + execute
└─────────── File type (- = file, d = directory, l = symlink)
```

```bash
# Numeric permissions
chmod 755 script.sh   # rwxr-xr-x
chmod 644 file.txt    # rw-r--r--
chmod 600 secret.key  # rw-------

# Symbolic
chmod +x script.sh    # add execute for all
chmod u+w file.txt    # add write for owner
chmod go-r file.txt   # remove read for group and others

# Ownership
chown user:group file.txt
chown -R user:group directory/
```

### Key Takeaway

Use `find` for complex file searches. Understand permission octals (755, 644, 600). Never use `rm -rf` without double-checking the path — especially with variables.

---

## 8. Regular Expressions

### Basic Syntax

| Pattern | Matches |
|---------|---------|
| `.` | Any single character |
| `*` | Zero or more of preceding |
| `+` | One or more of preceding (extended) |
| `?` | Zero or one of preceding (extended) |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Any character in set |
| `[^abc]` | Any character NOT in set |
| `[a-z]` | Range |
| `\d` | Digit (Perl regex) |
| `\w` | Word character (Perl regex) |
| `\s` | Whitespace (Perl regex) |
| `{n}` | Exactly n repetitions |
| `{n,m}` | Between n and m repetitions |
| `(...)` | Capture group |
| `\|` | Alternation (OR) |

### Practical Examples

```bash
# Validate IP address
grep -E '^([0-9]{1,3}\.){3}[0-9]{1,3}$' file

# Extract email addresses
grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file

# Match ISO date
grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}' file

# Replace date format (YYYY-MM-DD → DD/MM/YYYY)
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/g' file

# Extract URLs
grep -oE 'https?://[^ ]+' file
```

### BRE vs ERE

| Feature | BRE (Basic) | ERE (Extended) |
|---------|-------------|----------------|
| Tool default | `grep`, `sed` | `grep -E`, `sed -E`, `awk` |
| `+`, `?`, `{`, `\|`,`(` | Must escape: `\+`, `\?` | Literal: `+`, `?` |
| Backreferences | `\1`, `\2` | `\1`, `\2` |

### Key Takeaway

Regular expressions are a universal skill — they work in grep, sed, awk, Python, JavaScript, and every text editor. Master the basics (anchors, character classes, quantifiers, groups) and you can handle most text matching tasks.

---

## 9. Process Management

### Background Jobs

```bash
# Run in background
long_command &

# List background jobs
jobs

# Bring to foreground
fg %1

# Send to background
# Press Ctrl+Z (suspend), then:
bg %1

# Disown (detach from terminal)
disown %1
```

### Signals

| Signal | Number | Default Action | Use |
|--------|--------|---------------|-----|
| SIGHUP | 1 | Terminate | Terminal closed |
| SIGINT | 2 | Terminate | Ctrl+C |
| SIGQUIT | 3 | Core dump | Ctrl+\\ |
| SIGKILL | 9 | Terminate (uncatchable) | Force kill |
| SIGTERM | 15 | Terminate | Graceful shutdown |
| SIGSTOP | 19 | Stop (uncatchable) | Pause process |
| SIGCONT | 18 | Continue | Resume paused process |

```bash
# Send signals
kill PID          # SIGTERM (default)
kill -9 PID       # SIGKILL (force, last resort)
kill -HUP PID     # SIGHUP (reload config)
killall name      # kill by process name
pkill -f pattern  # kill by command pattern
```

### Traps

Catch signals in scripts for cleanup:

```bash
#!/bin/bash
TMPFILE=$(mktemp)

# Cleanup on exit (any reason)
cleanup() {
    rm -f "$TMPFILE"
    echo "Cleaned up"
}
trap cleanup EXIT

# Handle Ctrl+C gracefully
trap 'echo "Interrupted"; exit 1' INT

# Ignore SIGHUP
trap '' HUP

# Your script logic
echo "Working..." > "$TMPFILE"
sleep 100
```

### Process Information

```bash
# List processes
ps aux                    # all processes
ps -ef                    # full format
pgrep -f "pattern"        # find PID by pattern

# Real-time monitoring
top                       # interactive
htop                      # better interactive (if installed)

# Process tree
pstree -p                 # show PIDs
```

### Key Takeaway

Always use `trap` for cleanup in scripts that create temporary files or hold resources. Send SIGTERM first, SIGKILL only as a last resort. Use `trap EXIT` to guarantee cleanup regardless of how the script ends.

---

## 10. Script Debugging and Error Handling

### Strict Mode

```bash
#!/bin/bash
set -euo pipefail

# set -e: Exit immediately on any command failure
# set -u: Treat unset variables as errors
# set -o pipefail: Pipe fails if ANY command in the pipe fails
```

**Without `pipefail`:**

```bash
# This succeeds even though grep fails (cat succeeds)
cat nonexistent.txt 2>/dev/null | grep "pattern"
echo $?  # 1 (grep's exit code, but only by luck)
```

**With `pipefail`:**

```bash
set -o pipefail
cat nonexistent.txt 2>/dev/null | grep "pattern"
# Script exits because cat failed
```

### Error Handling Patterns

```bash
# Check command success
if ! command -v docker &>/dev/null; then
    echo "Error: docker is not installed" >&2
    exit 1
fi

# OR operator for error handling
cd "$target_dir" || { echo "Cannot cd to $target_dir" >&2; exit 1; }

# Custom error function
die() {
    echo "ERROR: $*" >&2
    exit 1
}

[[ -f "$config_file" ]] || die "Config file not found: $config_file"
```

### Debugging

```bash
# Trace execution (print each command before running)
set -x    # enable
set +x    # disable

# Or run with trace
bash -x script.sh

# Debug specific section
set -x
problematic_code_here
set +x

# Print variable state
declare -p variable_name  # shows type and value
```

### Key Takeaway

`set -euo pipefail` should be in every script. It catches 90% of silent failures. Add explicit error messages for user-facing scripts. Use `set -x` for debugging.

---

## 11. Cron, Systemd Timers, and Automation

### Cron

```bash
# Edit crontab
crontab -e

# Format: minute hour day_of_month month day_of_week command
# ┌───────── minute (0-59)
# │ ┌─────── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌─── month (1-12)
# │ │ │ │ ┌─ day of week (0-7, 0 and 7 = Sunday)
# │ │ │ │ │
# * * * * * command

# Examples
0 2 * * *     /opt/scripts/backup.sh        # Daily at 2:00 AM
*/5 * * * *   /opt/scripts/health-check.sh   # Every 5 minutes
0 9 * * 1-5   /opt/scripts/report.sh         # Weekdays at 9:00 AM
0 0 1 * *     /opt/scripts/monthly-cleanup.sh # First of each month
```

### Cron Best Practices

```bash
#!/bin/bash
# Good cron script pattern

# Use absolute paths (cron has minimal PATH)
PATH=/usr/local/bin:/usr/bin:/bin

# Redirect output to log
exec >> /var/log/my-script.log 2>&1

# Timestamp
echo "=== $(date '+%Y-%m-%d %H:%M:%S') ==="

# Lock file to prevent overlapping runs
LOCKFILE="/tmp/my-script.lock"
if ! mkdir "$LOCKFILE" 2>/dev/null; then
    echo "Already running, exiting"
    exit 0
fi
trap 'rmdir "$LOCKFILE"' EXIT

# Actual work
do_the_thing
```

### Systemd Timers (Modern Alternative)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup service

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

```bash
systemctl enable --now backup.timer
systemctl list-timers
```

### Key Takeaway

Use cron for simple scheduling. Use systemd timers for better logging, dependency management, and missed-run handling (`Persistent=true`). Always use lock files to prevent overlapping executions.

---

## 12. Portable Scripting and POSIX Compliance

### POSIX vs Bash

| Feature | POSIX (`sh`) | Bash |
|---------|-------------|------|
| `[[ ]]` | No | Yes |
| Arrays | No | Yes |
| `$(( ))` | Yes | Yes |
| `local` | Not guaranteed | Yes |
| `{1..10}` | No | Yes |
| `<<<` (here string) | No | Yes |
| Process substitution | No | Yes |
| `=~` (regex) | No | Yes |

### Writing Portable Scripts

```bash
#!/bin/sh
# POSIX-compliant script

# Use [ ] instead of [[ ]]
if [ "$var" = "value" ]; then
    echo "match"
fi

# Use $(command) not `command`
result=$(date +%s)

# No arrays — use positional parameters or files
# No local — use functions carefully
# No [[ ]] — use [ ] with proper quoting
```

### When to Use POSIX

- Docker containers with minimal shells (Alpine uses `ash`)
- Embedded systems
- Scripts that must run on any Unix (macOS, Linux, BSD)
- System init scripts

### When Bash is Fine

- Linux servers where bash is guaranteed
- CI/CD pipelines (specify `#!/bin/bash`)
- Developer tooling
- Any script where you control the environment

### Key Takeaway

Write POSIX when portability matters (Docker, multi-OS). Write bash when you control the environment and need its features (arrays, `[[ ]]`, process substitution). Always specify the interpreter in the shebang.

---

## Summary

| Topic | Core Concept |
|-------|-------------|
| Shell Basics | Shebang, strict mode, execution methods |
| Variables & Quoting | Always double-quote, parameter expansion |
| Control Flow | `[[ ]]` for tests, `case` for multi-branch |
| Functions | `local` variables, exit codes for status |
| Text Processing | grep (find), sed (transform), awk (process) |
| Pipes & Redirection | Connect tools, control data flow |
| Files & Permissions | `find` for search, octal for permissions |
| Regex | Universal pattern matching |
| Processes | Signals, traps, cleanup |
| Error Handling | `set -euo pipefail`, explicit checks |
| Automation | Cron/systemd timers, lock files |
| Portability | POSIX for compatibility, bash for features |

Shell scripting is not about writing elegant code — it's about getting things done reliably. A 10-line script that automates a daily task saves more time than a beautiful program that never gets written.
