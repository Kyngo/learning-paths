---
title: "Property Testing & Fuzzing"
weight: 7
---

# Property Testing & Fuzzing

Traditional unit tests verify specific examples. Property-based testing verifies that **invariants hold across all possible inputs**, automatically generating hundreds or thousands of test cases. Fuzzing takes this further by feeding random or mutated data to find crashes and security vulnerabilities.

---

## Property-Based Testing

### The Core Idea

Instead of writing:
```python
def test_sort_specific():
    assert sort([3, 1, 2]) == [1, 2, 3]
```

You write:
```python
def test_sort_property(random_list):
    result = sort(random_list)
    assert len(result) == len(random_list)      # preserves length
    assert all(result[i] <= result[i+1] for i in range(len(result)-1))  # ordered
    assert set(result) == set(random_list)      # same elements
```

The framework generates many random lists and verifies the properties hold for all of them.

### Properties vs Examples

| Aspect | Example-Based Tests | Property-Based Tests |
|--------|-------------------|---------------------|
| Input | Hand-picked by developer | Auto-generated (hundreds/thousands) |
| Coverage | Tests what you think of | Finds edge cases you don't think of |
| Maintenance | Update when requirements change | Properties are more stable |
| Debugging | Failure is obvious | Framework shrinks to minimal failing case |
| Best for | Known edge cases, regression | Invariants, encoders/decoders, parsers |

---

## Hypothesis (Python)

