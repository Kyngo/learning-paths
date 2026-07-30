---
title: "Error Handling Patterns"
weight: 8
---

## Error Philosophy in JavaScript

JavaScript has a unique error landscape: dynamic typing means errors that other languages catch at compile time only surface at runtime. TypeScript helps, but runtime errors from external data, network failures, and edge cases are inevitable.

---

## The Error Object

```javascript
// Built-in error types
new Error("Something went wrong");          // generic
new TypeError("Expected string, got number"); // wrong type
new RangeError("Index out of bounds");       // value out of range
new ReferenceError("x is not defined");      // undeclared variable
new SyntaxError("Unexpected token");         // parse error
new URIError("Invalid URI");                 // malformed URI

// Error properties
const err = new Error("Connection failed");
err.message;    // "Connection failed"
err.name;       // "Error"
err.stack;      // Stack trace (non-standard but universal)
err.cause;      // Chained error (ES2022)
```

### Custom Error Classes

```typescript
class AppError extends Error {
    constructor(
        message: string,
        public readonly code: string,
        public readonly statusCode: number = 500,
        options?: ErrorOptions
    ) {
        super(message, options);
        this.name = "AppError";
    }
}

class NotFoundError extends AppError {
    constructor(resource: string, id: string) {
        super(`${resource} with id '${id}' not found`, "NOT_FOUND", 404);
        this.name = "NotFoundError";
    }
}

class ValidationError extends AppError {
    constructor(
        message: string,
        public readonly fields: Record<string, string>
    ) {
        super(message, "VALIDATION_ERROR", 400);
        this.name = "ValidationError";
    }
}

// Usage with error cause (ES2022)
async function fetchUser(id: string) {
    try {
        const response = await fetch(`/api/users/${id}`);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return await response.json();
    } catch (err) {
        throw new AppError("Failed to fetch user", "FETCH_ERROR", 502, { cause: err });
    }
}
```

---

## Try/Catch Patterns

### Synchronous

```typescript
function parseJSON<T>(raw: string): T {
    try {
        return JSON.parse(raw);
    } catch (err) {
        throw new ValidationError("Invalid JSON input", {
            raw: "Must be valid JSON"
        });
    }
}

// Catch specific error types
function handleError(err: unknown) {
    if (err instanceof NotFoundError) {
        return { status: 404, body: { error: err.message } };
    }
    if (err instanceof ValidationError) {
        return { status: 400, body: { error: err.message, fields: err.fields } };
    }
    // Unknown error — don't expose internals
    console.error("Unexpected error:", err);
    return { status: 500, body: { error: "Internal server error" } };
}
```

### Asynchronous

```typescript
// async/await — try/catch works naturally
async function processOrder(orderId: string) {
    try {
        const order = await fetchOrder(orderId);
        const payment = await chargePayment(order);
        await sendConfirmation(order, payment);
        return { success: true };
    } catch (err) {
        if (err instanceof PaymentError) {
            await refund(orderId);
            throw new AppError("Payment failed", "PAYMENT_FAILED", 402, { cause: err });
        }
        throw err; // re-throw unexpected errors
    } finally {
        await releaseInventoryLock(orderId); // always runs
    }
}
```

---

## The Result Pattern

Instead of throwing exceptions for expected failures, return a discriminated union that forces the caller to handle both paths:

```typescript
type Result<T, E = Error> =
    | { ok: true; value: T }
    | { ok: false; error: E };

// Helper constructors
const Ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
const Err = <E>(error: E): Result<never, E> => ({ ok: false, error });

// Usage
function divide(a: number, b: number): Result<number, string> {
    if (b === 0) return Err("Division by zero");
    return Ok(a / b);
}

const result = divide(10, 0);
if (result.ok) {
    console.log(result.value); // TypeScript knows this is number
} else {
    console.error(result.error); // TypeScript knows this is string
}
```

### When to Use Result vs Exceptions

| Use Exceptions (throw) | Use Result Pattern |
|------------------------|-------------------|
| Truly exceptional situations | Expected failure modes |
| Programming errors (bugs) | Validation failures |
| Unrecoverable failures | Business rule violations |
| Framework boundaries | Parsing/conversion |
| When caller can't reasonably handle it | When caller MUST handle both paths |

### Async Result

