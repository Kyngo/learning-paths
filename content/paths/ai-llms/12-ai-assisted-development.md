---
title: "AI-Assisted Development"
weight: 12
---

AI coding assistants have become standard tools in software engineering. This chapter covers how to use them effectively, understand their limitations, and integrate them into development workflows without compromising code quality or security.

## Code Generation Tools

### The Landscape

| Tool | Type | Model | Integration |
|------|------|-------|-------------|
| GitHub Copilot | Inline completion + chat | GPT-4o / Claude | VS Code, JetBrains, Neovim |
| Cursor | AI-native IDE | Claude, GPT-4o, custom | Fork of VS Code |
| Claude (Anthropic) | Chat + artifacts + computer use | Claude 4 family | Web, API, IDE extensions |
| ChatGPT | Chat interface | GPT-4o | Web, API, plugins |
| Amazon Q Developer | Inline + chat + agents | Proprietary | VS Code, JetBrains, CLI |
| Codeium / Windsurf | Inline + chat | Proprietary | Multi-IDE |
| Aider | Terminal-based pair programming | Any (OpenAI, Anthropic, local) | CLI, git-integrated |
| Kiro | Spec-driven development | Claude | VS Code |

### How Code Completion Works

```text
Code Completion Pipeline:

1: Context Gathering
   ├── Current file (cursor position)
   ├── Open tabs / related files
   ├── Import statements (dependency hints)
   ├── Recent edits (intent signals)
   └── Project structure (file tree)

2: Prompt Construction
   ├── Prefix (code before cursor)
   ├── Suffix (code after cursor)
   └── Metadata (language, framework, file path)

3: Model Inference
   └── Generate completion tokens

4: Post-Processing
   ├── Filter low-confidence suggestions
   ├── Deduplicate
   ├── Format to match style
   └── Present to user (inline ghost text)
```

### IDE Integration Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| Inline ghost text | Grayed-out suggestion at cursor | Copilot tab-to-accept |
| Chat panel | Conversational coding assistant | Cursor chat, Copilot chat |
| Inline edit | Select code → describe change | Cursor Cmd+K |
| Agent mode | Multi-file autonomous changes | Cursor composer, Copilot agent |
| Terminal assist | Command suggestions in terminal | Amazon Q, GitHub Copilot CLI |

## Effective Prompting for Code

### Principles

| Principle | Bad Example | Good Example |
|-----------|-------------|--------------|
| Be specific | "Make a function" | "Write a Python function that validates email addresses using regex, returning True/False" |
| Provide context | "Fix this bug" | "This function should return sorted results but returns unsorted when the input has duplicates" |
| Specify constraints | "Make it fast" | "Optimize for O(n log n) time complexity, input size up to 1M elements" |
| Include examples | "Parse dates" | "Parse dates like '2024-01-15', 'Jan 15, 2024', '15/01/2024' into ISO format" |
| State the stack | "Build an API" | "Build a REST endpoint using FastAPI with Pydantic validation and SQLAlchemy ORM" |

### Comment-Driven Development

Write comments describing what you want, then let the AI generate the implementation:

```python
# Validate that the email has a valid format, domain exists (MX record),
# and is not from a disposable email provider. Return a ValidationResult
# with is_valid, reason, and suggestions fields.
def validate_email(email: str) -> ValidationResult:
    # AI generates implementation based on the detailed comment
    ...
```

### Prompt Patterns for Code

```text
Pattern: Describe → Constrain → Example

"Write a TypeScript function that [DESCRIPTION].
It should [CONSTRAINTS].
Example: [INPUT] → [OUTPUT]"
```

```text
Pattern: Context → Problem → Ask

"I have a React component that renders a list of users.
Currently it re-renders the entire list when any user updates.
How should I optimize this to only re-render changed items?"
```

```text
Pattern: Role → Task → Format

"As a senior Python developer reviewing this code,
identify potential issues with error handling, type safety, and performance.
Format as a numbered list with severity (high/medium/low)."
```

## Code Review with AI

### What AI Can Review

| Review Aspect | AI Effectiveness | Notes |
|---------------|-----------------|-------|
| Bug detection | Medium-High | Good at spotting null checks, off-by-one, race conditions |
| Security vulnerabilities | Medium | Catches common patterns (SQL injection, XSS) |
| Code style consistency | High | Excellent at enforcing conventions |
| Performance issues | Medium | Identifies obvious N+1 queries, unnecessary allocations |
| Logic errors | Medium | Better with clear specifications to compare against |
| Architecture decisions | Low-Medium | Lacks full system context |
| Business logic correctness | Low | Doesn't know your domain rules |

