---
title: "Script Debugging and Error Handling"
weight: 10
---

## The Problem: Silent Failures

Without explicit error handling, shell scripts fail silently and continue executing — often causing more damage than the original error:

```bash
# Dangerous: if cd fails, rm deletes from WRONG directory
cd /data/temp
rm -rf *

# Safe: exit if cd fails
cd /data/temp || exit 1
rm -rf *
```

---

## Strict Mode: `set -euo pipefail`

The single most important line in any bash script:

```bash
#!/bin/bash
set -euo pipefail
```

### What Each Flag Does

| Flag | Without It | With It |
|------|-----------|---------|
| `-e` (errexit) | Script continues after failed commands | Script exits immediately on failure |
| `-u` (nounset) | Unset variables expand to empty string | Unset variables cause an error |
| `-o pipefail` | Pipe exit code = last command's code | Pipe exit code = first failure's code |

### `-e` in Detail

```bash
set -e

cp important.dat /backup/    # if this fails...
rm important.dat             # ...this NEVER runs (script exits)
echo "Done"                  # also never runs
```

Commands that legitimately return non-zero need special handling:

```bash
# Pattern 1: || true (ignore failure)
grep "pattern" file.txt || true

# Pattern 2: disable temporarily
set +e
grep "pattern" file.txt
result=$?
set -e

# Pattern 3: if statement (doesn't trigger -e)
if grep -q "pattern" file.txt; then
    echo "Found"
fi
```

### `-u` in Detail

```bash
set -u

echo "$UNDEFINED_VAR"    # ERROR: unbound variable

# Safe defaults for optional variables
echo "${OPTIONAL_VAR:-default_value}"
echo "${1:-}"            # optional positional parameter
```

### `-o pipefail` in Detail

```bash
# Without pipefail:
false | true
echo $?    # 0 (only true's exit code matters)

# With pipefail:
set -o pipefail
false | true
echo $?    # 1 (false failed, pipe fails)
```

Real-world example:

```bash
set -o pipefail
curl -sf "http://api.example.com/data" | jq '.results'
# If curl fails (network error), script exits
# Without pipefail, jq would get empty input and "succeed"
```

---

## Error Handling Patterns

### Die Function

```bash
die() {
    echo "ERROR: $*" >&2
    exit 1
}

# Usage
[[ -f "$config" ]] || die "Config not found: $config"
command -v docker &>/dev/null || die "docker is required"
```

### Trap ERR

```bash
# Run a function on ANY error
on_error() {
    echo "Error on line $1, exit code $2" >&2
    echo "Command: $BASH_COMMAND" >&2
}
trap 'on_error $LINENO $?' ERR
```

### Cleanup on Exit

```bash
cleanup() {
    rm -rf "$TMPDIR"
    [[ -n "${PID:-}" ]] && kill "$PID" 2>/dev/null
}
trap cleanup EXIT

TMPDIR=$(mktemp -d)
```

### Validate Prerequisites

```bash
check_prerequisites() {
    local missing=()
    
    for cmd in docker kubectl jq; do
        command -v "$cmd" &>/dev/null || missing+=("$cmd")
    done
    
    if [[ ${#missing[@]} -gt 0 ]]; then
        die "Missing required tools: ${missing[*]}"
    fi
    
    [[ -n "${API_KEY:-}" ]] || die "API_KEY environment variable not set"
    [[ -f "$CONFIG_FILE" ]] || die "Config file not found: $CONFIG_FILE"
}

check_prerequisites
```

---

## Debugging Techniques

### Trace Execution (`set -x`)

Prints each command before executing it:

```bash
#!/bin/bash
set -euo pipefail
set -x    # Enable trace

name="world"
echo "Hello, $name"
# Output:
# + name=world
# + echo 'Hello, world'
# Hello, world
```

### Debug a Specific Section

```bash
# Normal execution...
echo "Before debug section"

set -x
# This section is traced
problematic_function
result=$?
set +x

echo "After debug section"
```

### Run Script with Trace

```bash
bash -x script.sh              # trace entire script
bash -xv script.sh             # trace + show source lines
```

### Inspect Variables

```bash
# Print variable type and value
declare -p my_array
# declare -a my_array=([0]="one" [1]="two" [2]="three")

# Print all variables matching a pattern
declare -p | grep "MY_"
```

### Debug Output Function

```bash
DEBUG="${DEBUG:-false}"

debug() {
    [[ "$DEBUG" == "true" ]] && echo "[DEBUG] $*" >&2
}

debug "Processing file: $file"
debug "Count: $count"

# Enable: DEBUG=true ./script.sh
```

---

## Common Pitfalls and Fixes

| Pitfall | Problem | Fix |
|---------|---------|-----|
| Unquoted variables | Word splitting, glob expansion | Always quote: `"$var"` |
| Missing error checks | Silent failures | `set -euo pipefail` |
| `cd` without check | Wrong directory operations | `cd dir \|\| exit 1` |
| Temp files not cleaned | Disk fills up | `trap cleanup EXIT` |
| Race conditions | Lock file not atomic | Use `mkdir` for locking |
| Subshell variable loss | Pipe creates subshell | Use process substitution |

### The Subshell Trap

```bash
# WRONG — count stays 0 (pipe creates subshell)
count=0
cat file.txt | while read -r line; do
    ((count++))
done
echo "$count"    # 0!

# CORRECT — use redirection (no subshell)
count=0
while read -r line; do
    ((count++))
done < file.txt
echo "$count"    # correct value
```

---

## Structured Error Reporting

```bash
#!/bin/bash
set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
log_error() { log "ERROR: $*" >&2; }

on_error() {
    log_error "Failed at line $1 (exit code: $2)"
    log_error "Command: $BASH_COMMAND"
}
trap 'on_error $LINENO $?' ERR

# Script logic
log "Starting $SCRIPT_NAME"
# ...
log "Completed successfully"
```

---

## Key Takeaways

1. **`set -euo pipefail`** in every script — catches 90% of silent failures
2. **`trap EXIT`** guarantees cleanup regardless of how the script ends
3. **`set -x`** for debugging — shows exactly what's executing
4. **Validate prerequisites** at the start — fail fast with clear messages
5. **Pipes create subshells** — variables set inside a pipe don't persist
6. **`die()` function** for consistent error reporting
7. **`$LINENO` and `$BASH_COMMAND`** in ERR traps for precise error location