```typescript
type AsyncResult<T, E = Error> = Promise<Result<T, E>>;

async function fetchUser(id: string): AsyncResult<User, AppError> {
    try {
        const res = await fetch(`/api/users/${id}`);
        if (res.status === 404) {
            return Err(new NotFoundError("User", id));
        }
        if (!res.ok) {
            return Err(new AppError(`HTTP ${res.status}`, "HTTP_ERROR", res.status));
        }
        return Ok(await res.json());
    } catch (err) {
        return Err(new AppError("Network error", "NETWORK_ERROR", 0, { cause: err }));
    }
}

// Caller is forced to handle both cases
const result = await fetchUser("123");
if (!result.ok) {
    // Handle error — can't accidentally ignore it
    showError(result.error.message);
    return;
}
// result.value is User here (narrowed)
renderProfile(result.value);
```

---

## Error Handling in Express/HTTP APIs

```typescript
import express from 'express';

const app = express();

// Route handler with async error handling
app.get('/users/:id', async (req, res, next) => {
    try {
        const user = await userService.findById(req.params.id);
        if (!user) throw new NotFoundError("User", req.params.id);
        res.json(user);
    } catch (err) {
        next(err); // pass to error middleware
    }
});

// Global error handler (must have 4 params)
app.use((err: unknown, req: express.Request, res: express.Response, next: express.NextFunction) => {
    if (err instanceof AppError) {
        res.status(err.statusCode).json({
            error: { code: err.code, message: err.message }
        });
        return;
    }
    
    // Unexpected error — log full details, return generic message
    console.error("Unhandled error:", err);
    res.status(500).json({
        error: { code: "INTERNAL_ERROR", message: "An unexpected error occurred" }
    });
});
```

---

## Global Error Handlers

### Node.js Process-Level

```typescript
// Unhandled promise rejections (would crash in Node 15+)
process.on('unhandledRejection', (reason, promise) => {
    console.error('Unhandled Rejection:', reason);
    // Log, alert, then exit gracefully
    process.exit(1);
});

// Uncaught exceptions (synchronous throws with no catch)
process.on('uncaughtException', (err) => {
    console.error('Uncaught Exception:', err);
    // MUST exit — process state is unreliable
    process.exit(1);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down...');
    await server.close();
    await db.disconnect();
    process.exit(0);
});
```

### Browser

```javascript
// Global error handler
window.addEventListener('error', (event) => {
    reportError({
        message: event.message,
        source: event.filename,
        line: event.lineno,
        column: event.colno,
        error: event.error
    });
});

// Unhandled promise rejections
window.addEventListener('unhandledrejection', (event) => {
    reportError({ reason: event.reason });
    event.preventDefault(); // prevent console error
});
```

---

## Defensive Patterns

### Validation at Boundaries

```typescript
import { z } from 'zod';

// Validate external data at the boundary
const UserSchema = z.object({
    name: z.string().min(1).max(100),
    email: z.string().email(),
    age: z.number().int().min(0).max(150),
});

type User = z.infer<typeof UserSchema>;

function createUser(input: unknown): Result<User, ValidationError> {
    const parsed = UserSchema.safeParse(input);
    if (!parsed.success) {
        const fields = Object.fromEntries(
            parsed.error.issues.map(i => [i.path.join('.'), i.message])
        );
        return Err(new ValidationError("Invalid user data", fields));
    }
    return Ok(parsed.data);
}
```

### Null Safety

```typescript
// Optional chaining + nullish coalescing
const city = user?.address?.city ?? "Unknown";

// Assertion functions
function assertDefined<T>(val: T | null | undefined, msg: string): asserts val is T {
    if (val == null) throw new AppError(msg, "ASSERTION_FAILED");
}

// Non-null assertion (use sparingly — you're telling TS "trust me")
const element = document.getElementById("app")!; // throws at runtime if null
```

---

## Key Takeaways

1. **Custom error classes** — extend `Error` with `code`, `statusCode`, and structured data
2. **Error cause chain** — use `{ cause: originalError }` to preserve the full error trail
3. **Result pattern for expected failures** — forces callers to handle both paths; no silent ignoring
4. **Validate at boundaries** — external data (API input, file content, env vars) is untrusted
5. **Never expose internal errors** — log full details server-side, return generic messages to clients
6. **`unknown` over `any` in catch** — TypeScript catch clauses are `unknown`; narrow before accessing properties
7. **Global handlers are last resort** — `unhandledRejection` and `uncaughtException` should log and exit, not recover