[Hypothesis](https://hypothesis.readthedocs.io/) is the standard property-based testing library for Python.

### Basic Usage

```python
from hypothesis import given
from hypothesis import strategies as st

@given(st.lists(st.integers()))
def test_sorted_list_is_ordered(xs):
    result = sorted(xs)
    for i in range(len(result) - 1):
        assert result[i] <= result[i + 1]

@given(st.lists(st.integers()))
def test_sort_preserves_length(xs):
    assert len(sorted(xs)) == len(xs)

@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs):
    assert sorted(xs) == sorted(sorted(xs))
```

### Strategies (Generators)

Strategies define how to generate test data:

```python
from hypothesis import strategies as st

# Primitives
st.integers()                          # Any integer
st.integers(min_value=0, max_value=100)  # Bounded
st.floats(allow_nan=False)             # Floats without NaN
st.text(min_size=1, max_size=50)       # Non-empty strings
st.booleans()                          # True or False

# Collections
st.lists(st.integers(), min_size=1)    # Non-empty list of ints
st.dictionaries(st.text(), st.integers())  # Dict[str, int]
st.tuples(st.integers(), st.text())    # Tuple[int, str]

# Composite / structured
st.one_of(st.integers(), st.text())    # Union type
st.none() | st.integers()              # Optional[int]
```

### Custom Strategies

```python
from hypothesis import strategies as st
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    email: str

# Build a strategy for User objects
user_strategy = st.builds(
    User,
    name=st.text(min_size=1, max_size=50, alphabet=st.characters(whitelist_categories=("L",))),
    age=st.integers(min_value=0, max_value=150),
    email=st.emails(),
)

@given(user_strategy)
def test_user_serialization_roundtrip(user):
    serialized = user_to_json(user)
    deserialized = user_from_json(serialized)
    assert deserialized == user
```

### Composite Strategies

For complex data with constraints:

```python
from hypothesis import strategies as st
from hypothesis.strategies import composite

@composite
def sorted_lists(draw):
    """Generate a pre-sorted list of integers."""
    xs = draw(st.lists(st.integers(), min_size=1))
    return sorted(xs)

@composite
def valid_date_ranges(draw):
    """Generate a (start, end) where start <= end."""
    start = draw(st.dates())
    end = draw(st.dates(min_value=start))
    return (start, end)
```

---

## Shrinking

When Hypothesis finds a failing input, it **shrinks** it to the minimal reproducing case.

```python
@given(st.lists(st.integers()))
def test_no_duplicates_after_dedup(xs):
    result = deduplicate(xs)
    assert len(result) == len(set(result))
```

If the function has a bug with `[0, 0]`, Hypothesis might first find failure with `[437, -28, 0, 0, 91, 0]`, then automatically shrink to `[0, 0]` — the simplest failing case.

### How Shrinking Works

1. Hypothesis finds a failing input
2. It tries simpler variants (shorter lists, smaller numbers, simpler strings)
3. Each variant that still fails becomes the new candidate
4. Process continues until no simpler failing case exists
5. The minimal case is reported

---

## Common Properties (Invariants)

| Property | Description | Example |
|----------|-------------|---------|
| **Roundtrip** | encode then decode = identity | `decode(encode(x)) == x` |
| **Idempotency** | Applying twice = applying once | `f(f(x)) == f(x)` |
| **Commutativity** | Order doesn't matter | `a + b == b + a` |
| **Associativity** | Grouping doesn't matter | `(a + b) + c == a + (b + c)` |
| **Invariant preservation** | Some property always holds | `len(sort(x)) == len(x)` |
| **Oracle** | Compare against reference impl | `my_sort(x) == sorted(x)` |
| **No crash** | Function doesn't throw for valid input | No exception raised |

### Real-World Property Examples

```python
from hypothesis import given
from hypothesis import strategies as st
import json

# Roundtrip: JSON serialization
@given(st.recursive(
    st.none() | st.booleans() | st.integers() | st.floats(allow_nan=False) | st.text(),
    lambda children: st.lists(children) | st.dictionaries(st.text(), children),
))
def test_json_roundtrip(value):
    assert json.loads(json.dumps(value)) == value

# Invariant: reversing twice = original
@given(st.lists(st.integers()))
def test_reverse_involution(xs):
    assert list(reversed(list(reversed(xs)))) == xs

# Oracle: compare custom implementation against stdlib
@given(st.lists(st.integers()))
def test_my_sort_matches_stdlib(xs):
    assert my_sort(xs) == sorted(xs)
```

---

## fast-check (JavaScript/TypeScript)

The equivalent library for JavaScript:

```javascript
const fc = require('fast-check');

// Property: sort preserves length
fc.assert(
  fc.property(fc.array(fc.integer()), (arr) => {
    const sorted = [...arr].sort((a, b) => a - b);
    return sorted.length === arr.length;
  })
);

// Property: JSON roundtrip
fc.assert(
  fc.property(fc.jsonValue(), (value) => {
    return JSON.parse(JSON.stringify(value)) === value
      || JSON.stringify(JSON.parse(JSON.stringify(value))) === JSON.stringify(value);
  })
);

// Custom arbitraries
const userArbitrary = fc.record({
  name: fc.string({ minLength: 1 }),
  age: fc.integer({ min: 0, max: 150 }),
  email: fc.emailAddress(),
});

fc.assert(
  fc.property(userArbitrary, (user) => {
    const result = validateUser(user);
    return result.valid === true;
  })
);
```

---

## Fuzzing

Fuzzing feeds random, malformed, or mutated inputs to a program to discover crashes, hangs, and security vulnerabilities.

### Types of Fuzzing

| Type | Approach | Tools |
|------|----------|-------|
| **Dumb fuzzing** | Completely random input | Basic scripts |
| **Mutation fuzzing** | Mutate valid inputs | AFL, LibFuzzer |
| **Generation fuzzing** | Generate from grammar/spec | Hypothesis, Peach |
| **Coverage-guided** | Use code coverage to guide exploration | AFL++, LibFuzzer |

### Fuzzing with Hypothesis

```python
from hypothesis import given, settings
from hypothesis import strategies as st

@settings(max_examples=10000)  # Run many more cases
@given(st.binary())
def test_parser_doesnt_crash(data):
    """Parser should never crash on arbitrary input."""
    try:
        parse(data)
    except ParseError:
        pass  # Expected for invalid input
    # No other exceptions should escape

@given(st.text())
def test_no_sql_injection(user_input):
    """Query builder should safely handle any input."""
    query = build_query(user_input)
    assert "DROP" not in query or is_properly_escaped(query)
```

### Security-Focused Fuzzing

```python
@given(st.text(alphabet=st.characters(whitelist_categories=("L", "N", "P", "S"))))
def test_xss_prevention(user_input):
    """HTML sanitizer removes all script content."""
    sanitized = sanitize_html(user_input)
    assert "<script" not in sanitized.lower()
    assert "javascript:" not in sanitized.lower()
    assert "onerror" not in sanitized.lower()

@given(st.binary(min_size=1, max_size=10000))
def test_image_parser_handles_malformed(data):
    """Image parser rejects malformed data without crashing."""
    try:
        parse_image(data)
    except InvalidImageError:
        pass  # Expected
    # Memory errors, segfaults, hangs = test failure
```

---

## Mutation Testing

Mutation testing evaluates **test quality** by introducing small changes (mutations) to source code and checking if tests catch them.

### How It Works

1. Tool creates **mutants** (modified versions of your code)
2. Each mutant has a small change: `>` → `>=`, `+` → `-`, `True` → `False`
3. Tests run against each mutant
4. If tests pass → **survived mutant** (tests are weak)
5. If tests fail → **killed mutant** (tests caught the change)

### Common Mutations

| Category | Original | Mutant |
|----------|----------|--------|
| Arithmetic | `a + b` | `a - b` |
| Comparison | `x > 0` | `x >= 0` or `x < 0` |
| Boolean | `and` | `or` |
| Return value | `return True` | `return False` |
| Boundary | `range(n)` | `range(n + 1)` |
| Removal | `if condition:` | (remove block) |

### Tools

| Language | Tool |
|----------|------|
| Python | mutmut, cosmic-ray |
| JavaScript | Stryker |
| Java | PIT |
| Go | go-mutesting |

### Example: mutmut (Python)

```bash
# Run mutation testing
mutmut run --paths-to-mutate=src/

# View results
mutmut results

# Inspect a surviving mutant
mutmut show 42
```

A **mutation score** of 80%+ indicates good test coverage quality (not just line coverage).

---

## Integrating Property Tests in CI

```toml
# pyproject.toml
[tool.hypothesis]
# Use more examples in CI, fewer locally for speed
[tool.hypothesis.profiles.ci]
max_examples = 1000
deadline = 5000  # ms

[tool.hypothesis.profiles.dev]
max_examples = 100
deadline = 1000
```

```python
# conftest.py
from hypothesis import settings
import os

if os.environ.get("CI"):
    settings.load_profile("ci")
else:
    settings.load_profile("dev")
```

---

## Key Takeaways

- Property-based testing finds edge cases you'd never think to write manually — especially boundary conditions, empty inputs, and encoding issues
- **Shrinking** is the killer feature — when a test fails, the framework automatically finds the simplest reproducing case
- Think in **properties** (roundtrip, idempotency, invariants) rather than specific input-output pairs
- **Hypothesis** (Python) and **fast-check** (JS) are mature, well-maintained libraries ready for production use
- **Fuzzing** extends property testing to security — feeding random data to parsers, validators, and serializers catches vulnerabilities
- **Mutation testing** measures test quality — surviving mutants reveal where your test suite has blind spots
- Combine property tests with example-based tests: properties cover breadth, examples document specific expected behaviors