### Effective Code Review Prompts

```text
Review this code for:
1. Correctness: Does it handle edge cases (empty input, null, overflow)?
2. Security: Any injection, auth bypass, or data exposure risks?
3. Performance: Any O(n²) where O(n) is possible? Unnecessary allocations?
4. Maintainability: Clear naming? Single responsibility? Testable?

For each issue found, provide:
- Severity (critical/high/medium/low)
- Line reference
- What's wrong
- Suggested fix
```

### Limitations of AI Code Review

- Cannot run the code or verify behavior
- Misses issues requiring full system context (distributed state, race conditions across services)
- May flag "issues" that are intentional design decisions
- Cannot verify that tests actually pass
- May miss domain-specific business logic errors

## Test Generation

### AI-Generated Tests

AI excels at generating test boilerplate and common test cases:

```python
# Prompt: "Generate comprehensive tests for this function"

def calculate_shipping(weight: float, distance: int, express: bool = False) -> float:
    """Calculate shipping cost based on weight, distance, and speed."""
    base_rate = 5.0
    weight_rate = 0.5 * weight
    distance_rate = 0.01 * distance
    total = base_rate + weight_rate + distance_rate
    if express:
        total *= 1.5
    return round(total, 2)

# AI-generated tests:
import pytest

class TestCalculateShipping:
    def test_base_rate_only(self):
        assert calculate_shipping(0, 0) == 5.0
    
    def test_weight_contribution(self):
        assert calculate_shipping(10, 0) == 10.0  # 5 + 0.5*10
    
    def test_distance_contribution(self):
        assert calculate_shipping(0, 1000) == 15.0  # 5 + 0.01*1000
    
    def test_combined(self):
        assert calculate_shipping(4, 500) == 12.0  # 5 + 2 + 5
    
    def test_express_multiplier(self):
        assert calculate_shipping(0, 0, express=True) == 7.5  # 5 * 1.5
    
    def test_express_with_weight_and_distance(self):
        result = calculate_shipping(4, 500, express=True)
        assert result == 18.0  # (5 + 2 + 5) * 1.5
    
    def test_negative_weight_raises(self):
        # AI might suggest this — reveals missing validation
        with pytest.raises(ValueError):
            calculate_shipping(-1, 100)
    
    def test_rounding(self):
        result = calculate_shipping(3.33, 777)
        assert result == round(5 + 0.5*3.33 + 0.01*777, 2)
```

### Test Generation Best Practices

| Do | Don't |
|----|-------|
| Use AI for boilerplate and edge case discovery | Blindly accept all generated tests |
| Review generated assertions for correctness | Trust AI-generated expected values without verification |
| Ask AI to identify untested edge cases | Skip reviewing test logic |
| Use AI to generate test data/fixtures | Let AI decide what's important to test |
| Ask for property-based test suggestions | Assume generated tests provide full coverage |

## Documentation Generation

### What AI Does Well

```python
# Give AI a function and ask for documentation:

def merge_intervals(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    if not intervals:
        return []
    sorted_intervals = sorted(intervals, key=lambda x: x[0])
    merged = [sorted_intervals[0]]
    for start, end in sorted_intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], end))
        else:
            merged.append((start, end))
    return merged

# AI-generated documentation:
"""Merge overlapping intervals into non-overlapping intervals.

Given a list of intervals represented as (start, end) tuples,
merge all overlapping intervals and return the result sorted
by start time.

Args:
    intervals: List of (start, end) tuples where start <= end.

Returns:
    List of merged non-overlapping intervals, sorted by start.

Examples:
    >>> merge_intervals([(1, 3), (2, 6), (8, 10)])
    [(1, 6), (8, 10)]
    >>> merge_intervals([(1, 4), (4, 5)])
    [(1, 5)]
    >>> merge_intervals([])
    []

Time Complexity: O(n log n) due to sorting.
Space Complexity: O(n) for the output list.
"""
```

### Documentation Types AI Can Generate

| Type | Quality | Notes |
|------|---------|-------|
| Function docstrings | High | Excellent at inferring purpose from code |
| README files | Medium-High | Needs project context, good at structure |
| API documentation | High | Great at describing endpoints, params, responses |
| Architecture docs | Medium | Needs human guidance on "why" decisions |
| Changelog entries | High | Good at summarizing diffs |
| Inline comments | Medium | Sometimes states the obvious |
| Migration guides | Medium | Needs before/after context |

## Debugging with AI

### Effective Debugging Prompts

