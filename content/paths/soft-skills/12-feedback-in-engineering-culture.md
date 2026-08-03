---
title: "Feedback in Engineering Culture"
weight: 12
---

# Feedback in Engineering Culture

Building on the fundamentals of feedback (the SBI model, Radical Candor, and basic code review etiquette), this section explores the harder edges: giving upward feedback, navigating emotionally charged situations, building systematic feedback practices, and creating a team culture where honest feedback flows naturally.

## Feedback Beyond the Basics

Most engineers learn to give simple code review comments. Fewer learn to navigate these situations:

| Scenario | Why it's hard | What makes it work |
|----------|--------------|-------------------|
| Telling a senior engineer their design has problems | Power dynamics, imposter syndrome | Focus on data and consequences, not opinion |
| Giving feedback to your manager | Fear of retaliation, awkward power inversion | Frame as helping them help you |
| Addressing persistent behaviour patterns | Feels personal, risk of damaging relationship | Accumulate specific examples, choose the right moment |
| Delivering feedback across cultures | Different norms around directness | Ask how they prefer to receive feedback |
| Giving feedback when you're emotionally triggered | Risk of being harsh or unfair | Wait 24 hours. Write it down. Edit. Then deliver. |

## Upward Feedback

Giving feedback to people with more authority requires different framing — not because their feelings matter more, but because the power dynamic creates real consequences for how your feedback lands.

### Framing Upward Feedback

| Instead of | Try | Why it works |
|-----------|-----|--------------|
| "You're micromanaging" | "When you check in daily on task X, it signals you don't trust my judgment. Could we try weekly check-ins?" | Describes impact on you, proposes alternative |
| "Your meetings waste time" | "I leave our 1:1s unsure what to focus on. Could we start with my blockers?" | Makes it about your experience, offers structure |
| "You don't give us enough context" | "I'd make better decisions if I understood the business pressure driving this deadline" | Positions as wanting to help, not criticising |
| "You play favourites" | "I notice [Person] gets high-visibility projects. I'd like a shot at one — what do I need to demonstrate?" | Focuses on what you want, not what's unfair |

### When to Give Upward Feedback

| Good timing | Bad timing |
|-------------|-----------|
| 1:1 meetings (that's what they're for) | Public settings (all-hands, team meetings) |
| After you've observed a pattern (3+ instances) | After one frustrating incident (could be a bad day) |
| When you have a specific request | When you only have a complaint |
| When the relationship has trust deposits | When the relationship is already strained |

## Feedback in 1:1 Meetings

The 1:1 is the most underused feedback channel. Most engineers use it for status updates. Better engineers use it for growth.

### Requesting Feedback in 1:1s

Generic questions get generic answers. Be specific:

| Weak question | Strong question |
|---------------|-----------------|
| "Do you have feedback for me?" | "What's one thing I could do differently in design reviews?" |
| "How am I doing?" | "Am I communicating enough context in my PRs for new team members?" |
| "Any concerns?" | "Is there a situation in the last sprint where I should have escalated earlier?" |
| "Am I on track for promotion?" | "What's the gap between my current impact and what you'd expect at senior level?" |

### Delivering Feedback in 1:1s

| Step | Purpose | Example |
|------|---------|---------|
| Ask permission | Signals this is deliberate, lets them prepare | "I have some feedback about sprint planning. Is now a good time?" |
| State the observation | Ground it in specifics (SBI) | "In the last three sprints, your estimates were 2-3x off from actuals." |
| Explore together | You might be missing context | "What's causing the gap? Are the tasks unclear, or is it scope creep?" |
| Agree on action | Feedback without action is venting | "Let's try breaking tasks into half-day chunks this sprint and see if that helps." |
| Follow up next time | Shows you care about progress | "How did the smaller-task approach work this sprint?" |

## Feedback Across Cultures

Engineering teams are increasingly global. What reads as "direct and helpful" in one culture reads as "rude and aggressive" in another.

### Cultural Feedback Dimensions

