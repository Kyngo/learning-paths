---
title: "Technical Writing"
weight: 11
---

# Technical Writing

The ability to explain complex technical ideas clearly — in docs, READMEs, commit messages, design proposals, and decision records — is what separates a productive engineer from a 10x force multiplier. Writing is thinking made visible. If you can't write it down clearly, you probably don't understand it clearly yet.

## Why Technical Writing Matters

| Situation | Poor writing cost | Good writing benefit |
|-----------|------------------|---------------------|
| README for a new service | Hours of Slack questions, onboarding friction | New devs self-serve in minutes |
| Commit message | Archaeology needed during debugging | Instant context when `git blame` surfaces it |
| ADR (Architecture Decision Record) | Decisions revisited endlessly, context lost | Team understands *why* even years later |
| RFC (Request for Comments) | Design disagreements in code review (too late) | Alignment before implementation begins |
| Incident postmortem | Same incidents recur | Organisational learning, permanent improvement |

## Audience Awareness

The single most important skill in technical writing. Before writing anything, ask:

| Question | Why it matters |
|----------|---------------|
| Who will read this? | Determines vocabulary, depth, and assumed knowledge |
| What do they already know? | Avoids patronising experts or losing beginners |
| What do they need to do after reading? | Shapes structure — reference vs tutorial vs explanation |
| When will they read it? | During an incident? Onboarding? Architecture review? |
| What state are they in? | Stressed (incident)? Curious (learning)? Skeptical (proposal)? |

### Audience Spectrum

| Audience | Assume they know | Write like this |
|----------|-----------------|-----------------|
| Future you (6 months) | The language, the tools | Explain the *why* and the *context* |
| Team member | The domain, the stack | Explain the specific decision/change |
| New hire | The language (maybe) | Explain the system, the domain, the conventions |
| External developer (API docs) | Their own stack | Explain your interface, not your internals |
| Non-technical stakeholder | Nothing technical | Explain the impact, not the implementation |

## Writing READMEs

A README is the front door of your project. If it doesn't answer "what is this, how do I run it, and where do I go next?" within 60 seconds, it has failed.

### README Structure

```markdown
# Project Name

One-sentence description of what this does and why.

## Quick Start

The fastest path from clone to running code.

## Prerequisites

What you need installed before anything works.

## Installation / Setup

Step-by-step. Copy-paste-able commands.

## Usage

How to use the thing once it's running. Common commands.

## Configuration

Environment variables, config files, feature flags.

## Architecture (optional)

Brief overview for orientation. Link to deeper docs.

## Contributing

How to submit changes. Branch naming, PR process.

## Troubleshooting (optional)

Common problems and their solutions.
```

### README Principles

| Principle | Application |
|-----------|-------------|
| Start with the user's goal | "How do I run this?" comes before "How is this architected?" |
| Make commands copy-pasteable | Use code blocks, include the full command |
| Keep it current | Outdated READMEs are worse than none (they actively mislead) |
| Link, don't duplicate | Reference deeper docs rather than inlining everything |
| Test your own README | Clone fresh, follow your instructions. Do they work? |

## Writing Commit Messages

Every commit message is a letter to future developers (including yourself). It costs seconds to write well, and saves hours during debugging, code archaeology, and `git bisect`.

### Anatomy of a Good Commit Message

```
<type>(<scope>): <short summary in imperative mood>

<body: explain WHY, not WHAT — the diff shows what>

<footer: ticket reference, breaking changes, co-authors>
```

### Commit Message Rules

| Rule | Good | Bad |
|------|------|-----|
| Use imperative mood | "Add retry logic to upload" | "Added retry logic" |
| Explain why, not what | "Retry prevents transient 503s from failing the pipeline" | "Add try/except block" |
| Keep subject ≤ 72 chars | "fix(auth): resolve race in token refresh" | "Fixed the bug where sometimes the authentication token would expire..." |
| Reference tickets | "TAI-86" in footer | No traceability to requirements |
| One logical change | Single concern per commit | Mixed refactor + feature + fix |

### When the Body Matters

Not every commit needs a body. Use one when:

- The *why* isn't obvious from the diff
- You chose between alternatives (briefly state why)
- The change has non-obvious consequences
- You're working around a known issue (link it)

## Architecture Decision Records (ADRs)

An ADR captures a significant technical decision with its context and consequences. ADRs prevent the "why did we do it this way?" question from recurring indefinitely.

### When to Write an ADR

Write an ADR when:
- Choosing between competing technologies or approaches
- Making a decision that would be expensive to reverse
- Deviating from an established pattern
- The reasoning might not be obvious in 6 months