```text
Pattern: Error → Context → Question

"I'm getting this error:
[paste full error with stack trace]

This happens when:
[describe the conditions]

The relevant code is:
[paste the function/module]

What's causing this and how do I fix it?"
```

### What AI Helps With in Debugging

| Scenario | AI Effectiveness | Why |
|----------|-----------------|-----|
| Interpreting error messages | High | Vast training on error patterns |
| Suggesting fixes for common errors | High | Pattern matching from millions of examples |
| Explaining unfamiliar code | High | Good at reading and summarizing |
| Identifying logic errors | Medium | Needs clear specification of expected behavior |
| Debugging race conditions | Low | Can't observe runtime state |
| Debugging distributed systems | Low | Lacks system-wide visibility |
| Performance profiling | Low-Medium | Can suggest optimizations but can't measure |

## Refactoring Assistance

### AI-Assisted Refactoring Patterns

| Refactoring | Prompt Approach |
|-------------|----------------|
| Extract function | "Extract the logic in lines X-Y into a well-named function" |
| Simplify conditionals | "Simplify this nested if/else into a cleaner pattern" |
| Remove duplication | "These 3 functions share similar logic — extract the common pattern" |
| Modernize syntax | "Update this to use modern Python 3.12+ features" |
| Add types | "Add comprehensive type annotations to this module" |
| Improve naming | "Suggest better names for these variables and functions" |
| Design pattern | "Refactor this to use the Strategy pattern" |

## Limitations and Risks

### Common Failure Modes

| Failure Mode | Description | Mitigation |
|-------------|-------------|------------|
| Hallucinated APIs | Invents functions/methods that don't exist | Always verify against docs |
| Outdated patterns | Uses deprecated APIs or old syntax | Specify version in prompt |
| Plausible but wrong | Code looks correct but has subtle bugs | Test thoroughly, review carefully |
| Security vulnerabilities | Generates code with injection, hardcoded secrets | Security review, SAST tools |
| License contamination | May reproduce copyrighted code verbatim | Use tools with license filtering |
| Over-engineering | Adds unnecessary complexity | Specify simplicity constraints |
| Inconsistent style | Doesn't match project conventions | Provide style examples, use linters |

### Security Risks

```text
Security Concerns with AI-Generated Code:

1: Injection vulnerabilities
   AI may generate string concatenation instead of parameterized queries

2: Hardcoded credentials
   Training data includes examples with real-looking secrets

3: Insecure defaults
   May use HTTP instead of HTTPS, weak crypto algorithms

4: Missing input validation
   Generated code often skips boundary checks

5: Dependency confusion
   May suggest packages that don't exist (typosquatting risk)

6: Information leakage
   Generated error handlers may expose internal details

Mitigation:
- Run SAST/DAST on all generated code
- Never trust AI-suggested package names without verification
- Review security-sensitive code manually regardless of source
- Use AI-aware security scanning tools
```

### Knowledge Cutoff Issues

```text
Problem: Models have training data cutoffs

Symptoms:
- Suggests deprecated APIs (e.g., old React lifecycle methods)
- Doesn't know about new language features
- References old library versions
- Misses breaking changes in frameworks

Mitigation:
- Always specify versions: "Using React 19, Next.js 15"
- Provide relevant documentation snippets in context
- Use tools with documentation retrieval (Cursor @docs, RAG)
- Verify suggestions against current official docs
```

## AI in CI/CD Pipelines

### Automated Code Review in CI

```text
CI Pipeline with AI Review:

┌─────────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│ Push / MR   │ ──→ │  Build   │ ──→ │   Test    │ ──→ │  Deploy  │
└─────────────┘     └──────────┘     └───────────┘     └──────────┘
       │                                    │
       ↓                                    ↓
┌─────────────┐                    ┌───────────────┐
│ AI Review   │                    │ AI Test Gen   │
│ (non-block) │                    │ (suggestions) │
└─────────────┘                    └───────────────┘
```

### AI-Powered CI Tasks

| Task | How | Blocking? |
|------|-----|-----------|
| PR description generation | Summarize diff into description | No |
| Code review comments | Analyze diff for issues | No (advisory) |
| Test suggestions | Identify untested paths in diff | No |
| Commit message validation | Check conventional commit format | Yes |
| Documentation updates | Flag when docs need updating | No |
| Security scanning | AI-enhanced SAST | Yes (for critical) |
| Changelog generation | Summarize changes for release notes | No |

### Considerations for AI in CI

- **Cost**: Every PR triggers API calls — budget accordingly
- **Latency**: AI review adds minutes to pipeline — run in parallel
- **Noise**: Too many low-quality comments cause alert fatigue
- **Determinism**: Same code may get different reviews — accept this
- **Privacy**: Code sent to external APIs — check compliance requirements

