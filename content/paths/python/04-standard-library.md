---
title: "Python: The Standard Library"
weight: 4
---

## File I/O and Pathlib

### pathlib — Object-Oriented Filesystem Paths

```python
from pathlib import Path

# Creating paths (OS-independent)
home = Path.home()                    # /Users/username
project = Path.cwd() / "src" / "main.py"  # Operator / joins paths
config = Path("/etc/nginx/nginx.conf")

# Path inspection
project.name        # "main.py"
project.stem        # "main"
project.suffix      # ".py"
project.parent      # PosixPath('/Users/.../src')
project.parts       # ('/', 'Users', '...', 'src', 'main.py')
project.is_absolute()  # True

# File operations
path = Path("data/output.json")
path.parent.mkdir(parents=True, exist_ok=True)  # Create directories
path.write_text('{"key": "value"}', encoding="utf-8")
content = path.read_text(encoding="utf-8")
path.write_bytes(b'\x00\x01\x02')

# Globbing
for py_file in Path("src").rglob("*.py"):  # Recursive
    print(py_file)

# Temporary files
import tempfile
with tempfile.NamedTemporaryFile(suffix=".json", delete=False) as f:
    f.write(b'{"temp": true}')
    temp_path = Path(f.name)
```

### File I/O Patterns

```python
# Reading large files line by line (memory efficient)
def count_errors(log_path: Path) -> int:
    count = 0
    with open(log_path, "r", encoding="utf-8") as f:
        for line in f:  # Lazy iteration — one line in memory at a time
            if "ERROR" in line:
                count += 1
    return count

# Binary file handling
def copy_file(src: Path, dst: Path, chunk_size: int = 8192) -> None:
    with open(src, "rb") as fsrc, open(dst, "wb") as fdst:
        while chunk := fsrc.read(chunk_size):
            fdst.write(chunk)
```

---

## Collections Module

```python
from collections import (
    defaultdict, Counter, deque, OrderedDict, ChainMap, namedtuple
)

# defaultdict — auto-creates missing keys
word_index = defaultdict(list)
for i, word in enumerate(words):
    word_index[word].append(i)  # No KeyError if word is new

graph = defaultdict(set)
graph["A"].add("B")
graph["A"].add("C")

# Counter — frequency counting
text = "the quick brown fox jumps over the lazy dog"
word_counts = Counter(text.split())
word_counts.most_common(3)  # [('the', 2), ('quick', 1), ('brown', 1)]

# Arithmetic on Counters
inventory = Counter(apples=5, oranges=3)
sold = Counter(apples=2, oranges=1)
remaining = inventory - sold  # Counter({'apples': 3, 'oranges': 2})

# deque — O(1) append/pop from both ends
from collections import deque
buffer = deque(maxlen=100)  # Fixed-size circular buffer
buffer.append("event_1")
buffer.appendleft("priority_event")
buffer.popleft()  # O(1) vs list.pop(0) which is O(n)

# ChainMap — layered dictionaries (first match wins)
defaults = {"color": "blue", "size": "medium", "debug": False}
user_prefs = {"color": "red"}
env_config = {"debug": True}
config = ChainMap(env_config, user_prefs, defaults)
config["color"]  # "red" (from user_prefs)
config["debug"]  # True (from env_config)
config["size"]   # "medium" (from defaults)
```

---

## itertools — Efficient Iteration

