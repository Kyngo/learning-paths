---
title: "Python: Control Flow, Functions, and Modules"
weight: 2
---

## Control Flow

### Conditionals — Beyond Basics

```python
# Walrus operator (Python 3.8+) — assign and test in one expression
if (n := len(data)) > 100:
    print(f"Processing {n} items in batches")

# Chained comparisons
if 0 <= score <= 100:
    print("Valid score")

# Match statement (Python 3.10+) — structural pattern matching
def handle_command(command):
    match command.split():
        case ["quit"]:
            return "Goodbye"
        case ["hello", name]:
            return f"Hello, {name}!"
        case ["add", *numbers] if all(n.isdigit() for n in numbers):
            return sum(int(n) for n in numbers)
        case _:
            return "Unknown command"

# Pattern matching with objects
match event:
    case {"type": "click", "x": x, "y": y}:
        handle_click(x, y)
    case {"type": "keypress", "key": key}:
        handle_key(key)
    case {"type": "scroll", "direction": "up" | "down" as direction}:
        handle_scroll(direction)
```

### Iteration Patterns

```python
# zip — iterate multiple sequences in parallel
names = ["Alice", "Bob", "Charlie"]
scores = [95, 87, 92]
for name, score in zip(names, scores):
    print(f"{name}: {score}")

# zip_longest — don't stop at shortest
from itertools import zip_longest
for a, b in zip_longest([1, 2, 3], [10, 20], fillvalue=0):
    print(a, b)  # (1,10), (2,20), (3,0)

# enumerate with start
for i, line in enumerate(lines, start=1):
    print(f"Line {i}: {line}")

# reversed and sorted
for item in reversed(items):
    process(item)

for item in sorted(items, key=lambda x: x.priority, reverse=True):
    process(item)

# Dictionary iteration patterns
for key in sorted(config.keys()):
    print(f"{key} = {config[key]}")

# Nested loop with product
from itertools import product
for x, y in product(range(3), range(3)):
    print(f"({x}, {y})")
```

### Comprehensions — Advanced

```python
# Nested comprehension (flatten)
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]  # [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Conditional expression in comprehension
labels = ["PASS" if score >= 60 else "FAIL" for score in scores]

# Dictionary comprehension with filtering
env_vars = {k: v for k, v in os.environ.items() if k.startswith("APP_")}

# Set comprehension
unique_domains = {email.split("@")[1] for email in emails}

# Walrus in comprehension (filter + transform)
results = [cleaned for raw in data if (cleaned := clean(raw)) is not None]
```

---

## Functions — Advanced Patterns

### Parameter Types

```python
def complex_function(
    required_arg,                    # positional or keyword
    *args,                           # variable positional
    keyword_only,                    # must be passed by name
    default_kwarg="default",         # keyword with default
    **kwargs                         # variable keyword
):
    pass

# Positional-only parameters (Python 3.8+)
def divmod(a, b, /):  # a and b MUST be positional
    return a // b, a % b

# Combined
def api_call(url, /, *, method="GET", headers=None, **params):
    # url is positional-only
    # method, headers are keyword-only
    # params catches extra keyword args
    pass
```

### Closures and Factories

```python
def make_validator(min_val, max_val, required=True):
    """Factory: creates a validation function with captured config."""
    def validate(value):
        if value is None:
            return not required
        return min_val <= value <= max_val
    return validate

validate_age = make_validator(0, 150)
validate_score = make_validator(0, 100)
validate_optional_rating = make_validator(1, 5, required=False)

validate_age(25)    # True
validate_age(200)   # False
validate_optional_rating(None)  # True
```

### Partial Application

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

square(5)  # 25
cube(3)    # 27

# Practical: configure a logger
import logging
debug = partial(logging.log, logging.DEBUG)
error = partial(logging.log, logging.ERROR)
debug("This is a debug message")
```

### Function Annotations and Overloading

```python
from typing import overload

@overload
def process(data: str) -> list[str]: ...
@overload
def process(data: list[str]) -> str: ...

def process(data):
    if isinstance(data, str):
        return data.split()
    elif isinstance(data, list):
        return " ".join(data)
```

---

## Modules and Packages

### Module System

```python
# Every .py file is a module
# __name__ is "__main__" when run directly, module name when imported

# my_module.py
def helper():
    return "I'm a helper"

_private_var = "not exported by convention"

if __name__ == "__main__":
    # Only runs when executed directly, not when imported
    print(helper())
```

### Package Structure

```text
my_package/
├── __init__.py          # Makes it a package, runs on import
├── core.py
├── utils/
│   ├── __init__.py
│   ├── strings.py
│   └── numbers.py
└── _internal.py         # Convention: private module
```

```python
# __init__.py — control what's exported
from .core import MainClass, main_function
from .utils.strings import clean_text

__all__ = ["MainClass", "main_function", "clean_text"]
# Controls what `from my_package import *` exports
```

### Import Mechanics

```python
# Python searches for modules in this order:
# 1. sys.modules cache (already imported)
# 2. Built-in modules
# 3. sys.path (current dir, PYTHONPATH, site-packages)

import sys
print(sys.path)  # shows search order

# Lazy imports (defer heavy imports)
def process_image(path):
    from PIL import Image  # only imported when function is called
    return Image.open(path)

# Conditional imports
try:
    import ujson as json  # faster JSON library
except ImportError:
    import json  # fallback to stdlib
```

---

## Hypothetical Use Cases

### Use Case: CLI Argument Parser

```python
def parse_args(argv: list[str]) -> dict:
    """Simple CLI argument parser using pattern matching."""
    args = {"verbose": False, "output": "stdout", "files": []}
    
    i = 0
    while i < len(argv):
        match argv[i]:
            case "-v" | "--verbose":
                args["verbose"] = True
            case "-o" | "--output":
                i += 1
                args["output"] = argv[i]
            case arg if not arg.startswith("-"):
                args["files"].append(arg)
            case unknown:
                raise ValueError(f"Unknown argument: {unknown}")
        i += 1
    
    return args
```

### Use Case: Plugin System

```python
import importlib
from pathlib import Path

def load_plugins(plugin_dir: str) -> dict:
    """Dynamically load all plugins from a directory."""
    plugins = {}
    
    for path in Path(plugin_dir).glob("*.py"):
        if path.name.startswith("_"):
            continue
        
        module_name = path.stem
        spec = importlib.util.spec_from_file_location(module_name, path)
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        
        # Each plugin must have a register() function
        if hasattr(module, "register"):
            plugin_info = module.register()
            plugins[plugin_info["name"]] = module
    
    return plugins
```

---

## Key Takeaways

1. **Pattern matching** (3.10+) replaces complex if/elif chains for structural data
2. **Comprehensions** are idiomatic Python — prefer them over manual loops for transforms
3. **Closures and factories** create configured functions without classes
4. **Positional-only and keyword-only** parameters make APIs clearer
5. **`__init__.py`** controls package exports — use `__all__` for explicit public API
6. **Lazy imports** improve startup time for heavy dependencies
