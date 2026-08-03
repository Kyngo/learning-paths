---
title: "Mentoring and Teaching"
weight: 14
---

# Mentoring and Teaching

The engineers who create the most lasting value aren't always the ones who write the most code — they're the ones who make everyone around them better. Mentoring, pair programming, knowledge sharing, and effective onboarding are force multipliers that compound over time.

## Why Engineers Should Teach

| Reason | Mechanism |
|--------|-----------|
| Teaching deepens your understanding | You can't explain what you don't truly know. Gaps become obvious when you try to teach. |
| It scales your impact | One great engineer produces X. One engineer who mentors five produces 5X (and growing). |
| It builds trust and influence | People follow those who invest in their growth. |
| It's a leadership prerequisite | You cannot lead effectively without being able to transfer knowledge. |
| It creates bus-factor resilience | Knowledge hoarding is a single point of failure. |

## Being a Good Mentor

Mentoring is not telling someone what to do. It's helping them develop the ability to figure things out themselves.

### Mentoring vs Managing vs Teaching

| Role | Relationship | Approach | Timeline |
|------|-------------|----------|----------|
| **Mentor** | Guide, advisor | Ask questions that provoke thinking | Ongoing, relationship-based |
| **Manager** | Supervisor, evaluator | Set goals, provide resources, assess performance | Role-based, organisational |
| **Teacher** | Expert, instructor | Transfer specific knowledge or skills | Session-based, topic-focused |
| **Coach** | Facilitator | Help them find their own answers through inquiry | Goal-oriented, time-bounded |
| **Sponsor** | Advocate, door-opener | Create opportunities, put their name forward | Relationship-based, political |

### The Mentor's Toolkit

| Tool | When to use | Example |
|------|-------------|---------|
| **Ask, don't tell** | When they can figure it out themselves | "What would happen if this function is called concurrently?" |
| **Think aloud** | When sharing your reasoning process | "I'd start by checking the logs because timeout errors usually indicate..." |
| **Share failures** | When they need permission to fail | "I once took down production by forgetting to add a WHERE clause. Here's what I learned." |
| **Set challenges** | When they need stretch opportunities | "Would you be up for leading the next design review?" |
| **Connect to resources** | When they need knowledge you don't have | "Talk to [person] — they solved exactly this last quarter." |
| **Provide safety** | When they're taking risks | "Try the new approach. I'll review it, and if it doesn't work, we'll learn something." |

### Questions Mentors Ask

The quality of your questions determines the quality of their thinking:

| Instead of | Ask |
|-----------|-----|
| "Here's how to do it" | "What approaches have you considered?" |
| "That's wrong" | "What would happen in the case where X?" |
| "You should use pattern Y" | "What trade-offs are you weighing?" |
| "Let me fix that for you" | "What's your next step for debugging this?" |
| "I would have done it differently" | "Walk me through your reasoning" |

### Common Mentoring Anti-Patterns

| Anti-pattern | Problem | Better approach |
|-------------|---------|----------------|
| Solving their problems for them | They don't develop problem-solving skills | Guide them to the answer through questions |
| Only mentoring mini-you | Bias toward people who think like you | Seek out mentees with different backgrounds |
| Ghost mentoring | Agreeing to mentor then never being available | Set a recurring cadence (biweekly 30 min minimum) |
| Only technical guidance | Ignoring career, communication, growth | Ask about their goals, not just their code |
| Waiting to be asked | Many people don't know they can ask | Proactively offer: "I noticed X — want to talk through it?" |
| One-size-fits-all | Different people need different styles | Ask how they learn best. Adapt. |

## Pair Programming

Pair programming is the highest-bandwidth knowledge transfer mechanism in engineering. Done well, it transfers not just *what* to do, but *how* to think.

### Pairing Modes

| Mode | Structure | Best for |
|------|-----------|----------|
| **Driver-Navigator** | One types, one directs strategy | Teaching, complex problems |
| **Ping-Pong** | Alternate writing test / making it pass (TDD) | Equal-skill pairs, maintaining focus |
| **Unstructured** | Fluid switching between who codes | Exploration, quick debugging |
| **Mob/Ensemble** | 3+ people, one driver, group navigates | Onboarding, complex decisions, knowledge spreading |

### Making Pairing Effective

| Principle | Implementation |
|-----------|---------------|
| Equal voice | Navigator's ideas matter as much as driver's. Switch roles every 25 minutes. |
| Think aloud | Both people verbalise their reasoning. Silence means someone is lost. |
| Small steps | Don't jump ahead. Make one thing work, then the next. |
| Take breaks | Pairing is mentally intense. 25 on, 5 off (pomodoro). |
| No grabbing | Never take the keyboard without asking. "Mind if I try something?" |
| Patient teaching | If you're senior, let them struggle productively before jumping in |

### When Pairing Works (and When It Doesn't)

| Pair | Works great | Doesn't work |
|------|-------------|-------------|
| Senior + Junior | Knowledge transfer, real-time teaching | Routine work the senior finds boring |
| Equal skill | Hard problems, design decisions | Mundane tasks that don't benefit from two brains |
| Cross-team | System understanding, reducing silos | When there's no shared context to build on |

### Remote Pairing Tips

| Challenge | Solution |
|-----------|----------|
| Screen lag | Use VS Code Live Share or similar (shared editor, not screen share) |
| Hard to point at code | Use line numbers verbally, or draw on shared whiteboard |
| Fatigue | Shorter sessions (60 min max). More breaks. |
| Awkward silence | Establish that silence is OK — someone is thinking |
| Timezone differences | Find overlapping hours. Agree on a shared schedule. |