```python
import itertools

# Infinite iterators
counter = itertools.count(start=1, step=2)  # 1, 3, 5, 7, ...
cycler = itertools.cycle(["red", "green", "blue"])  # repeats forever
repeater = itertools.repeat("hello", times=3)  # "hello" x3

# Combinatorics
list(itertools.permutations([1, 2, 3], r=2))
# [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]

list(itertools.combinations([1, 2, 3, 4], r=2))
# [(1,2), (1,3), (1,4), (2,3), (2,4), (3,4)]

list(itertools.product("AB", "12"))
# [('A','1'), ('A','2'), ('B','1'), ('B','2')]

# Grouping
data = [
    {"dept": "eng", "name": "Alice"},
    {"dept": "eng", "name": "Bob"},
    {"dept": "sales", "name": "Charlie"},
    {"dept": "sales", "name": "Diana"},
]
# MUST be sorted by key first!
data.sort(key=lambda x: x["dept"])
for dept, members in itertools.groupby(data, key=lambda x: x["dept"]):
    print(f"{dept}: {[m['name'] for m in members]}")

# Chain — flatten iterables
all_items = list(itertools.chain([1, 2], [3, 4], [5, 6]))  # [1,2,3,4,5,6]
all_items = list(itertools.chain.from_iterable(list_of_lists))

# Slicing iterators
first_10 = list(itertools.islice(infinite_gen(), 10))

# Accumulate (running totals)
list(itertools.accumulate([1, 2, 3, 4, 5]))  # [1, 3, 6, 10, 15]
list(itertools.accumulate([3, 1, 4, 1, 5], max))  # [3, 3, 4, 4, 5]
```

---

## functools — Higher-Order Functions

```python
from functools import lru_cache, cache, reduce, wraps, total_ordering

# Caching expensive computations
@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(100)  # Instant — without cache this would take forever
fibonacci.cache_info()  # CacheInfo(hits=98, misses=101, maxsize=128, currsize=101)

# @cache — unlimited cache (Python 3.9+)
@cache
def expensive_lookup(key: str) -> dict:
    # Simulates slow database/API call
    return fetch_from_database(key)

# reduce — fold a sequence into a single value
from functools import reduce
product = reduce(lambda a, b: a * b, [1, 2, 3, 4, 5])  # 120

# total_ordering — generate comparison methods from __eq__ and one other
@total_ordering
class Version:
    def __init__(self, major, minor, patch):
        self.major, self.minor, self.patch = major, minor, patch
    
    def __eq__(self, other):
        return (self.major, self.minor, self.patch) == (other.major, other.minor, other.patch)
    
    def __lt__(self, other):
        return (self.major, self.minor, self.patch) < (other.major, other.minor, other.patch)

# Now Version also has __le__, __gt__, __ge__ automatically
```

---

## datetime and Time

```python
from datetime import datetime, date, timedelta, timezone
from zoneinfo import ZoneInfo  # Python 3.9+

# Current time
now = datetime.now()                          # Naive (no timezone)
now_utc = datetime.now(timezone.utc)          # Aware (UTC)
now_madrid = datetime.now(ZoneInfo("Europe/Madrid"))

# Parsing and formatting
dt = datetime.fromisoformat("2026-05-29T12:00:00+02:00")
dt.strftime("%Y-%m-%d %H:%M:%S %Z")  # "2026-05-29 12:00:00 CEST"

# Arithmetic
tomorrow = now + timedelta(days=1)
next_week = now + timedelta(weeks=1)
duration = datetime(2026, 12, 31) - datetime(2026, 1, 1)
duration.days  # 364

# Timezone conversion
utc_time = datetime(2026, 5, 29, 10, 0, tzinfo=timezone.utc)
tokyo_time = utc_time.astimezone(ZoneInfo("Asia/Tokyo"))
# 2026-05-29 19:00:00+09:00
```

---

## JSON, CSV, and Data Serialization

```python
import json
import csv
from dataclasses import dataclass, asdict

# JSON with custom serialization
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, set):
            return list(obj)
        if hasattr(obj, "__dict__"):
            return obj.__dict__
        return super().default(obj)

data = {"timestamp": datetime.now(), "tags": {"python", "tutorial"}}
json.dumps(data, cls=CustomEncoder, indent=2)

# CSV reading/writing
def read_csv_as_dicts(path: Path) -> list[dict]:
    with open(path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        return list(reader)

def write_csv(path: Path, rows: list[dict]) -> None:
    if not rows:
        return
    with open(path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=rows[0].keys())
        writer.writeheader()
        writer.writerows(rows)
```

---

## subprocess — Running External Commands