## Measuring Productivity Impact

### Metrics That Matter

| Metric | How to Measure | Caveats |
|--------|---------------|---------|
| Time to first commit | Track from ticket start to first push | Doesn't measure quality |
| Code review cycles | Number of review rounds before merge | AI may reduce or increase |
| Bug escape rate | Bugs found in production per release | Lagging indicator |
| Developer satisfaction | Survey (1-10 scale) | Subjective but important |
| Lines of code | Diff size per task | More code ≠ better |
| Test coverage delta | Coverage change per PR | AI can inflate with low-value tests |
| Time in flow state | Self-reported focus time | AI interruptions can break flow |

### What Research Shows

```text
Findings from studies (GitHub, Microsoft, academic):

- 30-55% faster task completion for well-defined tasks
- Minimal improvement for complex architectural work
- Junior developers benefit more than seniors
- Quality impact is mixed (faster but sometimes buggier)
- Biggest gains: boilerplate, tests, documentation
- Smallest gains: novel algorithms, system design, debugging complex issues
```

## When AI Helps vs When It Hurts

### AI Excels At

| Task | Why |
|------|-----|
| Boilerplate code | Repetitive patterns well-represented in training data |
| Unit test generation | Mechanical transformation of specs to assertions |
| Code translation | Language-to-language mapping is well-defined |
| Documentation | Summarizing code intent is a strength |
| Regex and SQL | Complex syntax that humans struggle to write correctly |
| API integration | Common patterns with well-documented libraries |
| Explaining code | Reading comprehension is a core LLM capability |
| Prototyping | Speed matters more than perfection |

### AI Struggles With

| Task | Why |
|------|-----|
| Novel algorithms | Can't reason about truly new approaches |
| System architecture | Lacks full context of constraints and tradeoffs |
| Performance optimization | Can't profile or measure actual performance |
| Debugging production issues | Can't observe runtime state or logs |
| Security-critical code | May introduce subtle vulnerabilities |
| Domain-specific logic | Doesn't know your business rules |
| Large-scale refactoring | Loses coherence across many files |
| Concurrency bugs | Can't reason about timing and state |

### Decision Framework

```text
Should I use AI for this task?

Is the task well-defined with clear inputs/outputs?
  YES → AI likely helps
  NO  → AI may produce plausible but wrong results

Is correctness critical (security, financial, safety)?
  YES → Use AI for drafting, human verifies everything
  NO  → AI can do more autonomously

Does the task require full system context?
  YES → Provide extensive context or do it manually
  NO  → AI works well with local context

Is this a common pattern or novel problem?
  COMMON → AI excels (it's seen thousands of examples)
  NOVEL  → AI may hallucinate; use for brainstorming only

Am I faster doing this manually?
  YES → Skip AI (context switching has a cost)
  NO  → Use AI and review the output
```

## Architecture and Design with AI

### Using AI as a Design Partner

AI is effective as a sounding board for architectural decisions — a rubber duck that talks back. The key is providing enough context about your constraints so the AI can reason about trade-offs rather than give generic advice.

```text
Effective design prompt structure:

"I'm designing [system/component] that needs to:
- [Requirement 1 with specific numbers]
- [Requirement 2]

Constraints:
- [Budget, team size, timeline]
- [Existing systems it must integrate with]
- [Non-negotiable technology choices]

I'm considering [Option A] vs [Option B].
What are the trade-offs I might be missing?"
```

### Generating Architecture Diagrams

AI can produce Mermaid diagrams from natural language descriptions, which is faster than drawing from scratch:

```mermaid
graph TB
    Client[Client App] --> Gateway[API Gateway]
    Gateway --> AuthService[Auth Service]
    Gateway --> OrderService[Order Service]
    Gateway --> SearchService[Search Service]
    OrderService --> OrderDB[(PostgreSQL)]
    OrderService --> Queue[SQS Queue]
    Queue --> NotificationService[Notification<br/>Service]
    SearchService --> ElasticSearch[(Elasticsearch)]
    OrderService --> Cache[Redis Cache]
```

Prompt that generated this:

```text
"Generate a Mermaid diagram for an e-commerce backend with:
- API gateway routing to auth, orders, and search services
- Orders use PostgreSQL and publish to SQS for notifications
- Search uses Elasticsearch
- Orders service has a Redis cache layer"
```

### Evaluating Trade-offs