### ADR Structure

| Section | Purpose | Length |
|---------|---------|--------|
| **Title** | Clear statement of the decision | One line |
| **Status** | Proposed / Accepted / Superseded / Deprecated | One word |
| **Context** | Why the decision is needed, forces at play | 2–5 paragraphs |
| **Decision** | What was decided | 1–3 paragraphs |
| **Consequences** | What becomes easier and harder | Balanced list |

### ADR Writing Tips

- Write the Context section for someone who joins the team next year
- Include the alternatives you considered and why you rejected them
- Be honest about trade-offs — every decision has downsides
- Keep them short — if it's over 2 pages, you're writing a design doc
- Number them sequentially. Don't delete or reorder. Supersede instead.

## Requests for Comments (RFCs)

An RFC is a proposal for a significant change that invites structured feedback *before* implementation begins. Unlike ADRs (which record decisions already made), RFCs are pre-decision documents.

### When to Write an RFC

| Write an RFC | Don't need an RFC |
|-------------|-------------------|
| New service or system | Bug fix |
| Major architectural change | Small refactor |
| Cross-team impact | Team-internal tooling |
| Reversing a previous ADR | Routine feature work |
| Introducing a new technology | Using established patterns |

### RFC Structure

```markdown
# RFC: [Title]

**Author:** [name]
**Status:** Draft | In Review | Accepted | Rejected | Withdrawn
**Date:** [date]
**Reviewers:** [names]

## Summary

[1-2 sentences: what you're proposing]

## Motivation

[Why this change is needed. What problem it solves.]

## Proposed Design

[The solution in detail. Diagrams welcome.]

## Alternatives Considered

[What else you evaluated and why you didn't choose it.]

## Risks and Mitigations

[What could go wrong and how you'd handle it.]

## Open Questions

[Things you're unsure about and want input on.]

## Timeline

[Rough estimate of implementation effort.]
```

### RFC Process Tips

- Share early (it's a Draft, not a finished product)
- Actively solicit feedback — don't just post and hope
- Set a review deadline (1-2 weeks is typical)
- Respond to every comment, even if the answer is "won't do"
- Update the document as decisions are made during review

## General Technical Writing Principles

### Clarity Checklist

| Principle | Technique |
|-----------|-----------|
| Use simple words | "use" not "utilise", "start" not "initialise" (unless it's a method name) |
| One idea per sentence | If a sentence has "and" or "but" with two different ideas, split it |
| Active voice | "The service processes the request" not "The request is processed by the service" |
| Concrete examples | Don't just explain — show a before/after, a code snippet, a scenario |
| Front-load the important bit | Lead with the conclusion, then support it |
| Cut ruthlessly | If removing a sentence doesn't lose meaning, remove it |

### The Inverted Pyramid

Borrowed from journalism — put the most important information first:

```
1. The conclusion / answer / decision         ← Most readers stop here
2. The supporting evidence / reasoning        ← Interested readers continue
3. The full context and background            ← Deep-dive readers reach here
```

This respects your reader's time. Engineers scanning docs in a hurry get value from the first paragraph alone.

### Technical vs Conversational Tone

| Document type | Tone | Example |
|--------------|------|---------|
| API docs | Precise, neutral | "Returns a 404 if the resource does not exist." |
| README | Friendly, direct | "Clone the repo and run `make dev` to get started." |
| ADR | Professional, balanced | "We chose PostgreSQL over DynamoDB because..." |
| Postmortem | Blameless, factual | "The deploy at 14:32 introduced a regression in..." |
| RFC | Proposing, open | "This RFC proposes X. Feedback welcome on sections 3 and 5." |

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Wall of text | No one reads it | Use headings, lists, tables, whitespace |
| Assuming context | Reader doesn't know what you know | State the context explicitly |
| Writing for yourself | Others can't follow your mental model | Write for the *next* reader |
| Documenting the obvious | Noise buries the signal | Document the *non-obvious* decisions |
| Never updating | Docs become misleading | Treat docs like code — maintain or delete |
| Perfection paralysis | Nothing gets written | A good-enough doc today beats a perfect doc never |

## Key Takeaways

- Know your audience before you write a single word — it determines everything else
- READMEs should get someone from zero to running in under 5 minutes
- Commit messages are for future debuggers — explain *why*, the diff shows *what*
- ADRs prevent "why did we do this?" from being asked repeatedly. Write them for decisions that cost money to reverse
- RFCs invite feedback before code exists — much cheaper than rework after implementation
- Use the inverted pyramid: conclusion first, details second, background last
- Technical writing is a practice. The more you write, the clearer your thinking becomes
