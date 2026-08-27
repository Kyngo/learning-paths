---
title: "gRPC & Binary Protocols"
weight: 9
---

# gRPC & Binary Protocols

REST with JSON dominates public APIs, but service-to-service communication benefits from binary protocols — smaller payloads, schema enforcement, and code generation. gRPC is the industry standard for internal microservice communication.

---

## Why Binary Protocols?

| | REST + JSON | gRPC + Protobuf |
|-|-------------|----------------|
| Encoding | Text (UTF-8 JSON) | Binary (Protocol Buffers) |
| Payload size | Larger (~2-10×) | Smaller |
| Schema | Optional (OpenAPI) | Required (.proto files) |
| Code generation | Optional | Built-in (every language) |
| Streaming | Workarounds (SSE, WebSocket) | Native (4 streaming modes) |
| Browser support | Native | Requires gRPC-Web proxy |
| Human-readable | Yes | No (binary wire format) |
| Type safety | Runtime validation | Compile-time from .proto |
| Transport | HTTP/1.1 or HTTP/2 | HTTP/2 (required) |

---

## Protocol Buffers (Protobuf)

The serialisation format used by gRPC:

```protobuf
// user.proto
syntax = "proto3";

package user.v1;

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    repeated string roles = 4;
    optional string phone = 5;
    google.protobuf.Timestamp created_at = 6;
}

message GetUserRequest {
    int64 id = 1;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    string next_page_token = 2;
}
```

### Field Numbers

Field numbers are the wire identifier — **never reuse or change them**:

| Rule | Why |
|------|-----|
| Never change a field number | Old clients use the number to decode |
| Never reuse a deleted field number | Old messages in transit/storage would decode wrong |
| Reserve removed field numbers | `reserved 5, 8;` prevents reuse |
| Fields 1-15 use 1 byte on wire | Use them for frequently populated fields |

### Code Generation

```bash
protoc --go_out=. --go-grpc_out=. user.proto    # Go
protoc --python_out=. --grpc_python_out=. user.proto  # Python
protoc --java_out=. --grpc-java_out=. user.proto      # Java
```

Generated code includes typed structs/classes and client/server stubs.

---

## gRPC Service Definition

```protobuf
service UserService {
    // Unary — one request, one response
    rpc GetUser(GetUserRequest) returns (User);

    // Server streaming — one request, stream of responses
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client streaming — stream of requests, one response
    rpc UploadUsers(stream User) returns (UploadResponse);

    // Bidirectional streaming — both sides stream
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

### The Four Communication Patterns

| Pattern | Use Case |
|---------|----------|
| Unary | Simple request-response (most common) |
| Server streaming | Large result sets, real-time feeds |
| Client streaming | File upload, batch ingestion |
| Bidirectional | Chat, collaborative editing, real-time sync |

---

## gRPC in Practice

### Metadata (Headers)

```
// Like HTTP headers — carry auth tokens, request IDs, trace context
metadata = {"authorization": "Bearer <token>", "x-request-id": "abc-123"}
```

### Error Handling

gRPC uses status codes (not HTTP status codes):

| gRPC Code | Meaning | HTTP Equivalent |
|-----------|---------|----------------|
| `OK` | Success | 200 |
| `INVALID_ARGUMENT` | Client error | 400 |
| `NOT_FOUND` | Resource missing | 404 |
| `ALREADY_EXISTS` | Conflict | 409 |
| `PERMISSION_DENIED` | Forbidden | 403 |
| `UNAUTHENTICATED` | No valid credentials | 401 |
| `INTERNAL` | Server error | 500 |
| `UNAVAILABLE` | Service temporarily down | 503 |
| `DEADLINE_EXCEEDED` | Timeout | 504 |

### Deadlines (Timeouts)

Every gRPC call should have a deadline:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
```

Deadlines propagate through the call chain — if service A calls B calls C with a 5s deadline, C knows how much time is left.

---

## When to Use What

| Use REST | Use gRPC | Use GraphQL |
|----------|----------|-------------|
| Public APIs (external consumers) | Internal service-to-service | Frontend-driven data fetching |
| Browser clients | High-throughput microservices | Aggregating multiple backends |
| Simple CRUD | Streaming (real-time, uploads) | Flexible queries (mobile clients) |
| Human-debuggable | Type safety and code generation | Reducing over/under-fetching |

### Hybrid Architecture

Most systems use multiple protocols:

```
Browser → REST/GraphQL → API Gateway → gRPC → Internal Services
Mobile  → REST          → API Gateway → gRPC → Internal Services
Service → gRPC                        → gRPC → Internal Services
```

---

## Other Binary Protocols

| Protocol | Use Case |
|----------|----------|
| Apache Thrift | Facebook-origin RPC (similar to gRPC, less popular) |
| Apache Avro | Schema evolution, Kafka serialisation |
| MessagePack | JSON-compatible binary format (schemaless) |
| FlatBuffers | Zero-copy access (games, low-latency) |
| Cap'n Proto | Zero-copy, no encode/decode step |
| CBOR | Binary JSON for IoT/constrained devices |

---

## Key Takeaways

- gRPC + Protobuf is the standard for internal service-to-service communication. REST for external APIs.
- Protobuf field numbers are the wire identity — never reuse or change them.
- gRPC supports four patterns: unary, server streaming, client streaming, bidirectional.
- Always set deadlines on gRPC calls. They propagate through the call chain automatically.
- gRPC status codes are not HTTP status codes — learn the gRPC-specific codes.
- The hybrid pattern (REST at the edge, gRPC internally) gives you the best of both worlds.