| Design Question | What AI Can Help With | What AI Cannot Help With |
|----------------|----------------------|--------------------------|
| "SQL vs NoSQL?" | General trade-offs, access pattern analysis | Your actual query patterns, data volume projections |
| "Monolith vs microservices?" | Complexity analysis, team size considerations | Your team's skill level, org politics |
| "Sync vs async?" | Latency/throughput trade-offs, failure modes | Your SLA requirements, actual traffic patterns |
| "Build vs buy?" | Feature comparison, integration complexity | Vendor reliability, contract terms, budget |

### Limitations in Design Discussions

```text
What AI doesn't know about your system:

- Actual traffic patterns and growth projections
- Team expertise and hiring plans
- Budget constraints and cost sensitivity
- Existing technical debt and migration timelines
- Organizational politics and decision-making culture
- Compliance requirements specific to your industry
- Performance characteristics of your current infrastructure
- Undocumented dependencies between services

Rule: AI proposes options and trade-offs.
      Humans make decisions based on constraints AI can't see.
```

### Practical Design Session Pattern

```python
# Step 1: Describe the problem with constraints
prompt_1 = """
We need a real-time notification system for 50K concurrent users.
Constraints: AWS only, budget < $2K/month, team of 3 backend devs,
must integrate with existing PostgreSQL database.
Current stack: Python/FastAPI, deployed on ECS Fargate.
What architecture options should we consider?
"""

# Step 2: Drill into the recommended option
prompt_2 = """
You suggested WebSockets via API Gateway + Lambda.
How does this handle:
- Connection drops and reconnection?
- Message ordering guarantees?
- Scaling beyond 50K connections?
Show me the failure modes.
"""

# Step 3: Ask for the diagram
prompt_3 = """
Generate a Mermaid sequence diagram showing the happy path
and one failure/reconnection scenario for this WebSocket approach.
"""
```

## Migration and Modernization

### Understanding Legacy Code with AI

AI excels at reading and explaining code you didn't write — especially when documentation is missing or outdated:

```python
# Prompt: "Explain what this legacy function does, what its side effects are,
# and what would break if I changed the sort order"

def process_batch(records, config):
    # 200 lines of undocumented legacy code
    # AI can trace the logic and explain:
    # - Data transformations applied
    # - External calls made (DB, APIs)
    # - Implicit assumptions (sort order, null handling)
    # - Side effects (file writes, state mutations)
    ...
```

### Planning Migrations

| Migration Type | How AI Helps | Limitations |
|---------------|-------------|-------------|
| Framework upgrade (e.g., Vue 2 → 3) | Identifies deprecated APIs, suggests replacements | May miss custom plugins or edge cases |
| Language port (Python → TypeScript) | Translates idioms, maps type systems | Loses runtime behavior nuances |
| Database migration (SQL → NoSQL) | Generates schema mappings, query rewrites | Can't assess data volume impact |
| API versioning (REST → GraphQL) | Generates schema from endpoints | Doesn't know client usage patterns |
| Monolith decomposition | Identifies bounded contexts from code | Can't see runtime coupling |

### Generating Migration Scripts

```typescript
// Prompt: "Generate a migration script that converts our Express.js
// middleware to Fastify plugins. Here's an example middleware:"

// Before (Express):
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token' });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// AI-generated (Fastify):
import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';
import jwt from 'jsonwebtoken';

const authPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.decorateRequest('user', null);

  fastify.addHook('onRequest', async (request, reply) => {
    const token = request.headers.authorization?.split(' ')[1];
    if (!token) {
      reply.code(401).send({ error: 'No token' });
      return;
    }
    try {
      request.user = jwt.verify(token, process.env.JWT_SECRET!);
    } catch {
      reply.code(401).send({ error: 'Invalid token' });
    }
  });
};

export default fp(authPlugin, { name: 'auth' });
```

### Identifying Breaking Changes

```text
Migration checklist prompt:

"I'm upgrading from [library v2] to [library v4].
Here are my imports and usages: [paste grep results]

For each usage, tell me:
1. Is this API still available in v4?
2. If changed, what's the new equivalent?
3. Are there behavioral changes (same API, different behavior)?
4. Priority: breaking (won't compile) vs subtle (compiles but behaves differently)"
```

### Context Window Limitations for Large Codebases

