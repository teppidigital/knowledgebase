# Engineering Leadership

## Context

Engineering leadership is not a title — it's a set of behaviours. It begins before anyone gives you the "Tech Lead" label: you start leading the moment you take ownership of outcomes beyond your own tasks.

The distinction between **Tech Lead** and **Engineering Manager** matters because they are different jobs, often conflated:

| Dimension | Tech Lead | Engineering Manager |
|---|---|---|
| Primary accountability | Technical quality and direction | People health and performance |
| Output measured by | System design quality, delivery, technical debt trend | Team velocity, retention, growth, morale |
| Spends time on | Architecture, code review, unblocking technical problems | 1:1s, performance, hiring, org coordination |
| Reports to | Engineering Manager or Staff Engineer | Director / VP of Engineering |
| Common failure mode | Becoming the single point of bottleneck | Losing technical credibility |

A strong tech lead **amplifies the team** — every hour you invest in a junior engineer who then ships independently returns more than that hour spent coding yourself.

---

## The Delegation Spectrum

Delegation is not binary (do it yourself / hand it off entirely). There are 7 levels:

| Level | Description | When to use |
|---|---|---|
| 1. Tell | You decide and inform | Emergency; irreversible decision |
| 2. Sell | You decide and persuade | Significant change needing buy-in |
| 3. Consult | You gather input, then decide | Important decision; team expertise matters |
| 4. Agree | You decide together consensually | Cross-team impact; shared ownership |
| 5. Advise | They decide; you offer perspective | Growth task; you retain veto |
| 6. Inquire | They decide; they update you after | Trusted team member; normal work |
| 7. Delegate | They decide fully | Fully developed capability; low risk |

**Most tech leads get stuck at level 1–3**. The goal is to push as many decisions as possible to levels 5–7, saving your own involvement for decisions that are actually irreversible or high-risk.

---

## The 1:1

The 1:1 is the most important recurring meeting a tech lead or manager runs. Done well, it surfaces problems before they become crises and accelerates growth.

**What a 1:1 is NOT**:
- A status update (use a written async update for that)
- A project review (cover that in planning meetings)
- A performance review (that has its own cadence)

**The 1:1 is for**:
- Understanding how the person is doing (workload, morale, blockers)
- Career growth and development goals
- Feedback exchange (both directions)
- Removing obstacles only you can remove

### 1:1 agenda template (shared doc, updated before each meeting)

```markdown
## 1:1 — [Name] / [Your Name] — [Date]

### Their agenda (add before meeting)
-
-

### Your agenda (add before meeting)
-
-

### Running notes

#### Career / growth
-

#### Feedback
-

#### Blockers I need to remove
-

### Actions from this meeting
| Action | Owner | By when |
|---|---|---|
| | | |

### Previous actions
(carry forward unresolved items)
```

**3-question minimum for every 1:1**:
1. "What's going well that you want to keep doing?"
2. "What's getting in your way right now?"
3. "What would help you grow most in the next 3 months?"

---

## Decision Ownership

Ambiguity about who owns a decision is a team tax. Every significant decision should have a named owner — not a committee, not "the team", not "we'll figure it out".

### DACI framework (Driver, Approver, Contributor, Informed)

| Role | Count | Responsibility |
|---|---|---|
| **Driver** | 1 | Researches, writes the proposal, runs the process |
| **Approver** | 1 (max 2) | Makes the final call; accountable for outcome |
| **Contributor** | N | Provides input; shapes the decision; must be listened to |
| **Informed** | N | Notified of the outcome; not consulted during |

**Rule**: If there are 2 Approvers who disagree, escalate to 1. Never "joint approval" — it means neither owns it.

---

## Technical Direction

### Setting and maintaining direction

A tech lead's job on direction is:
1. Translate business goals into engineering constraints and priorities.
2. Establish the architectural principles that bound daily decisions.
3. Keep the team aligned on those principles without approving every PR.

**Architecture principles template** (3–7 rules, measurable):
```
1. Services communicate via events, not direct DB reads.
   Measurable: dependency-cruiser lint rule in CI.

2. All external input is validated at the boundary (parse, don't validate).
   Measurable: no-unvalidated-input ESLint rule passes.

3. No service is an island — every deployment is observable from day 1.
   Measurable: DORA metric dashboard exists; alerts configured.
```

### Tech debt governance

Not all debt is equal. Classify before acting:

| Category | Definition | Action |
|---|---|---|
| **Deliberate / strategic** | Knowingly took a shortcut to ship faster | Document the shortcut; schedule repayment in next cycle |
| **Inadvertent** | Didn't know a better approach existed | Address at next natural touch point |
| **Bit rot** | Worked then; aged out | Tackle in background cleanup / dedicated sprint |
| **Architectural** | Wrong abstraction; must be redesigned | Requires major investment; write RFC, get buy-in |

Rule: **Pay down the minimum to keep the team productive, not everything that bothers you.**

---

## Growing Others

The best measure of a tech lead is how the team performs when you're on holiday.

### Levelling model — observable behaviours

| Level | What they need from you |
|---|---|
| Junior (L1–2) | Clear tasks, frequent check-ins, pairing, safe space to make mistakes |
| Mid (L3) | Direction + context; stretched assignments just outside comfort zone; candid feedback |
| Senior (L4) | Ownership of outcomes not just tasks; peer feedback; "here's the opportunity" not "here's the task" |
| Staff+ | Alignment on strategy; visibility to leadership; co-ownership of technical direction |

### Stretch assignment framework

When assigning a growth task:
1. **State the goal**, not the method: "We need to reduce P99 by 50%." Not: "Add a Redis cache here."
2. **Define success up front**: What does done look like? What's the acceptance criterion?
3. **Set a check-in point**, not a deadline: "Let's talk on Thursday to see if you're unblocked."
4. **Debrief after**: "What would you do differently? What did you learn?"

---

## Common Anti-Patterns

| Anti-pattern | What it looks like | Fix |
|---|---|---|
| **Bottleneck tech lead** | Every PR waiting on one reviewer; nothing ships without their input | Delegate review areas; grow a second reviewer |
| **Hero culture** | One person rescues every incident; team doesn't learn | Rotate on-call; run blameless post-mortems; share runbooks |
| **Invisible standards** | "I'll know good code when I see it" | Write down the principles; encode them in lint rules and ADRs |
| **Scope creep acceptance** | "We might as well add X while we're in here" | Say "let's log that as a separate ticket" every time |
| **Feedback avoidance** | Team finds out about problems during performance reviews | Monthly no-agenda 1:1s; normalise small regular feedback |

---

## Self-Assessment Checklist

```
Direction and ownership
  [ ] Our team has 3–7 written architecture principles that new engineers read on day 1
  [ ] Every significant decision has a named owner (not "the team")
  [ ] I can describe our top 3 pieces of tech debt and have a plan for each

People
  [ ] I run weekly 1:1s with a shared agenda doc
  [ ] I can name the growth goal for each person on my team
  [ ] I have delegated at least one thing this month that makes me slightly uncomfortable

Communication
  [ ] I can describe the team's priorities to a VP in 2 minutes
  [ ] I proactively update stakeholders on delays before they ask
  [ ] I write RFCs when I need buy-in instead of asking in Slack

Execution
  [ ] The team ships reliably without my direct involvement in every task
  [ ] I review for design and direction, not style (that's the linter's job)
  [ ] I call out scope creep in planning, not in retrospective
```
