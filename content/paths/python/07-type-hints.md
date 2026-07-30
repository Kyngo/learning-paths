---
title: "Python: Type Hints and Static Analysis"
weight: 7
---

## Why Type Hints

```python
# Without types — what does this accept? What does it return?
def process(data, config):
    ...

# With types — self-documenting, IDE-supported, statically checkable
def process(data: list[dict[str, Any]], config: ProcessConfig) -> ProcessResult:
    ...
```

**Benefits:**

- IDE autocompletion and error detection
- Static analysis catches bugs before runtime
- Self-documenting code
- Refactoring confidence

**Important:** Type hints are NOT enforced at runtime by default. They're for tools (mypy, pyright, IDEs).

---

## Basic Type Annotations

```python
from typing import Any

# Variables
name: str = "Alice"
age: int = 30
active: bool = True
score: float = 95.5

# Functions
def greet(name: str, excited: bool = False) -> str:
    if excited:
        return f"Hello, {name}!!!"
    return f"Hello, {name}"

# None return
def log(message: str) -> None:
    print(message)

# Collections (Python 3.9+ — use built-in types directly)
numbers: list[int] = [1, 2, 3]
mapping: dict[str, int] = {"a": 1, "b": 2}
coordinates: tuple[float, float] = (3.14, 2.71)
unique: set[str] = {"apple", "banana"}

# Variable-length tuple
values: tuple[int, ...] = (1, 2, 3, 4, 5)
```

---

## Union Types and Optional

```python
# Union — accepts multiple types
def parse_id(value: int | str) -> int:  # Python 3.10+ syntax
    if isinstance(value, str):
        return int(value)
    return value

# Optional — shorthand for X | None
def find_user(user_id: int) -> dict | None:
    """Returns user dict or None if not found."""
    ...

# Pre-3.10 syntax (still valid)
from typing import Union, Optional
def parse_id(value: Union[int, str]) -> int: ...
def find_user(user_id: int) -> Optional[dict]: ...
```

---

## Generics

```python
from typing import TypeVar, Generic

T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

# Generic function
def first(items: list[T]) -> T | None:
    return items[0] if items else None

# The return type matches the input type
first([1, 2, 3])      # inferred as int | None
first(["a", "b"])      # inferred as str | None

# Bounded TypeVar — restricts to specific types
from typing import TypeVar
Numeric = TypeVar("Numeric", int, float)

def add(a: Numeric, b: Numeric) -> Numeric:
    return a + b

# Generic class
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items.pop()
    
    def peek(self) -> T:
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items[-1]
    
    def __len__(self) -> int:
        return len(self._items)

# Usage — type checker knows the element type
int_stack: Stack[int] = Stack()
int_stack.push(42)
value: int = int_stack.pop()  # Type checker knows this is int
```

---

## Protocols — Structural Typing (Duck Typing with Types)

```python
from typing import Protocol, runtime_checkable

# Protocol defines what methods/attributes an object must have
# WITHOUT requiring inheritance

@runtime_checkable
class Renderable(Protocol):
    def render(self) -> str: ...

@runtime_checkable
class Sized(Protocol):
    def __len__(self) -> int: ...

class HTMLWidget:
    """No inheritance from Renderable — just implements the method."""
    def render(self) -> str:
        return "<div>Widget</div>"

class JSONResponse:
    def render(self) -> str:
        return '{"status": "ok"}'

# Both satisfy Renderable without inheriting from it
def display(item: Renderable) -> None:
    print(item.render())

display(HTMLWidget())    # OK
display(JSONResponse())  # OK

# Runtime check (because @runtime_checkable)
isinstance(HTMLWidget(), Renderable)  # True

# Complex protocol
class Repository(Protocol[T]):
    def get(self, id: str) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def delete(self, id: str) -> bool: ...
    def list(self, limit: int = 100) -> list[T]: ...
```

---

## TypedDict — Typed Dictionaries

```python
from typing import TypedDict, Required, NotRequired

class UserProfile(TypedDict):
    name: str
    email: str
    age: int
    bio: NotRequired[str]  # Optional key

class APIResponse(TypedDict):
    status: int
    data: dict[str, Any]
    errors: NotRequired[list[str]]

def create_user(profile: UserProfile) -> APIResponse:
    # Type checker knows exactly which keys exist and their types
    print(profile["name"])  # OK — str
    # print(profile["phone"])  # Error — key doesn't exist
    
    return {"status": 201, "data": {"id": "123", **profile}}

# TypedDict with inheritance
class BaseEvent(TypedDict):
    event_type: str
    timestamp: str

class ClickEvent(BaseEvent):
    x: int
    y: int
    element_id: str
```

---

## Callable Types

```python
from typing import Callable, ParamSpec, Concatenate
from collections.abc import Callable as CallableABC

# Function that accepts a callback
def apply_operation(
    values: list[int],
    operation: Callable[[int], int]  # Takes int, returns int
) -> list[int]:
    return [operation(v) for v in values]

apply_operation([1, 2, 3], lambda x: x * 2)  # [2, 4, 6]

# ParamSpec — preserve parameter types through decorators
P = ParamSpec("P")
R = TypeVar("R")

def logged(func: Callable[P, R]) -> Callable[P, R]:
    """Decorator that preserves the exact signature of the wrapped function."""
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logged
def add(a: int, b: int) -> int:
    return a + b

# Type checker knows add still takes (int, int) -> int
```