```text
Problem: Legacy codebases exceed AI context windows

A typical enterprise monolith:
- 500K+ lines of code
- 2000+ files
- Context window: 100K-200K tokens ≈ 50-100 files max

Strategies:
┌─────────────────────────────────────────────────┐
│ 1. Work module by module                        │
│    - Migrate one bounded context at a time      │
│    - Provide AI with just that module + its     │
│      interfaces to the rest of the system       │
│                                                 │
│ 2. Use dependency graphs as context             │
│    - Feed AI the import/call graph, not all code│
│    - Ask it to identify migration order         │
│                                                 │
│ 3. Summarize then detail                        │
│    - First pass: AI summarizes each module      │
│    - Second pass: detailed migration per module │
│                                                 │
│ 4. Focus on interfaces                          │
│    - Migrate public APIs first                  │
│    - Internal implementation can follow         │
└─────────────────────────────────────────────────┘
```

## AI-Powered Code Search and Understanding

### Navigating Unfamiliar Codebases

When joining a new project or investigating an unfamiliar service, AI can accelerate understanding dramatically:

```text
Onboarding prompt sequence:

1. "Here's the project structure [paste tree output].
    What kind of application is this? What framework and patterns does it use?"

2. "Here's the main entry point [paste file].
    Trace the request lifecycle from incoming HTTP to response."

3. "Here are the model files [paste].
    What's the domain model? Draw a Mermaid ER diagram."

4. "Here's the test suite structure [paste].
    What's well-tested vs undertested?"
```

### Explaining Complex Code

```python
# Feed AI a complex function and ask targeted questions:

# "What does this function do? What are the edge cases?
#  Why might the original author have used a generator here
#  instead of returning a list?"

def resolve_dependencies(graph: dict[str, set[str]]) -> Iterator[set[str]]:
    """Kahn's algorithm for topological sort, yielding parallel batches."""
    in_degree = {n: 0 for n in graph}
    for node in graph:
        for dep in graph[node]:
            in_degree[dep] = in_degree.get(dep, 0) + 1

    queue = {n for n, d in in_degree.items() if d == 0}
    while queue:
        yield queue
        next_queue = set()
        for node in queue:
            for dep in graph.get(node, []):
                in_degree[dep] -= 1
                if in_degree[dep] == 0:
                    next_queue.add(dep)
        queue = next_queue
```

### Tracing Data Flows

```mermaid
sequenceDiagram
    participant Client
    participant Controller
    participant Service
    participant Repository
    participant DB

    Client->>Controller: POST /orders
    Controller->>Controller: Validate DTO
    Controller->>Service: createOrder(dto)
    Service->>Service: Apply business rules
    Service->>Repository: save(entity)
    Repository->>DB: INSERT
    DB-->>Repository: id
    Repository-->>Service: entity
    Service->>Service: Emit OrderCreated event
    Service-->>Controller: OrderResponse
    Controller-->>Client: 201 Created
```

Prompt: "Trace the data flow for order creation from HTTP request to database. Show me where validation, transformation, and side effects happen."

### Finding Usage Patterns

| Question | Prompt Pattern |
|----------|---------------|
| "How is this function used?" | "Here are all call sites for `processPayment` [paste grep results]. Summarize the usage patterns — are there common pre/post conditions?" |
| "What's the error handling pattern?" | "Here are 5 error handlers from this codebase [paste]. What's the consistent pattern? Where does it deviate?" |
| "How do others solve this?" | "Here are 3 similar services in our codebase [paste interfaces]. What's the common pattern I should follow for a new one?" |

### Generating Onboarding Documentation

```text
Prompt: "Based on these files, generate a developer onboarding doc covering:
1. How to run the project locally
2. Architecture overview (with Mermaid diagram)
3. Key abstractions and where to find them
4. Common development tasks (add endpoint, add migration, run tests)
5. Gotchas and non-obvious conventions"

Input: package.json, docker-compose.yml, src/ tree, 2-3 key files
Output: A README that would save a new developer 2 days of exploration
```

## Pair Programming with AI

### Collaboration Patterns

| Pattern | When to Use | How It Works |
|---------|-------------|--------------|
| AI leads, you review | Boilerplate, repetitive code, well-defined tasks | Describe what you need, review and refine output |
| You lead, AI assists | Complex logic, architectural decisions | Write the structure, ask AI to fill in details |
| Ping-pong | Iterative refinement | You write a draft, AI improves it, you adjust, repeat |
| AI as reviewer | After you've written code | Ask AI to critique your implementation |
| AI as explainer | Learning new APIs or patterns | Write code, ask AI to explain what it does and why |

### When to Lead vs Follow

```mermaid
flowchart TD
    Start[New Task] --> Q1{Do I understand<br/>the problem well?}
    Q1 -->|Yes| Q2{Is it mostly<br/>boilerplate?}
    Q1 -->|No| Lead[You lead:<br/>explore and ask<br/>AI questions]
    Q2 -->|Yes| Follow[AI leads:<br/>describe what you<br/>need, review output]
    Q2 -->|No| Q3{Is correctness<br/>critical?}
    Q3 -->|Yes| Lead2[You lead:<br/>write core logic,<br/>AI fills details]
    Q3 -->|No| PingPong[Ping-pong:<br/>iterate together]
```

