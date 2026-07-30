---
title: "Python: Packaging and Distribution"
weight: 9
---

## Project Structure

```text
my-project/
├── pyproject.toml          # THE single source of truth for project metadata
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── core.py
│       ├── utils.py
│       └── py.typed        # Marker file: "this package ships type stubs"
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_core.py
│   └── test_utils.py
├── docs/
│   └── index.md
├── README.md
├── LICENSE
└── .gitignore
```

### Why `src/` Layout?

```text
# Without src/ — accidental imports from project root
my_project/
├── my_package/     ← Python might import THIS instead of installed version
├── tests/
└── pyproject.toml

# With src/ — forces you to install the package to import it
my_project/
├── src/
│   └── my_package/  ← Only importable after `pip install -e .`
├── tests/
└── pyproject.toml
```

---

## pyproject.toml — Modern Configuration

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-package"
version = "1.2.0"
description = "A useful Python package"
readme = "README.md"
license = "MIT"
requires-python = ">=3.11"
authors = [
    { name = "Alice Developer", email = "alice@example.com" },
]
keywords = ["utility", "tools"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Typing :: Typed",
]

dependencies = [
    "httpx>=0.25.0",
    "pydantic>=2.0,<3.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "mypy>=1.5",
    "ruff>=0.1.0",
]
docs = [
    "mkdocs>=1.5",
    "mkdocs-material>=9.0",
]

[project.scripts]
my-cli = "my_package.cli:main"

[project.urls]
Homepage = "https://github.com/alice/my-package"
Documentation = "https://my-package.readthedocs.io"
Repository = "https://github.com/alice/my-package"

# Tool configuration in the same file
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"

[tool.mypy]
python_version = "3.12"
strict = true

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "SIM"]
```

---

## Dependency Management

### Version Specifiers

| Specifier | Meaning | Example |
|-----------|---------|---------|
| `>=1.0` | At least 1.0 | `httpx>=0.25.0` |
| `>=1.0,<2.0` | Compatible range | `pydantic>=2.0,<3.0` |
| `~=1.4` | Compatible release (>=1.4, <2.0) | `~=1.4.2` means >=1.4.2, <1.5 |
| `==1.2.3` | Exact version | Pin for reproducibility |

### Virtual Environments

```bash
# Create and activate
python -m venv .venv
source .venv/bin/activate  # macOS/Linux

# Install in development mode (editable)
pip install -e ".[dev]"

# Lock dependencies for reproducibility
pip freeze > requirements.lock

# Or use pip-tools for better dependency resolution
pip install pip-tools
pip-compile pyproject.toml -o requirements.lock
pip-sync requirements.lock
```

### uv — Modern Package Manager (Faster Alternative)

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create project
uv init my-project
cd my-project

# Add dependencies
uv add httpx pydantic
uv add --dev pytest mypy ruff

# Sync environment
uv sync

# Run commands in the environment
uv run pytest
uv run mypy src/

# Lock file is automatic (uv.lock)
```

---

## Building and Publishing

### Build the Package

```bash
# Install build tool
pip install build

# Build sdist and wheel
python -m build

# Output:
# dist/
# ├── my_package-1.2.0.tar.gz      (source distribution)
# └── my_package-1.2.0-py3-none-any.whl  (wheel — faster to install)
```

### Publish to PyPI

```bash
# Install twine
pip install twine

# Upload to Test PyPI first
twine upload --repository testpypi dist/*

# Upload to real PyPI
twine upload dist/*

# Or use trusted publishing (GitHub Actions → PyPI without tokens)
```

### GitHub Actions CI/CD

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - run: pip install -e ".[dev]"
      - run: ruff check src/
      - run: mypy src/
      - run: pytest --cov=my_package --cov-report=xml
  
  publish:
    needs: test
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Trusted publishing
    
    steps:
      - uses: actions/checkout@v4
      - run: pip install build
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
```

---

## Entry Points and CLI Tools

```python
# src/my_package/cli.py
import argparse
import sys

def main():
    parser = argparse.ArgumentParser(description="My CLI tool")
    parser.add_argument("input", help="Input file path")
    parser.add_argument("-o", "--output", default="stdout", help="Output destination")
    parser.add_argument("-v", "--verbose", action="store_true")
    
    args = parser.parse_args()
    
    try:
        result = process(args.input)
        if args.output == "stdout":
            print(result)
        else:
            Path(args.output).write_text(result)
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    main()
```

```toml
# In pyproject.toml
[project.scripts]
my-cli = "my_package.cli:main"
# After `pip install`, `my-cli` is available as a command
```

---

## Namespace Packages

```text
# Multiple packages sharing a namespace (no __init__.py at namespace level)
company-auth/
└── src/
    └── company/        ← NO __init__.py here
        └── auth/
            ├── __init__.py
            └── tokens.py

company-logging/
└── src/
    └── company/        ← NO __init__.py here
        └── logging/
            ├── __init__.py
            └── handlers.py

# Both installable, both importable:
from company.auth.tokens import create_token
from company.logging.handlers import JSONHandler
```

---

## Hypothetical Use Case: Internal Library

```toml
# pyproject.toml for an internal company library
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "acme-utils"
version = "2.1.0"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.25",
    "pydantic>=2.0,<3.0",
    "structlog>=23.0",
]

[project.optional-dependencies]
aws = ["boto3>=1.28"]
testing = ["pytest>=7.0", "pytest-asyncio>=0.21", "respx>=0.20"]

[tool.hatch.build.targets.wheel]
packages = ["src/acme"]
```

```bash
# Install from private registry
pip install acme-utils --index-url https://pypi.internal.company.com/simple/

# Or from git
pip install git+ssh://git@github.com/acme/acme-utils.git@v2.1.0
```

---

## Key Takeaways

1. **`pyproject.toml` is the standard** — replaces setup.py, setup.cfg, requirements.txt for metadata
2. **Use `src/` layout** — prevents accidental imports from the project root
3. **Pin ranges for libraries** (`>=1.0,<2.0`), **lock exact versions for applications**
4. **`uv`** is the modern, fast alternative to pip + venv + pip-tools
5. **Wheels are pre-built** — faster to install than source distributions
6. **Entry points** (`[project.scripts]`) create CLI commands on install
7. **Trusted publishing** (OIDC) eliminates the need for PyPI API tokens in CI
