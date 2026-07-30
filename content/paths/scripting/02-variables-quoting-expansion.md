---
title: "Variables, Quoting, and Expansion"
weight: 2
---

## Variables

Shell variables are untyped — everything is a string. The shell interprets context to decide if something is a number or text.

### Assignment Rules

```bash
# Correct — NO spaces around =
name="Alice"
count=42
path="/usr/local/bin"

# WRONG — spaces cause errors
name = "Alice"    # Error: "name: command not found"
```

### Variable Usage

```bash
name="world"
echo "Hello, $name"        # Hello, world
echo "Hello, ${name}"      # Same — braces for clarity
echo "${name}_backup"      # world_backup (braces needed here)
echo "$name_backup"        # Empty! (looks for variable name_backup)
```

### Variable Types

```bash
# Regular variable (local to current shell/script)
greeting="Hello"

# Environment variable (inherited by child processes)
export API_KEY="abc123"

# Readonly (constant)
readonly MAX_RETRIES=3
MAX_RETRIES=5              # Error: readonly variable

# Unset
unset greeting             # removes the variable
```

### Special Variables

| Variable | Meaning |
|----------|---------|
| `$0` | Script name |
| `$1`-`$9` | Positional parameters |
| `$#` | Number of arguments |
| `$@` | All arguments (as separate words) |
| `$*` | All arguments (as single string) |
| `$?` | Exit code of last command |
| `$$` | Current process PID |
| `$!` | PID of last background process |

---

## Quoting

Quoting is the #1 source of shell scripting bugs. Understanding it prevents most issues.

### The Three Quoting Modes

| Quoting | Expands Variables | Preserves Spaces | Expands Globs |
|---------|:-:|:-:|:-:|
| `"double"` | ✅ | ✅ | ❌ |
| `'single'` | ❌ | ✅ | ❌ |
| none | ✅ | ❌ | ✅ |

### Why Quoting Matters

```bash
file="my report.txt"

# Without quotes — DISASTER
rm $file              # Runs: rm my report.txt (deletes TWO files!)

# With quotes — correct
rm "$file"            # Runs: rm "my report.txt" (one file)
```

### Word Splitting

Without quotes, the shell splits values on whitespace (IFS):

```bash
files="one.txt two.txt three.txt"

# Unquoted — splits into 3 arguments
for f in $files; do echo "$f"; done
# one.txt, two.txt, three.txt (3 iterations)

# Quoted — single argument
for f in "$files"; do echo "$f"; done
# "one.txt two.txt three.txt" (1 iteration)
```

### Glob Expansion

Without quotes, `*`, `?`, `[...]` expand to matching filenames:

```bash
msg="Files: *.txt"
echo $msg             # Files: report.txt data.txt (expanded!)
echo "$msg"           # Files: *.txt (literal)
```

### The Golden Rule

> **Always double-quote variables** unless you specifically want word splitting or glob expansion.

---

## Parameter Expansion

Bash provides powerful string manipulation without external tools:

### Default Values

```bash
${var:-default}       # use default if var is unset or empty
${var:=default}       # assign default if var is unset or empty
${var:?error msg}     # exit with error if unset/empty
${var:+alternate}     # use alternate if var IS set
```

Practical use:

```bash
config_file="${1:-/etc/myapp/config.yml}"
log_level="${LOG_LEVEL:-info}"
```

### String Operations

```bash
filepath="/home/user/documents/report.final.txt"

# Length
echo "${#filepath}"              # 42

# Prefix removal
echo "${filepath##*/}"           # report.final.txt (basename)

# Suffix removal
echo "${filepath%.*}"            # /home/user/documents/report.final
echo "${filepath%%.*}"           # /home/user/documents/report
echo "${filepath%/*}"            # /home/user/documents (dirname)

# Substitution
echo "${filepath/report/summary}"    # first occurrence
echo "${filepath//o/0}"              # all occurrences

# Case conversion (bash 4+)
name="Hello World"
echo "${name,,}"                 # hello world (lowercase)
echo "${name^^}"                 # HELLO WORLD (uppercase)
```

### Substring

```bash
str="Hello, World!"
echo "${str:7}"                  # World! (from position 7)
echo "${str:7:5}"                # World (5 chars from position 7)
```

---

## Command Substitution

Capture command output into a variable:

```bash
# Modern syntax (preferred — nestable)
today=$(date +%Y-%m-%d)
file_count=$(find . -name "*.py" | wc -l)
git_branch=$(git rev-parse --abbrev-ref HEAD)

# Nested
backup_name="backup_$(hostname)_$(date +%s).tar.gz"

# Legacy syntax (avoid)
today=`date +%Y-%m-%d`
```

---

## Arithmetic Expansion

```bash
# Integer arithmetic with $(( ))
result=$((5 + 3))               # 8
echo $((10 / 3))                # 3 (integer division!)
echo $((10 % 3))                # 1 (modulo)
echo $((2 ** 10))               # 1024 (exponentiation)

# Increment/decrement
count=0
((count++))
((count += 5))

# Floating point — use bc
result=$(echo "scale=2; 10 / 3" | bc)    # 3.33
```

---

## Arrays

```bash
# Indexed arrays
fruits=("apple" "banana" "cherry")
echo "${fruits[0]}"              # apple
echo "${fruits[@]}"              # all elements
echo "${#fruits[@]}"             # 3 (count)
fruits+=("date")                 # append

# Iterate (MUST quote)
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Associative arrays (bash 4+)
declare -A config
config[host]="localhost"
config[port]="5432"
echo "${config[host]}"           # localhost
echo "${!config[@]}"             # keys: host port
```

---

## Key Takeaways

1. **Always double-quote** — `"$var"`, `"${array[@]}"`, `"$(command)"`
2. **No spaces around `=`** in assignments
3. **Use `${var:-default}`** for optional parameters with defaults
4. **Parameter expansion** replaces `basename`, `dirname`, `sed` for simple string ops
5. **`$(...)` over backticks** — nestable, readable
6. **Arrays need `"${arr[@]}"`** — unquoted arrays break on spaces
7. **Bash has no floats** — use `bc` for decimal math