---

## Literal, Final, and TypeGuard

```python
from typing import Literal, Final, TypeGuard

# Literal — restrict to specific values
def set_direction(direction: Literal["north", "south", "east", "west"]) -> None:
    ...

set_direction("north")  # OK
set_direction("up")     # Type error!

# Final — cannot be reassigned or overridden
MAX_RETRIES: Final = 3
MAX_RETRIES = 5  # Type error!

class Base:
    @final
    def critical_method(self) -> None:
        """Subclasses cannot override this."""
        ...

# TypeGuard — narrow types in conditional checks
def is_string_list(val: list[Any]) -> TypeGuard[list[str]]:
    """Returns True if all elements are strings."""
    return all(isinstance(x, str) for x in val)

def process(data: list[Any]) -> None:
    if is_string_list(data):
        # Type checker now knows data is list[str]
        for s in data:
            print(s.upper())  # OK — s is str
```

---

## Type Aliases and NewType

```python
from typing import NewType, TypeAlias

# Type alias — just a shorthand (no runtime distinction)
JSON: TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None
Headers: TypeAlias = dict[str, str]

def fetch(url: str, headers: Headers) -> JSON:
    ...

# NewType — creates a distinct type (catches mixing up same-typed values)
UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)

def get_user(user_id: UserId) -> dict:
    ...

def get_order(order_id: OrderId) -> dict:
    ...

uid = UserId(42)
oid = OrderId(42)

get_user(uid)  # OK
get_user(oid)  # Type error! OrderId is not UserId
get_user(42)   # Type error! int is not UserId
```

---

## Overload — Multiple Signatures

```python
from typing import overload

@overload
def fetch(url: str, raw: Literal[True]) -> bytes: ...
@overload
def fetch(url: str, raw: Literal[False] = ...) -> str: ...

def fetch(url: str, raw: bool = False) -> bytes | str:
    response = requests.get(url)
    if raw:
        return response.content
    return response.text

# Type checker knows:
result1 = fetch("http://example.com", raw=True)   # bytes
result2 = fetch("http://example.com", raw=False)  # str
result3 = fetch("http://example.com")             # str
```

---

## Running Static Analysis

### mypy Configuration

```ini
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_any_generics = true
check_untyped_defs = true

# Per-module overrides
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = "third_party_lib.*"
ignore_missing_imports = true
```

```bash
# Run mypy
mypy src/
mypy src/ --strict
mypy src/module.py --show-error-codes
```

### Common mypy Errors and Fixes

```python
# Error: Incompatible return value type (got "Optional[str]", expected "str")
def get_name(user: dict) -> str:
    return user.get("name")  # .get() returns Optional[str]

# Fix: handle the None case
def get_name(user: dict) -> str:
    name = user.get("name")
    if name is None:
        raise ValueError("User has no name")
    return name

# Error: Item "None" of "Optional[User]" has no attribute "email"
user = find_user(123)
print(user.email)  # user might be None!

# Fix: narrow the type
user = find_user(123)
if user is not None:
    print(user.email)  # OK — type narrowed to User

# Or use assert (for cases where None is a bug)
user = find_user(123)
assert user is not None, f"User 123 must exist"
print(user.email)
```

---

## Hypothetical Use Case: Typed Event System

```python
from typing import TypeVar, Generic, Callable, Protocol
from dataclasses import dataclass

E = TypeVar("E", bound="Event")

@dataclass(frozen=True)
class Event:
    """Base event class."""
    pass

@dataclass(frozen=True)
class UserCreated(Event):
    user_id: str
    email: str

@dataclass(frozen=True)
class OrderPlaced(Event):
    order_id: str
    user_id: str
    total: float

class EventHandler(Protocol[E]):
    def __call__(self, event: E) -> None: ...

class EventBus:
    def __init__(self) -> None:
        self._handlers: dict[type[Event], list[Callable]] = {}
    
    def subscribe(self, event_type: type[E], handler: EventHandler[E]) -> None:
        self._handlers.setdefault(event_type, []).append(handler)
    
    def publish(self, event: Event) -> None:
        for handler in self._handlers.get(type(event), []):
            handler(event)

# Usage — fully type-safe
bus = EventBus()

def on_user_created(event: UserCreated) -> None:
    send_welcome_email(event.email)

def on_order_placed(event: OrderPlaced) -> None:
    process_payment(event.order_id, event.total)

bus.subscribe(UserCreated, on_user_created)
bus.subscribe(OrderPlaced, on_order_placed)

bus.publish(UserCreated(user_id="u1", email="alice@example.com"))
```

---

## Key Takeaways

1. **Type hints don't run** — they're for tools (mypy, pyright, IDEs)
2. **Use `X | None`** instead of `Optional[X]` (Python 3.10+)
3. **Protocols** enable duck typing with type safety — no inheritance required
4. **Generics** make reusable, type-safe containers and functions
5. **NewType** prevents mixing up same-typed values (UserId vs OrderId)
6. **TypedDict** types dictionary shapes — essential for JSON/API data
7. **ParamSpec** preserves function signatures through decorators
8. **Start with `mypy --strict`** on new projects — retrofitting is harder
