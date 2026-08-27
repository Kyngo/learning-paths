---
title: "Smart Pointers & Memory"
weight: 10
---

# Smart Pointers & Memory

Smart pointers are types that act like pointers but have additional metadata and capabilities. They manage memory automatically through ownership and the `Drop` trait. Understanding them is essential for complex data structures, interior mutability, and interop with C.

---

## Box<T> — Heap Allocation

`Box<T>` puts a value on the heap. The `Box` itself (a pointer) lives on the stack.

```rust
let b = Box::new(5);
println!("{b}");  // auto-derefs to i32
```

### When to Use Box

| Use Case | Why |
|----------|-----|
| Recursive types | Compiler needs a known size |
| Large data | Avoid stack overflow or expensive copies |
| Trait objects | `Box<dyn Trait>` for dynamic dispatch |
| Transferring ownership | When you need a heap allocation |

### Recursive Types

```rust
// ERROR: size of List is infinite
// enum List { Cons(i32, List), Nil }

// OK: Box gives a fixed-size pointer
enum List {
    Cons(i32, Box<List>),
    Nil,
}

let list = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
```

---

## Rc<T> — Reference Counting (Single-Threaded)

Multiple ownership via reference counting. The value is dropped when the last `Rc` is dropped.

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared"));
let b = Rc::clone(&a);  // increment reference count (cheap, no deep copy)
let c = Rc::clone(&a);

println!("count: {}", Rc::strong_count(&a));  // 3
drop(c);
println!("count: {}", Rc::strong_count(&a));  // 2
```

**`Rc<T>` is NOT thread-safe.** Use `Arc<T>` for multi-threaded code.

### Weak References

`Rc::downgrade` creates a weak reference that does not prevent dropping:

```rust
use std::rc::{Rc, Weak};

let strong = Rc::new(42);
let weak: Weak<i32> = Rc::downgrade(&strong);

// Upgrade weak to strong (may fail if value is dropped)
if let Some(value) = weak.upgrade() {
    println!("{value}");
}

drop(strong);
assert!(weak.upgrade().is_none());  // value is gone
```

Use weak references to break reference cycles (e.g., parent ↔ child in a tree).

---

## Arc<T> — Atomic Reference Counting (Thread-Safe)

Same as `Rc<T>` but uses atomic operations for the reference count:

```rust
use std::sync::Arc;

let data = Arc::new(vec![1, 2, 3]);
let data_clone = Arc::clone(&data);

std::thread::spawn(move || {
    println!("{:?}", data_clone);
});
```

`Arc` alone only gives shared immutable access. For mutation, combine with `Mutex` or `RwLock`.

---

## Interior Mutability

Normal Rust: you cannot mutate through an immutable reference. Interior mutability patterns move the borrow check to runtime.

### Cell<T> — Copy Types Only

```rust
use std::cell::Cell;

let cell = Cell::new(42);
cell.set(100);             // mutate through &Cell
let value = cell.get();    // 100
```

No references to the inner value — only `get`/`set`. Only works for `Copy` types.

### RefCell<T> — Runtime Borrow Checking

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

{
    let mut v = data.borrow_mut();  // runtime mutable borrow
    v.push(4);
}  // borrow released

let v = data.borrow();  // runtime immutable borrow
println!("{:?}", *v);    // [1, 2, 3, 4]

// PANICS at runtime if rules are violated:
// let a = data.borrow();
// let b = data.borrow_mut();  // panic: already borrowed
```

### When to Use Which

| Type | Thread Safe | Borrow Check | Copy Required | Use |
|------|:-----------:|:------------:|:-------------:|-----|
| `Cell<T>` | ✗ | None (get/set) | ✓ | Simple counters, flags |
| `RefCell<T>` | ✗ | Runtime | ✗ | Complex single-threaded mutation |
| `Mutex<T>` | ✓ | Runtime (lock) | ✗ | Multi-threaded mutation |
| `RwLock<T>` | ✓ | Runtime (read/write lock) | ✗ | Multiple readers, single writer |

### Rc<RefCell<T>> — Shared Mutable Single-Threaded

```rust
use std::rc::Rc;
use std::cell::RefCell;

let shared = Rc::new(RefCell::new(vec![1, 2, 3]));

let clone1 = Rc::clone(&shared);
clone1.borrow_mut().push(4);

let clone2 = Rc::clone(&shared);
println!("{:?}", clone2.borrow());  // [1, 2, 3, 4]
```

---

## Cow<T> — Clone on Write

`Cow` (Clone on Write) holds either a borrowed reference or an owned value. It clones only when mutation is needed:

```rust
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.contains("bad") {
        // Need to modify — clone into owned String
        Cow::Owned(input.replace("bad", "good"))
    } else {
        // No modification needed — just borrow
        Cow::Borrowed(input)
    }
}

let result = process("this is fine");     // Cow::Borrowed — no allocation
let result = process("this is bad");      // Cow::Owned — allocated
```

Use `Cow` when most inputs pass through unchanged but some need modification.

---

## Unsafe Rust

`unsafe` lets you bypass the borrow checker for operations the compiler cannot verify:

```rust
unsafe {
    // These are the ONLY things unsafe allows:
    // 1. Dereference raw pointers
    // 2. Call unsafe functions
    // 3. Access mutable statics
    // 4. Implement unsafe traits
    // 5. Access union fields
}
```

### Raw Pointers

```rust
let x = 42;
let raw = &x as *const i32;       // immutable raw pointer
let raw_mut = &mut x as *mut i32;  // mutable raw pointer

// Creating raw pointers is safe. Dereferencing them is unsafe.
unsafe {
    println!("{}", *raw);
}
```

### When Unsafe Is Appropriate

| Appropriate | Not Appropriate |
|-------------|----------------|
| FFI (calling C code) | Working around the borrow checker |
| Hardware access | Avoiding Clone/Arc costs |
| Performance-critical intrinsics | Convenience |
| Implementing data structures (linked lists) | Anything safe Rust can express |

### The Unsafe Contract

`unsafe` does not mean "no rules." It means "I (the programmer) am responsible for maintaining the invariants the compiler normally checks." The surrounding safe API must ensure that no safe code can cause undefined behaviour through the unsafe code.

---

## Memory Layout

```rust
use std::mem;

mem::size_of::<i32>()        // 4 bytes
mem::size_of::<&str>()       // 16 bytes (ptr + len)
mem::size_of::<String>()     // 24 bytes (ptr + len + cap)
mem::size_of::<Vec<i32>>()   // 24 bytes (ptr + len + cap)
mem::size_of::<Option<&str>>()  // 16 bytes (niche optimization — no extra tag)
mem::size_of::<Box<i32>>()   // 8 bytes (just a pointer)
```

### Niche Optimisation

`Option<&T>` is the same size as `&T` because the compiler uses the null value (which a reference can never be) to represent `None`. This also works for `Box<T>`, `NonZeroU32`, etc.

---

## Key Takeaways

- `Box<T>` allocates on the heap. Use it for recursive types, large data, and trait objects.
- `Rc<T>` enables shared ownership (single-threaded). `Arc<T>` is the thread-safe version.
- `RefCell<T>` moves borrow checking to runtime — panics on violation instead of compile error.
- `Cow<T>` avoids unnecessary cloning — borrows when possible, owns when necessary.
- `unsafe` is a tool, not a escape hatch. Use it for FFI, hardware, and data structures — with a safe wrapper.
- Rust's smart pointers compose: `Arc<Mutex<T>>` for thread-safe shared mutation, `Rc<RefCell<T>>` for single-threaded.