| Dimension | Low-context cultures (US, Netherlands, Israel) | High-context cultures (Japan, Korea, India) |
|-----------|-----------------------------------------------|---------------------------------------------|
| Directness | Explicit, say exactly what you mean | Indirect, imply through context and phrasing |
| Face | Criticism is about the work | Criticism can cause loss of face |
| Written vs verbal | Written feedback is normal | Sensitive feedback must be verbal and private |
| Public praise | Welcome and expected | Can be embarrassing ("why single me out?") |
| Disagreeing with seniors | Expected and valued | Requires careful framing to avoid disrespect |

### Practical Adaptations

- **Ask early:** "How do you prefer to receive feedback? Written or verbal? In our 1:1 or in real-time?"
- **Start with relationship:** In high-context cultures, invest in the relationship before challenging
- **Watch for indirect signals:** "That's an interesting approach" might mean "I disagree but won't say directly"
- **Adjust, don't patronise:** Adapting your style isn't dumbing down — it's communicating effectively

## The Feedback Frequency Problem

Most teams give too little feedback, too late.

| Feedback frequency | Result |
|-------------------|--------|
| Never | Problems fester, surprises at performance reviews |
| Quarterly (performance reviews only) | Too infrequent to change behaviour |
| Monthly (1:1s only) | Better, but missed moments pile up |
| Weekly | Pattern recognition becomes possible |
| Real-time (within 24 hours of the event) | Most effective — context is fresh for both parties |

### Building a Real-Time Feedback Habit

1. **Set a trigger:** After every meeting, ask yourself "Was there something worth feeding back?"
2. **Lower the bar:** Feedback doesn't need to be a formal conversation. "Hey, your explanation in standup was really clear today" counts.
3. **Balance positive and constructive:** If people only hear from you when something's wrong, they'll dread your messages.
4. **Use Slack/DMs for lightweight praise:** "Nice catch on that race condition in the PR" takes 10 seconds and makes someone's day.
5. **Save complex feedback for synchronous:** Text strips tone. If it's nuanced or could be misread, say it in a call or face-to-face.

## Handling Criticism Poorly Delivered

Not all feedback comes wrapped in SBI and Radical Candor. Sometimes it arrives as:
- "This code is terrible"
- "You clearly didn't think about this"
- "Why would you do it that way?"

### The Response Framework

```
┌─────────────────────────────────────────────────────┐
│ Step 1: Pause. Don't react. Breathe.                │
│ Step 2: Separate the signal from the delivery.      │
│ Step 3: Ask a clarifying question.                  │
│ Step 4: Decide later whether to act on it.          │
│ Step 5: (Optional) Address the delivery separately. │
└─────────────────────────────────────────────────────┘
```

| Scenario | Unhelpful response | Effective response |
|----------|-------------------|-------------------|
| "This code is terrible" | "No it isn't!" (defensive) | "What specifically would you improve?" (curiosity) |
| "You clearly didn't think about X" | "I did think about it!" (justifying) | "You're right I didn't cover X. What should I consider?" (learning) |
| "Why would you do it that way?" | "Because I'm not an idiot" (hostile) | "I chose it because [reason]. Is there a better approach you'd suggest?" (collaborative) |

### When to Push Back on Delivery

If someone consistently delivers feedback poorly:

1. **Address it privately**, not in the moment
2. **Use SBI on them:** "When you said 'this code is terrible' in the PR (Situation + Behaviour), I shut down and stopped engaging with your other comments (Impact)."
3. **Propose an alternative:** "I'd find it much more useful if you said what specifically needs changing and why."
4. **Acknowledge the good intent:** "I can tell you care about code quality. I want to hear your feedback — the delivery just makes it hard."

## Feedback Anti-Patterns in Teams