```python
import subprocess

# Simple command execution
result = subprocess.run(
    ["git", "status", "--short"],
    capture_output=True,
    text=True,
    check=True,  # Raises CalledProcessError on non-zero exit
    timeout=30,
)
print(result.stdout)

# Piping commands
ps = subprocess.run(["ps", "aux"], capture_output=True, text=True)
grep = subprocess.run(
    ["grep", "python"],
    input=ps.stdout,
    capture_output=True,
    text=True,
)

# Streaming output
process = subprocess.Popen(
    ["tail", "-f", "/var/log/app.log"],
    stdout=subprocess.PIPE,
    text=True,
)
for line in process.stdout:
    if "ERROR" in line:
        alert(line)
```

---

## logging — Structured Logging

```python
import logging
import logging.handlers
import json

# Basic configuration
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

# Module-level logger (best practice)
logger = logging.getLogger(__name__)

# Structured JSON logging
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)
        if hasattr(record, "extra_data"):
            log_data.update(record.extra_data)
        return json.dumps(log_data)

# Rotating file handler
handler = logging.handlers.RotatingFileHandler(
    "app.log", maxBytes=10_000_000, backupCount=5
)
handler.setFormatter(JSONFormatter())
logger.addHandler(handler)

# Usage with context
logger.info("Order processed", extra={"extra_data": {"order_id": "ORD-123", "total": 99.99}})
```

---

## re — Regular Expressions

```python
import re

# Common patterns
email_pattern = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}")
ip_pattern = re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")
url_pattern = re.compile(r"https?://[^\s<>\"]+")

# Named groups
log_pattern = re.compile(
    r"(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})"
    r" \[(?P<level>\w+)\]"
    r" (?P<message>.*)"
)

line = "2026-05-29 12:00:00 [ERROR] Connection timeout"
match = log_pattern.match(line)
if match:
    match.group("level")    # "ERROR"
    match.group("message")  # "Connection timeout"
    match.groupdict()       # {"timestamp": "...", "level": "ERROR", "message": "..."}

# Substitution
cleaned = re.sub(r"\s+", " ", messy_text)  # Collapse whitespace
redacted = re.sub(r"\b\d{4}-\d{4}-\d{4}-\d{4}\b", "XXXX-XXXX-XXXX-XXXX", text)

# findall vs finditer
emails = re.findall(email_pattern, document)  # List of all matches
for match in re.finditer(email_pattern, document):  # Iterator of Match objects
    print(f"Found {match.group()} at position {match.start()}")
```

---

## Hypothetical Use Cases

### Use Case: Log Analyzer

```python
from pathlib import Path
from collections import Counter, defaultdict
from datetime import datetime
import re

def analyze_logs(log_path: Path) -> dict:
    """Parse and analyze application logs."""
    pattern = re.compile(
        r"(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})"
        r" (?P<level>DEBUG|INFO|WARN|ERROR|FATAL)"
        r" \[(?P<service>[\w-]+)\]"
        r" (?P<msg>.*)"
    )
    
    level_counts = Counter()
    errors_by_service = defaultdict(list)
    hourly_distribution = Counter()
    
    with open(log_path, encoding="utf-8") as f:
        for line in f:
            if m := pattern.match(line):
                level = m.group("level")
                service = m.group("service")
                ts = datetime.fromisoformat(m.group("ts"))
                
                level_counts[level] += 1
                hourly_distribution[ts.hour] += 1
                
                if level in ("ERROR", "FATAL"):
                    errors_by_service[service].append({
                        "timestamp": ts,
                        "message": m.group("msg"),
                    })
    
    return {
        "total_lines": sum(level_counts.values()),
        "level_distribution": dict(level_counts),
        "peak_hour": hourly_distribution.most_common(1)[0],
        "error_services": {
            svc: len(errs) for svc, errs in errors_by_service.items()
        },
    }
```

---

## Key Takeaways

1. **pathlib** replaces os.path — use `/` operator for path joining
2. **defaultdict** eliminates key-existence checks; **Counter** handles frequency counting
3. **itertools** provides memory-efficient iteration — never build full lists when you can iterate
4. **lru_cache** gives free memoization for pure functions
5. **Always use `encoding="utf-8"`** explicitly in file operations
6. **logging** over print — configure once at application entry point, use `getLogger(__name__)` everywhere
7. **subprocess.run** with `check=True` and `capture_output=True` is the modern way to run commands
