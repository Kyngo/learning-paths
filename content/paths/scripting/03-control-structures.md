---
title: "Control Structures"
weight: 3
---

## Conditionals: if/elif/else

```bash
if [[ "$status" == "active" ]]; then
    echo "Service is running"
elif [[ "$status" == "stopped" ]]; then
    echo "Service is stopped"
else
    echo "Unknown status: $status"
fi
```

The semicolon before `then` replaces a newline — these are equivalent:

```bash
if [[ condition ]]; then command; fi

if [[ condition ]]
then
    command
fi
```

---

## Test Commands: `[[ ]]` vs `[ ]`

### `[[ ]]` — Bash Extended Test (Preferred)

| Category | Operator | Meaning |
|----------|----------|---------|
| String | `==` | Equal |
| String | `!=` | Not equal |
| String | `=~` | Regex match |
| String | `-z "$s"` | String is empty |
| String | `-n "$s"` | String is non-empty |
| Numeric | `-eq`, `-ne` | Equal, not equal |
| Numeric | `-lt`, `-le` | Less than, less or equal |
| Numeric | `-gt`, `-ge` | Greater than, greater or equal |
| File | `-f` | Is regular file |
| File | `-d` | Is directory |
| File | `-e` | Exists (any type) |
| File | `-r`, `-w`, `-x` | Readable, writable, executable |
| File | `-s` | Exists and non-empty |
| Logic | `&&` | AND |
| Logic | `\|\|` | OR |
| Logic | `!` | NOT |

### `[ ]` vs `[[ ]]` Comparison

| Feature | `[ ]` (POSIX) | `[[ ]]` (Bash) |
|---------|:-:|:-:|
| Word splitting on vars | Yes (must quote!) | No |
| Glob expansion | Yes | No |
| Regex (`=~`) | ❌ | ✅ |
| Pattern matching (`==` with `*`) | ❌ | ✅ |
| `&&`, `\|\|` inside | ❌ | ✅ |
| Portability | All shells | Bash/Zsh only |

**Rule:** Use `[[ ]]` in bash scripts. Use `[ ]` only for POSIX portability.

### Practical Examples

```bash
# File checks
[[ -f "$config" ]] || die "Config not found: $config"
[[ -d "$output_dir" ]] || mkdir -p "$output_dir"

# String checks
[[ -z "$API_KEY" ]] && die "API_KEY not set"
[[ "$env" == "prod" ]] && echo "⚠️  Production!"

# Regex
[[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]

# Pattern matching (glob)
[[ "$file" == *.tar.gz ]] && echo "Compressed archive"
[[ "$branch" == feature/* ]] && echo "Feature branch"

# Numeric
[[ $count -gt 0 ]] && echo "Has items"

# Combined
[[ -f "$file" && -r "$file" ]] && echo "Readable file"
```

---

## Case Statement

For multi-branch string matching — cleaner than chained if/elif:

```bash
case "$command" in
    start)
        start_service
        ;;
    stop)
        stop_service
        ;;
    restart|reload)          # multiple patterns
        stop_service
        start_service
        ;;
    *)                       # default (catch-all)
        echo "Usage: $0 {start|stop|restart}" >&2
        exit 1
        ;;
esac
```

### Pattern Matching in Case

```bash
case "$filename" in
    *.tar.gz|*.tgz)  tar xzf "$filename" ;;
    *.zip)           unzip "$filename" ;;
    *.deb)           dpkg -i "$filename" ;;
    *)               echo "Unknown format" >&2 ;;
esac
```

---

## Loops

### For Loop — Iterate Over a List

```bash
# Over literal values
for color in red green blue; do
    echo "$color"
done

# Over glob results
for file in *.log; do
    [[ -f "$file" ]] || continue
    echo "Processing: $file"
done

# Over array elements
servers=("web-01" "web-02" "db-01")
for server in "${servers[@]}"; do
    ping -c 1 "$server" &>/dev/null && echo "$server: UP"
done
```

### C-Style For Loop

```bash
for ((i = 0; i < 10; i++)); do
    echo "Iteration $i"
done
```

### While Loop

```bash
# Counter-based
count=0
while [[ $count -lt 5 ]]; do
    echo "Count: $count"
    ((count++))
done

# Read file line by line (CORRECT way)
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt

# Infinite loop with break
while true; do
    read -rp "Continue? (y/n): " answer
    [[ "$answer" == "n" ]] && break
done
```

### Until Loop

Runs until condition becomes true (opposite of while):

```bash
until ping -c 1 server.example.com &>/dev/null; do
    echo "Waiting for server..."
    sleep 5
done
echo "Server is up!"
```

### Loop Control

```bash
# break — exit loop entirely
for i in {1..100}; do
    [[ $i -gt 10 ]] && break
done

# continue — skip to next iteration
for file in *.log; do
    [[ -s "$file" ]] || continue    # skip empty files
    process "$file"
done
```

---

## Reading Files in Loops

### The Correct Way

```bash
while IFS= read -r line; do
    echo "$line"
done < "$filename"
```

| Component | Purpose |
|-----------|---------|
| `IFS=` | Don't strip leading/trailing whitespace |
| `read -r` | Don't interpret backslashes |
| `< file` | Redirect file to loop's stdin |

### Process CSV

```bash
while IFS=',' read -r name age city; do
    echo "$name is $age, lives in $city"
done < data.csv
```

### Why NOT `for line in $(cat file)`

```bash
# WRONG — breaks on spaces, expands globs
for line in $(cat file.txt); do ...
# "hello world" becomes TWO iterations: "hello" and "world"
```

---

## Short-Circuit Evaluation

```bash
# AND — run second only if first succeeds
[[ -d "$dir" ]] && cd "$dir"

# OR — run second only if first fails
[[ -f "$config" ]] || die "Config missing"
cd "$dir" || exit 1

# Combined: try or die with message
mkdir -p "$output_dir" || { echo "Cannot create dir" >&2; exit 1; }
```

---

## Key Takeaways

1. **Use `[[ ]]`** for all tests in bash — safer, more features
2. **Case statements** are cleaner than long if/elif chains for string matching
3. **`while IFS= read -r`** is the only correct way to read files line by line
4. **Never use `for line in $(cat file)`** — it breaks on spaces and globs
5. **Short-circuit** (`&&`, `||`) for concise one-liners
6. **Always quote variables** in conditions