| Anti-pattern | Symptom | Cure |
|-------------|---------|------|
| Feedback avoidance | Issues raised only in retros (or never) | Normalise real-time micro-feedback |
| Feedback sandwich (praise-criticism-praise) | People tune out the praise, waiting for the "but" | Just be direct. The sandwich is a well-known trick. |
| Proxy feedback | "Someone said you..." (anonymous, unverifiable) | Own your feedback. If you won't say who, don't relay it. |
| Accumulation bombs | Months of stored-up issues delivered at once | Give feedback within a week of the event |
| Reciprocal suppression | "I won't mention your issue if you don't mention mine" | Create psychological safety so feedback isn't transactional |
| Performance review surprises | First time hearing negative feedback is in a formal review | Nothing in a review should be new information |

## Building a Feedback-Rich Team Culture

Culture change doesn't happen through mandates. It happens through modelling:

| Action | Signal it sends |
|--------|----------------|
| Ask for feedback publicly ("What could I have done better in that meeting?") | Vulnerability is safe here |
| Thank someone for tough feedback | Honesty is rewarded, not punished |
| Act on feedback visibly | Feedback leads to change, so it's worth giving |
| Give micro-feedback in real-time ("Nice commit message" / "That test name is confusing") | Feedback is normal, lightweight, constant |
| Share your own mistakes openly ("I should have escalated sooner. Here's what I'll do differently.") | Growth mindset is the norm |

## Feedback in Code Reviews — Advanced Patterns

Beyond basic PR etiquette, senior engineers develop nuanced review habits:

### Scaling Your Review Impact

| Level | Review behaviour | Impact |
|-------|-----------------|--------|
| Junior | Check correctness, style | Catches bugs |
| Mid | Check design, naming, testability | Improves architecture |
| Senior | Check system impact, future maintenance, team patterns | Shapes engineering culture |
| Staff+ | Review for strategic alignment, cross-team implications | Prevents systemic problems |

### The "Why" Review

Instead of: "Change this to X"
Try: "What would happen if this function is called concurrently? I'm thinking about the case where two requests hit this endpoint within the same millisecond."

Explaining *why* something concerns you teaches the author to spot the pattern themselves next time.

### Feedback Load Management

When a PR has many issues:

1. **Triage:** Fix the architectural issue first — style nits don't matter if the design is wrong
2. **Limit volume:** If you have 20 comments, pick the 5 most important. File a follow-up for the rest.
3. **Offer to pair:** "This has a few structural issues. Want to pair for 20 minutes? It'll be faster than async comments."
4. **Separate concerns:** "Approving the logic. Separate PR for the rename/refactor would be easier to review."

## Feedback Cadence for Different Relationships

| Relationship | Recommended cadence | Primary channel |
|-------------|--------------------|-----------------| 
| Direct report | Weekly (1:1) + real-time | 1:1 meeting, DM |
| Peer on your team | Real-time + sprint retro | PR comments, DM, retro |
| Cross-team collaborator | After significant interactions | DM or scheduled call |
| Manager (upward) | Biweekly or monthly in 1:1 | 1:1 meeting |
| Skip-level | Quarterly (if the opportunity exists) | Skip-level 1:1 |

## Measuring Feedback Culture Health

| Signal | Healthy | Unhealthy |
|--------|---------|-----------|
| Time to feedback | Hours to days | Weeks to months (or never) |
| Direction | Multi-directional (up, down, sideways) | Top-down only |
| Specificity | Concrete examples, actionable | Vague ("do better") |
| Response | "Thank you, let me think about that" | Defensive, dismissive, or silent |
| Consequence | Visible behaviour change | Nothing changes |
| Consistency | Regular, normalised | Only when forced (reviews, PIPs) |

If you see more than two "unhealthy" signals on your team, feedback culture is the first thing to fix — because all other improvement depends on honest communication.

## Key Takeaways

- Upward feedback requires different framing but is just as important as peer feedback
- Ask specific questions in 1:1s — generic questions get generic answers
- Cultural context matters. Ask people how they prefer to receive feedback
- Real-time feedback (within 24 hours) is dramatically more effective than quarterly reviews
- When feedback is poorly delivered, separate the signal from the delivery — then address both
- The strongest teams have high feedback frequency and low feedback formality
- Senior code reviewers explain *why* something is a problem, not just *what* to change
- Model the culture you want: ask for feedback, thank people for it, and act on it visibly
