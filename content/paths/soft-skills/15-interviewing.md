---
title: "Interviewing"
weight: 15
---

# Interviewing

Interviewing is a skill engineers use throughout their careers — on both sides of the table. As a candidate, structured preparation and storytelling separate "great engineer who bombed the interview" from "great engineer who got the offer." As an interviewer, designing fair, signal-rich questions and evaluating without bias is a craft that most people never formally learn.

## Being Interviewed

### The STAR Method

Behavioural questions ("Tell me about a time when...") are best answered with the STAR framework:

| Component | Purpose | Duration | Example |
|-----------|---------|----------|---------|
| **Situation** | Set the scene — context the interviewer needs | 15–20% | "Our payment service was processing 10k transactions/hour and we started seeing timeout errors during peak..." |
| **Task** | What was your responsibility specifically? | 10–15% | "As the backend lead, I was responsible for diagnosing the issue and implementing a fix before Black Friday." |
| **Action** | What did YOU do? (not the team) | 50–60% | "I added distributed tracing, identified the N+1 query, implemented batch loading with a circuit breaker..." |
| **Result** | Quantified outcome | 15–20% | "Latency dropped from 800ms to 120ms at P95. Zero timeouts during Black Friday (3x normal traffic)." |

### STAR Anti-Patterns

| Mistake | Problem | Fix |
|---------|---------|-----|
| "We did X" without your contribution | Interviewer can't assess YOUR skill | Use "I" — be specific about your role |
| All situation, no action | Story without substance | Keep Situation brief, spend time on Action |
| No measurable result | Impact is unclear | Quantify: time saved, revenue impact, uptime, team velocity |
| Rehearsed and robotic | Feels inauthentic | Practice the structure, not a script. Know your stories, speak naturally. |
| Only success stories | Seems unreflective | Have 2–3 failure stories with clear learnings |

### Building Your Story Bank

Prepare 8–10 stories that cover these common behavioural dimensions:

| Dimension | Example questions | Story you need |
|-----------|-------------------|---------------|
| **Technical leadership** | "Tell me about a difficult technical decision" | Architecture choice with trade-offs |
| **Conflict resolution** | "How did you handle a disagreement?" | Disagreement with a peer that you resolved constructively |
| **Failure and learning** | "Tell me about a mistake" | A real failure where you grew |
| **Influence without authority** | "How did you convince others?" | Persuading stakeholders or another team |
| **Delivery under pressure** | "Tell me about a tight deadline" | Shipping despite constraints |
| **Ambiguity** | "How do you handle unclear requirements?" | Navigating uncertainty, asking the right questions |
| **Mentoring** | "How have you developed others?" | Growing a junior engineer |
| **Cross-team collaboration** | "How did you work with other teams?" | Coordinating a cross-functional initiative |

### System Design Interviews

System design interviews test your ability to design large-scale systems from ambiguous requirements. They evaluate breadth of knowledge, communication, and structured thinking — not memorisation of specific architectures.

#### The System Design Framework

| Phase | Duration | What you do |
|-------|----------|-------------|
| **Clarify requirements** | 5 min | Ask questions. Functional vs non-functional. Scale, consistency, latency. |
| **High-level design** | 10 min | Draw boxes and arrows. Core components, data flow, APIs. |
| **Deep dive** | 15 min | Interviewer picks an area. Go deep on schema, algorithms, trade-offs. |
| **Discuss trade-offs** | 5 min | What would you change at 10x scale? What are the failure modes? |

#### Requirement Clarification Questions

Always ask these before designing anything:

| Category | Questions to ask |
|----------|-----------------|
| **Scale** | How many users? Requests per second? Data volume? |
| **Consistency** | Strong consistency or eventual? What happens with conflicts? |
| **Availability** | What's the SLA? Can we tolerate brief downtime? |
| **Latency** | What's acceptable? P50? P99? |
| **Access patterns** | Read-heavy or write-heavy? Hot spots? |
| **Constraints** | Budget? Team size? Existing infrastructure? Compliance? |

#### Common System Design Pitfalls

| Pitfall | Fix |
|---------|-----|
| Jumping to solution without clarifying requirements | Always spend the first 5 minutes asking questions |
| Over-engineering (adding Kafka, Redis, ML on day one) | Start simple. Add complexity only to solve stated requirements. |
| Ignoring failure modes | For every component, ask "what happens when this fails?" |
| Not discussing trade-offs | There's no perfect design. Show you understand the costs. |
| Monologue without checking in | "Does this approach make sense so far? Should I go deeper on any part?" |
| Only one "right answer" mindset | Good interviewers want to see your reasoning, not a specific architecture |

### Coding Interviews

| Stage | What to do |
|-------|-----------|
| **Read the problem** | Understand completely. Restate in your own words. |
| **Clarify** | Edge cases, constraints, input format. "Can the array be empty?" |
| **Think aloud** | Share your approach before coding. "I'm thinking a two-pointer approach because..." |
| **Start simple** | Brute force first (if asked to optimise, you can iterate) |
| **Test your code** | Walk through with a small example. Check edge cases. |
| **Discuss complexity** | Time and space. Without being asked. |

### Interview Day Logistics

| Preparation | Why it matters |
|-------------|---------------|
| Test your setup (camera, mic, IDE, whiteboard tool) | Technical failures create stress and waste time |
| Have water nearby | Talking for hours dehydrates you |
| Prepare questions for each interviewer | Shows genuine interest and evaluates the company |
| Know the interview format in advance | Ask the recruiter. Don't be surprised by a system design round. |
| Get sleep | Cognitive performance drops 25%+ when tired |