## Knowledge Sharing

Individual mentoring helps one person. Knowledge sharing scales to the entire team or organisation.

### Knowledge Sharing Formats

| Format | Effort | Reach | Shelf life |
|--------|--------|-------|------------|
| Slack message / thread | Low | Team | Days |
| Code comments | Low | Anyone reading the code | Long |
| PR description | Low-Medium | Team + future archaeologists | Long |
| README / wiki page | Medium | Organisation | Long (if maintained) |
| Brown bag / tech talk | Medium-High | Team / org (live) | Short (unless recorded) |
| Blog post (internal) | High | Organisation | Long |
| Workshop / hands-on session | High | Attendees | Medium (skill retention varies) |
| Recorded video walkthrough | High | Organisation | Medium-Long |

### Running Effective Tech Talks

| Element | Recommendation |
|---------|---------------|
| Duration | 20–30 minutes (attention drops after 30) |
| Structure | Problem → What I tried → What worked → Demo → Q&A |
| Level | Aim for the "interested non-expert" — explain enough context |
| Engagement | Live demos, code examples, ask the audience questions |
| Recording | Always record. The async viewers outnumber live attendees. |
| Follow-up | Post slides/recording + a short written summary |

### Creating a Learning Culture

| Behaviour | Impact |
|-----------|--------|
| Schedule regular "show and tell" sessions | Learning becomes routine, not exceptional |
| Celebrate teaching, not just building | Incentivises knowledge sharing |
| Maintain a team wiki with decision records | Institutional memory persists |
| Rotate "on-call" or "investigator" roles | Everyone learns different parts of the system |
| Make documentation part of "done" | Knowledge sharing is built into delivery |

## Onboarding Others

Onboarding is the highest-leverage teaching you'll ever do. A developer who gets productive in 2 weeks instead of 6 weeks saves 4 weeks of salary and months of frustration.

### The 30-60-90 Day Framework

| Timeframe | Goal | They should be able to |
|-----------|------|----------------------|
| **Week 1** | Orient | Navigate the codebase, run the app locally, find documentation |
| **Days 8–30** | Contribute | Ship small changes independently, review PRs, participate in ceremonies |
| **Days 31–60** | Own | Take ownership of a feature or component, drive small decisions |
| **Days 61–90** | Influence | Propose improvements, mentor newer joiners, contribute to architecture discussions |

### Onboarding Checklist (Week 1)

| Category | Items |
|----------|-------|
| **Access** | Git repos, CI/CD, cloud accounts, monitoring dashboards, Slack channels |
| **Context** | Architecture overview, team norms doc, key stakeholders map |
| **Setup** | Development environment running, tests passing, can deploy to test |
| **Human** | Assigned buddy, intro meetings with key people, 1:1 with manager |
| **First task** | A real (but small) ticket — not busywork. Something that ships. |

### Being an Onboarding Buddy

| Do | Don't |
|----|-------|
| Proactively check in daily (first 2 weeks) | Wait for them to ask (they won't — they don't want to seem incompetent) |
| Share "unwritten rules" | Assume they'll figure out the culture by osmosis |
| Pair on their first few tasks | Just assign work and hope for the best |
| Introduce them to key people personally | Send a list of names with no context |
| Celebrate their first merge | Let it pass unremarked |
| Tell them it's OK to be confused | Let them assume everyone else "got it" instantly |

### Onboarding Anti-Patterns

| Anti-pattern | Impact | Fix |
|-------------|--------|-----|
| "Sink or swim" | High attrition, slow ramp-up, anxiety | Structured first 30 days with buddy |
| Busywork tickets | New hire feels useless and undervalued | Give real work with appropriate support |
| Information firehose (day 1 brain dump) | Retention is near zero | Spread context over weeks, just-in-time |
| No feedback for 3 months | Bad habits form uncorrected | Weekly feedback in first month |
| Outdated onboarding docs | Frustration, lost time, bad first impression | Review and update quarterly |

## Scaling Yourself Through Teaching

As you grow senior, your individual contribution hits a ceiling. Teaching is how you break through it:

| Career level | Primary impact mechanism | Teaching role |
|--------------|------------------------|---------------|
| Junior | Personal output (code written) | Learn from others |
| Mid | Personal output + quality (code reviewed, bugs prevented) | Explain in PRs and docs |
| Senior | Team output (unblocking others, design leverage) | Mentor, pair, teach |
| Staff | Organisation output (standards, patterns, enablement) | Create frameworks others use |
| Principal | Industry impact (publications, open source, conference talks) | Teach beyond your org |

### The Teaching Ladder

1. **Write it down** — Document what you know (lowest barrier to entry)
2. **Review thoroughly** — Transfer knowledge through PR feedback
3. **Pair regularly** — Real-time teaching, highest bandwidth
4. **Present to your team** — Brown bags, tech talks, demos
5. **Mentor individuals** — Long-term investment in specific people
6. **Create learning paths** — Curated sequences for common growth needs
7. **Teach externally** — Blog, conference talk, open source contribution

## Key Takeaways

- Mentoring is asking questions, not providing answers — develop their thinking, not their dependency
- Pair programming is the highest-bandwidth knowledge transfer; use Driver-Navigator for teaching
- Knowledge sharing should be built into daily work (PR descriptions, code comments, docs), not just special events
- Onboarding is your highest-leverage teaching opportunity — a structured 30-60-90 plan beats "sink or swim"
- Your proactive check-ins matter more than their willingness to ask — new hires won't ask for fear of seeming incompetent
- As you grow senior, teaching is how you break through the ceiling of individual contribution
- The strongest teams have learning cultures where everyone teaches and everyone learns
