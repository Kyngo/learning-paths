---
title: "Functions"
weight: 4
---

## Defining Functions

```bash
# Standard syntax
greet() {
    local name="${1:?Error: name required}"
    local greeting="${2:-Hello}"
    echo "${greeting}, ${name}!"
}

# Alternative syntax (less common)
function greet {
    echo "Hello, $1"
}
```

The `function` keyword is optional in bash. Prefer the `name()` syntax — it's POSIX-compatible.

---

## Parameters and Return Values

### Positional Parameters

```bash
create_user() {
    local username="$1"
    local email="$2"
    local role="${3:-viewer}"    # default value
    
    echo "Creating $username ($email) with role: $role"
}

create_user "alice" "alice@example.com" "admin"
create_user "bob" "bob@example.com"          # role defaults to "viewer"
```

### Parameter Validation

```bash
deploy() {
    local env="${1:?Error: environment required (test|pre|prod)}"
    local version="${2:?Error: version required}"
    
    [[ "$env" =~ ^(test|pre|prod)$ ]] || {
        echo "Invalid environment: $env" >&2
        return 1
    }
    
    echo "Deploying v$version to $env"
}
```

### Return Values

Functions return exit codes (0-255), not data. Use `echo` + command substitution for data:

```bash
# Exit codes for success/failure
validate_email() {
    local email="$1"
    [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
}

if validate_email "$user_email"; then
    echo "Valid"
else
    echo "Invalid" >&2
fi

# Echo for returning data
get_timestamp() {
    date +%s
}
ts=$(get_timestamp)

# Multiple values — use a delimiter or global variables
get_dimensions() {
    echo "1920 1080"
}
read -r width height <<< "$(get_dimensions)"
```

---

## Local Variables

Without `local`, variables leak to the calling scope:

```bash
# BAD — leaks to caller
bad_function() {
    x=42    # modifies global scope!
}
x=10
bad_function
echo "$x"   # 42 (unexpected!)

# GOOD — scoped to function
good_function() {
    local x=42
}
x=10
good_function
echo "$x"   # 10 (preserved)
```

**Rule:** Always use `local` for every variable inside a function.

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
cd "$work_dir" || die "Cannot access: $work_dir"
```

### Logging Functions

```bash
readonly RED='\033[0;31m'
readonly YELLOW='\033[0;33m'
readonly GREEN='\033[0;32m'
readonly NC='\033[0m'

log_info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $*" >&2; }
log_error() { echo -e "${RED}[ERROR]${NC} $*" >&2; }

log_info "Starting deployment"
log_warn "Disk space below 20%"
log_error "Connection failed"
```

### Retry Pattern

```bash
retry() {
    local max_attempts="${1:?}"
    local delay="${2:?}"
    shift 2
    
    local attempt=1
    while [[ $attempt -le $max_attempts ]]; do
        if "$@"; then
            return 0
        fi
        log_warn "Attempt $attempt/$max_attempts failed. Retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
    done
    
    log_error "All $max_attempts attempts failed"
    return 1
}

# Usage
retry 3 5 curl -sf "http://api.example.com/health"
```

---

## Function Libraries

Organize reusable functions in separate files:

```bash
# lib/logging.sh
log_info()  { echo "[INFO] $(date '+%H:%M:%S') $*"; }
log_error() { echo "[ERROR] $(date '+%H:%M:%S') $*" >&2; }

# lib/validation.sh
require_command() {
    command -v "$1" &>/dev/null || die "$1 is required but not installed"
}

require_var() {
    [[ -n "${!1:-}" ]] || die "Variable $1 must be set"
}

# main.sh
#!/bin/bash
set -euo pipefail

source "$(dirname "$0")/lib/logging.sh"
source "$(dirname "$0")/lib/validation.sh"

require_command docker
require_var API_KEY

log_info "All checks passed"
```

---

## Advanced Patterns

### Function as Argument (Callbacks)

```bash
apply_to_files() {
    local callback="$1"
    shift
    for file in "$@"; do
        "$callback" "$file"
    done
}

compress() { gzip "$1"; }
apply_to_files compress *.log
```

### Cleanup with Trap

```bash
main() {
    local tmpdir
    tmpdir=$(mktemp -d)
    trap "rm -rf '$tmpdir'" EXIT
    
    # Work in tmpdir safely — cleanup guaranteed
    cp data.txt "$tmpdir/"
    process "$tmpdir/data.txt"
}
main "$@"
```

---

## Key Takeaways

1. **Always use `local`** for function variables
2. **Return exit codes** for success/failure (0 = success)
3. **Use `echo` + command substitution** to return data
4. **Validate parameters** early with `${1:?message}`
5. **Create a `die()` function** in every script
6. **Source library files** to share functions across scripts
7. **Use `"$@"`** to pass all arguments through to another command