## Conducting Interviews

Being on the interviewing side is equally skilled work. Poor interviewers waste the company's time, alienate candidates, and make bad hiring decisions.

### Designing Interview Questions

| Principle | Implementation |
|-----------|---------------|
| Test what matters for the role | Don't ask algorithm puzzles for a frontend role |
| Multiple signals per question | A good question reveals several things at once |
| Consistent across candidates | Everyone gets the same core question (allows comparison) |
| Calibrated difficulty | The question should differentiate, not just pass/fail |
| No gotchas or trick questions | You're evaluating skills, not cleverness |

### Behavioural Question Design

| Weak question | Strong question | Why the strong version is better |
|---------------|-----------------|----------------------------------|
| "Are you a team player?" | "Tell me about a time you disagreed with a teammate about a technical approach." | Tests behaviour, not self-assessment |
| "How do you handle stress?" | "Describe a situation where you had competing urgent priorities." | Specific, forces a real example |
| "What's your biggest weakness?" | "Tell me about a recent piece of feedback you received and what you did with it." | Tests self-awareness and growth mindset |

### Evaluating Candidates Fairly

| Bias | How it manifests | Mitigation |
|------|-----------------|------------|
| **Halo effect** | Strong in one area → assume strong everywhere | Score each dimension independently |
| **Similar-to-me** | Favour candidates like yourself | Diverse interview panels |
| **Anchoring** | First impression colours everything | Take notes during, score after |
| **Recency** | Remember the last candidate best | Score immediately after each interview |
| **Confirmation** | Look for evidence supporting your initial impression | Actively seek disconfirming evidence |
| **Culture fit vs culture add** | Reject people who are "different" | Define culture values explicitly. Hire for values, not sameness. |

### Writing Good Interview Feedback

Your interview scorecard should be useful to the hiring committee, not just "thumbs up/down":

| Component | What to include |
|-----------|----------------|
| Summary | 2-3 sentences: overall impression and key signal |
| Specific evidence | What exactly did they say/do that informed your assessment? |
| Dimension scores | Rate each area independently (e.g., technical depth, communication, problem-solving) |
| Concerns | Any flags — be specific about what concerned you |
| Comparison to bar | "Meets/exceeds/below bar for [level]" with reasoning |

**Rules for feedback:**
- Write it immediately after the interview (memory decays fast)
- Don't read others' feedback before writing yours (prevents groupthink)
- Be specific — "good communicator" is worthless; "explained complex async concepts clearly using analogies" is useful
- Don't evaluate based on nervousness — judge substance, not presentation anxiety

### Giving Good Signal as an Interviewer

Your job is to create conditions where the candidate can demonstrate their actual ability:

| Principle | Application |
|-----------|-------------|
| Put them at ease | 2 minutes of warm-up. Introduce yourself. Explain the format. |
| Provide context | "There's no single right answer. I'm interested in your reasoning." |
| Help when stuck | Give a nudge after 2–3 minutes of struggle (not the answer) |
| Manage time | "We have 10 minutes left — let's make sure we cover X" |
| Leave time for their questions | Their questions reveal priorities, curiosity, and judgment |
| Be honest about the role | Don't oversell. Candidates who join with false expectations leave. |

### The Candidate Experience

Remember: you're being interviewed too. The candidate is evaluating your company, your team, and you.

| Signal you send | Candidate interpretation |
|-----------------|-------------------------|
| Prepared with questions, on time | This team values my time and is organised |
| Distracted, late, didn't read resume | This company doesn't care about people |
| Genuinely interested in their answers | This team values me as a person |
| Adversarial, trying to stump them | This is not a safe place to be vulnerable or learn |
| Honest about challenges | Trustworthy — I'll know what I'm signing up for |
| Only talks positives | Something's being hidden |

## Interview Types and What They Test

| Interview type | What it evaluates | Duration |
|---------------|-------------------|----------|
| **Behavioural** | Past behaviour, values, communication, self-awareness | 45–60 min |
| **System design** | Architecture skills, trade-off thinking, communication | 45–60 min |
| **Coding** | Problem-solving, code quality, algorithmic thinking | 45–60 min |
| **Take-home project** | Real-world code quality, completeness, autonomy | 2–4 hours (time-boxed) |
| **Pair programming** | Collaboration, communication, real-time problem solving | 45–60 min |
| **Culture / values** | Alignment with team norms, communication style | 30–45 min |
| **Presentation** | Ability to explain complex topics, handle questions | 30–45 min |

## After the Interview

### As a Candidate

| Outcome | Action |
|---------|--------|
| Offer received | Evaluate against your criteria (not just compensation). Ask clarifying questions. |
| Rejection | Ask for feedback (many companies provide it). Learn. Don't take it personally. |
| No response | Follow up once after a week. If still nothing, move on. |

### As an Interviewer

| Action | Timeline |
|--------|----------|
| Write feedback | Same day, within 2 hours |
| Attend debrief | Within 48 hours of final interview |
| Communicate decision to candidate | Within 1 week of final interview |

## Key Takeaways

- Use STAR (Situation-Task-Action-Result) for behavioural questions — spend 50%+ on the Action
- Build a bank of 8–10 stories covering common dimensions (leadership, failure, conflict, delivery)
- System design interviews: always clarify requirements first, start simple, discuss trade-offs
- As an interviewer: test what matters for the role, not abstract puzzles
- Score candidates independently per dimension to avoid halo effects
- Write feedback immediately after each interview — before reading others' assessments
- Both sides are evaluating. Candidate experience is employer branding.
- The best interviews feel like a conversation between potential colleagues, not an interrogation