### The "AI as Junior Dev" Mental Model

Treat AI-generated code the way you'd treat a pull request from a capable but inexperienced developer:

| Junior Dev Behavior | AI Equivalent | Your Response |
|--------------------|---------------|---------------|
| Writes working code that misses edge cases | Generates happy-path implementation | Add error handling, boundary checks |
| Follows patterns they've seen without understanding why | Applies common patterns regardless of fit | Verify the pattern suits your context |
| Doesn't know your team's conventions | Ignores project-specific style | Enforce your linting and conventions |
| Asks good questions when stuck | N/A — AI doesn't know it's stuck | You must identify when output is wrong |
| Learns from code review feedback | Responds to follow-up prompts | Iterate with specific corrections |

### Iterative Refinement

```python
# Round 1: Initial generation
# Prompt: "Write a rate limiter using sliding window"
# AI produces a basic implementation

# Round 2: Add constraints
# "This needs to work in a distributed system with Redis.
#  Multiple instances share the same rate limit per user."

# Round 3: Handle edge cases
# "What happens during Redis failover? I need a fallback
#  that's permissive (allow traffic) rather than blocking."

# Round 4: Production hardening
# "Add structured logging, metrics emission, and make the
#  window size and limit configurable per endpoint."

# Each round builds on the previous — don't start over.
# Keep the conversation context so AI remembers decisions.
```

### Avoiding Over-Reliance

```text
Warning signs you're over-relying on AI:

- You accept code you don't fully understand
- You can't explain WHY the code works, only THAT it works
- You've stopped reading documentation directly
- You paste errors into AI without trying to understand them first
- Your debugging process is "ask AI" before "read the error message"
- You feel less confident coding without AI assistance

Healthy habits:
- Understand every line before committing
- Periodically code without AI to maintain skills
- Use AI to learn, not just to produce
- Read the docs AI references — don't just trust the summary
- If AI gives you a pattern you don't recognize, learn it properly
```

### Maintaining Code Ownership

You own every line that gets committed, regardless of who (or what) wrote it:

- **Review AI output with the same rigor as any PR** — you're the author now
- **Be able to explain and defend every decision** in code review
- **Don't commit code you can't debug** — if it breaks at 3am, you need to fix it
- **Refactor AI output to match your style** — consistency matters more than speed

## Security Considerations in AI-Generated Code

### Common Vulnerabilities in Generated Code

| Vulnerability | How AI Introduces It | Example |
|--------------|---------------------|---------|
| SQL Injection | String concatenation instead of parameterized queries | `f"SELECT * FROM users WHERE id = {user_id}"` |
| Hardcoded secrets | Training data includes example credentials | `api_key = "sk-1234567890abcdef"` |
| Insecure defaults | Uses HTTP, weak algorithms, permissive CORS | `cors(origin: '*')`, `algorithm='md5'` |
| Path traversal | No input sanitization on file paths | `open(f"/uploads/{filename}")` |
| Missing auth checks | Generates endpoint without middleware | Route handler with no authentication |
| Verbose error messages | Exposes stack traces and internals | `catch(e) { res.send(e.stack) }` |
| Insecure deserialization | Uses `pickle`, `eval`, or `yaml.load` | `data = pickle.loads(request.body)` |

### Dangerous vs Safe Patterns

```python
# DANGEROUS — AI often generates this:
query = f"SELECT * FROM users WHERE email = '{email}'"
cursor.execute(query)

# SAFE — what you should always use:
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

```typescript
// DANGEROUS — AI often generates this:
const token = "ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
const response = await fetch(url, {
  headers: { Authorization: `Bearer ${token}` }
});

// SAFE — environment variables:
const token = process.env.GITHUB_TOKEN;
if (!token) throw new Error('GITHUB_TOKEN not configured');
const response = await fetch(url, {
  headers: { Authorization: `Bearer ${token}` }
});
```

### Security Review Checklist for AI-Generated Code

```text
Before committing AI-generated code, verify:

□ Authentication & Authorization
  - Are all endpoints protected appropriately?
  - Is there role-based access where needed?

□ Input Validation
  - Are all user inputs validated and sanitized?
  - Are file uploads restricted by type and size?
  - Are path parameters checked for traversal?

□ Secrets Management
  - No hardcoded credentials, API keys, or tokens?
  - Secrets loaded from environment or vault?

