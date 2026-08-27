---
title: "Contract Testing"
weight: 9
---

# Contract Testing

In a microservice architecture, services communicate through APIs. Contract testing verifies that these APIs remain compatible as services evolve independently — without requiring all services to be running simultaneously.

---

## The Problem

Integration tests between microservices are:

- **Slow** — requires running multiple services, databases, message brokers
- **Flaky** — network issues, environment differences, timing
- **Expensive** — full environment per test run
- **Late** — failures discovered at deploy time, not development time

Contract testing catches incompatibilities at build time, in isolation.

---

## Consumer-Driven Contract Testing

The **consumer** (API caller) defines what it expects. The **provider** (API implementation) verifies it can satisfy those expectations.

```
Consumer: "I call GET /users/1 and expect { id: 1, name: string, email: string }"
           ↓ (writes contract / pact file)
Provider: "Can I serve this request with this response shape?"
           ↓ (verifies against its real implementation)
Result:   ✓ Compatible  or  ✗ Breaking change detected
```

### Why Consumer-Driven?

The consumer knows what it actually uses. If the provider adds a new field, that's fine. If the provider removes a field the consumer depends on, that's caught.

---

## Pact

The most popular contract testing framework. Supports HTTP and message-based interactions.

### Consumer Side (Write the Contract)

```python
# Python example using pact-python
from pact import Consumer, Provider

pact = Consumer('OrderService').has_pact_with(Provider('UserService'))

(pact
    .given('a user with ID 1 exists')
    .upon_receiving('a request for user 1')
    .with_request('GET', '/users/1')
    .will_respond_with(200, body={
        'id': 1,
        'name': Like('Alice'),         # type matching — any string
        'email': Like('a@example.com'), # type matching
    }))

with pact:
    # Run your consumer code against the Pact mock server
    user = user_client.get_user(1)
    assert user.name == 'Alice'
```

This generates a **pact file** (JSON) that describes the interaction.

### Provider Side (Verify the Contract)

```python
# Provider verifies it can satisfy all consumer pacts
from pact import Verifier

verifier = Verifier(provider='UserService', provider_base_url='http://localhost:8080')

verifier.verify_pacts(
    './pacts/orderservice-userservice.json',
    provider_states_setup_url='http://localhost:8080/_pact/setup'
)
```

### Provider States

The provider needs to set up the right data for each test:

```python
# Provider state endpoint
@app.route('/_pact/setup', methods=['POST'])
def provider_states():
    state = request.json['state']
    if state == 'a user with ID 1 exists':
        db.create_user(id=1, name='Alice', email='alice@example.com')
    elif state == 'no users exist':
        db.clear_users()
    return '', 200
```

---

## The Pact Workflow

```
1. Consumer writes tests → generates pact file (JSON)
2. Pact file is published to Pact Broker (shared registry)
3. Provider CI fetches pacts → verifies against real API
4. Results published back to Pact Broker
5. Pact Broker tracks compatibility matrix

Consumer CI:
  test → publish pact → check "can I deploy?" → deploy

Provider CI:
  fetch pacts → verify → publish results → check "can I deploy?" → deploy
```

### PactFlow / Pact Broker

The central registry that tracks:

- All consumer-provider relationships
- Current contract versions
- Verification results
- Deployment environment status
- **"Can I Deploy?"** — answers whether deploying a version would break any consumer

```bash
# Before deploying, check compatibility
pact-broker can-i-deploy --pacticipant UserService --version 1.2.3 --to production
# Yes / No with reasons
```

---

## Message-Based Contracts

Pact also supports asynchronous messaging (Kafka, SQS, RabbitMQ):

```python
# Consumer expects a message with this shape
(pact
    .given('an order is placed')
    .expects_to_receive('an order event')
    .with_content({
        'orderId': Like(123),
        'userId': Like(1),
        'total': Like(99.99),
        'status': Term('placed', regex=r'placed|confirmed|shipped')
    }))
```

The provider verifies it can produce messages matching this contract.

---

## Matching Rules

Pact provides flexible matching beyond exact values:

| Matcher | What It Checks | Example |
|---------|---------------|---------|
| Exact | Literal value | `'Alice'` |
| `Like(example)` | Same type as example | `Like(42)` matches any integer |
| `EachLike(example)` | Array where each element matches | `EachLike({'id': Like(1)})` |
| `Term(example, regex)` | Matches regex pattern | `Term('2024-01-15', r'\d{4}-\d{2}-\d{2}')` |
| `Format.iso_datetime` | ISO datetime | Any valid datetime string |

---

## Contract Testing vs Other Testing

| | Contract Testing | Integration Testing | E2E Testing |
|-|-----------------|-------------------|-------------|
| Scope | One consumer-provider pair | Multiple services | Full system |
| Speed | Seconds | Minutes | Minutes-hours |
| Dependencies | None (isolated) | Real services | Everything |
| Flakiness | Very low | Medium | High |
| Catches | API incompatibilities | Integration bugs | User-facing bugs |
| Cost | Low | Medium | High |
| When to run | Every commit | Daily / PR | Pre-release |

---

## Key Takeaways

- Contract tests verify API compatibility between services without running them together.
- **Consumer-driven:** the consumer defines expectations, the provider verifies it can satisfy them.
- Pact is the standard tool — supports HTTP and message-based interactions, with a broker for coordination.
- The "Can I Deploy?" check prevents breaking deployments before they reach production.
- Contract testing is not a replacement for integration or E2E testing — it catches API shape mismatches, not logic bugs.
- Start with your most critical service-to-service boundary. Expand to other integrations once the workflow is established.