□ Query Safety
  - All database queries parameterized?
  - No string interpolation in queries?

□ Error Handling
  - No stack traces or internal details in responses?
  - Errors logged server-side, generic message to client?

□ Dependencies
  - All suggested packages verified to exist?
  - Package names checked for typosquatting?
  - Versions pinned?

□ Cryptography
  - Using current algorithms (bcrypt/argon2, AES-256, SHA-256+)?
  - No MD5 or SHA-1 for security purposes?
  - No custom crypto implementations?
```

### Supply Chain Risks: Hallucinated Packages

AI models sometimes suggest packages that don't exist. Attackers exploit this by publishing malicious packages with names that AI commonly hallucinates:

```text
Real attack vector:

1. Researchers find AI frequently suggests "python-jwt" (doesn't exist)
2. Attacker publishes malicious "python-jwt" on PyPI
3. Developer asks AI for JWT code, gets: pip install python-jwt
4. Developer installs it without checking → compromised

Defense:
- ALWAYS verify package exists on the official registry
- Check download counts, maintainer, and publish date
- Compare against known legitimate packages:
  PyJWT (not python-jwt), requests (not python-requests)
- Use lockfiles and dependency scanning in CI
```

| Legitimate Package | Common AI Hallucination | Risk |
|-------------------|------------------------|------|
| `PyJWT` | `python-jwt`, `jwt` | Typosquatting |
| `python-dateutil` | `dateutil` | Name confusion |
| `beautifulsoup4` | `beautifulsoup`, `bs4` | Version confusion |
| `Pillow` | `PIL`, `python-pillow` | Legacy name |
| `@tanstack/react-query` | `react-query` | Deprecated name |

### OWASP Considerations

Map AI-generated code risks to the OWASP Top 10:

| OWASP Category | AI Risk | Mitigation |
|---------------|---------|------------|
| A01: Broken Access Control | AI skips auth middleware on new endpoints | Template all routes with auth by default |
| A02: Cryptographic Failures | AI uses weak/outdated algorithms | Enforce algorithm allowlist in code review |
| A03: Injection | AI concatenates strings in queries | SAST rules that block string interpolation in queries |
| A05: Security Misconfiguration | AI uses permissive defaults (CORS *, debug=true) | Environment-specific config, never trust defaults |
| A06: Vulnerable Components | AI suggests outdated or hallucinated packages | Dependency scanning, version pinning, registry verification |
| A07: Auth Failures | AI implements custom auth instead of using established libraries | Mandate use of proven auth libraries (Passport, NextAuth, etc.) |
| A09: Logging Failures | AI either logs too much (secrets) or too little | Structured logging template with PII redaction |

---

## Key Takeaways

1. **AI coding tools are amplifiers, not replacements** — they accelerate well-defined tasks (boilerplate, tests, docs) but struggle with novel problems, system design, and domain-specific logic that requires context they don't have.

2. **Effective prompting is the highest-leverage skill** — being specific about constraints, providing examples, stating the tech stack, and giving context dramatically improves output quality compared to vague requests.

3. **Always verify AI-generated code** — hallucinated APIs, subtle logic errors, security vulnerabilities, and outdated patterns are common; treat AI output as a draft that requires human review and testing.

4. **AI code review complements but doesn't replace human review** — AI catches mechanical issues (style, common bugs, missing null checks) but misses architectural concerns, business logic errors, and system-level implications.

5. **Security requires extra vigilance with AI code** — generated code may include injection vulnerabilities, insecure defaults, or suggest non-existent packages; run SAST tools and manually review security-sensitive paths.

6. **Measure impact honestly** — faster completion doesn't always mean better outcomes; track bug escape rate and code review cycles alongside velocity metrics to get the full picture.

7. **Know when to skip AI** — for tasks requiring deep system context, novel algorithmic thinking, or where the cost of context-switching exceeds the benefit, working without AI assistance is often faster and produces better results.

8. **AI is a design sounding board, not a decision maker** — use it to explore trade-offs, generate diagrams, and challenge your assumptions, but architectural decisions must account for constraints (budget, team, timeline) that AI cannot see.

9. **Migrations benefit from AI but require chunking** — AI can translate code, identify breaking changes, and generate migration scripts, but large codebases exceed context windows; work module by module with clear interface boundaries.

10. **Treat AI like a junior pair programmer** — it produces working code that needs your review for edge cases, conventions, and context-specific correctness; you own every line that gets committed regardless of who generated it.

11. **Verify every dependency AI suggests** — hallucinated package names are a real attack vector; always confirm packages exist on official registries, check download counts, and pin versions before installing.
