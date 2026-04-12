# Presenting Technical Ideas

## Context

Technical excellence is necessary but not sufficient. The ability to communicate a technical idea clearly — to the right audience, at the right level of detail, with the right framing — is what converts good engineering into adopted decisions and funded work.

The failure mode that kills most technical proposals is not being wrong. It is being right in a way the audience cannot follow.

**Why technical presentations fail**:
| Failure | Root cause |
|---|---|
| Audience loses interest in the first 3 minutes | Led with implementation detail, not with the problem |
| No decision reached | The ask was missing or unclear |
| "Can we follow up?" | Too much content for the time; no clear recommendation |
| "We need more data" | Stakes and urgency weren't established |
| Executive approves but nothing happens | The right people weren't in the room; ownership not assigned |

---

## Know Your Audience

Every presentation needs to be rewritten for its audience. The same proposal will need different framing for an architecture review board than for a VP of Engineering.

**Audience ladder**:

| Audience | What they care about | Appropriate depth | Language |
|---|---|---|---|
| **Engineers (peers)** | Correctness, tradeoffs, implementation details | Deep — show the code/design | Technical terminology fine |
| **Tech leads / architects** | System boundaries, consistency, long-term maintainability | Medium — design decisions and alternatives | Technical, concise |
| **Engineering manager** | Timeline, risk, team impact, dependencies | Light-medium — outcomes and blockers | Low jargon |
| **VP / Director** | Business outcomes, cost, strategic fit | Headline + 2 supporting points | No acronyms, no implementation |
| **C-level** | ROI, competitive risk, regulatory exposure | 3 sentences max + recommendation | Business language only |

**Practical test**: Write your executive summary first. If it can't stand alone for a non-technical reader in 5 sentences, you haven't identified the core argument yet.

---

## The Situation-Complication-Resolution (SCR) Structure

SCR is the most reliable narrative structure for technical proposals. It moves the audience from shared context to productive action.

```
SITUATION   Shared context — what everyone agrees is true
            "We deploy to production 12 times per week."

COMPLICATION  The tension or problem — why the situation is no longer acceptable
            "Since we moved to a microservices architecture, 30% of deployments
             require manual coordination across 4 teams, causing 2–3 hour delays
             and frequent Friday afternoon incidents."

RESOLUTION  Your recommendation — specific, bounded, with an ask
            "I'm proposing we implement a deployment orchestration service that
             coordinates cross-service deploys automatically. Prototyped in 2 weeks,
             full rollout in Q2. I need 30 minutes from the platform team leads today
             to validate the integration points."
```

This structure works because the audience is oriented (Situation), feels the urgency (Complication), and knows exactly what you need from them (Resolution).

---

## Executive Summary Format

Every technical proposal should have a one-page (or less) executive summary that can be read in under 3 minutes:

```markdown
## [Title of proposal]

**Problem**: [1 sentence — what is broken or missing]
**Impact**: [Quantified — cost, frequency, risk, who is affected]
**Proposed solution**: [1 sentence — what you're recommending]
**Why this approach**: [2–3 bullet points — why this is better than the alternatives]
**Tradeoffs**: [What you are NOT doing; what this costs]
**Ask**: [Specific decision, resource, or approval needed]
**Timeline**: [Milestones, dependencies]
**Owner**: [Who is accountable for delivery]
```

---

## Speaking to Non-Technical Audiences

Non-technical audiences are not less intelligent. They are applying a different priority filter: outcomes, risk, and cost — not implementation elegance.

**Non-technical communication rules**:
1. **No acronyms without definition**. "K8s", "gRPC", "SSO" mean nothing.
2. **Lead with business impact, not technical cause**. "This causes the checkout flow to fail for 2% of users on mobile" not "there's a race condition in the payment service".
3. **Use analogies for architecture concepts**. "The database is a filing cabinet. Right now everyone in the building shares one and it's jammed. We're adding a second filing cabinet for the finance team."
4. **Three points maximum**. If you can't say it in three points, you haven't chosen your three most important points yet.
5. **Make the ask explicit**. Non-technical audiences will not infer what you need. Say: "At the end of this conversation I need a go/no-go decision on Option A."

---

## Architecture Review Presentation

Architecture review boards typically evaluate proposals for correctness, risk, and fit with existing systems. They will probe for gaps.

**Structure for architecture reviews**:
1. **Context** (2 min): What problem does this solve? What scale?
2. **Design diagram** (5 min): Component view; data flow; deployment boundary. Walk it slowly.
3. **Key decisions and alternatives** (5 min): What approaches were considered and why this one?
4. **Failure modes** (3 min): What happens when X fails? How does it recover?
5. **Observability** (2 min): How will we know it's working? What do we alert on?
6. **Open questions** (2 min): What are you still uncertain about? This is not weakness — it is honesty.

**Anticipating review questions**:
- "What's the failure mode if [dependency] is unavailable?"
- "How does this interact with [existing system]?"
- "What's the data migration path?"
- "How do we roll this back if it fails in production?"
- "What does operational load look like at 10× current volume?"

Prepare brief answers for each. You don't need to read them out — just be ready.

---

## Demo Preparation

A live demo is high-impact and high-risk. Technical failure in a demo is recoverable; a demo that doesn't demonstrate the thing you're claiming is not.

**Demo preparation checklist**:
```
[ ] Demo environment is isolated from production data
[ ] "Happy path" run-through completed 3x in the 24 hours before the demo
[ ] Fallback: screenshots or a recording if live demo fails
[ ] Context-setting: 60-second framing before touching the keyboard
    "What you're about to see is [system] doing [task]. Watch for [specific thing]."
[ ] No live Slack visible on screen during demo
[ ] Every menu click is intentional — no browsing mid-demo
[ ] The demo ends on the outcome, not on configuration screens
[ ] You have a closing line: "So what you saw there was X, which means Y for us"
```

---

## Handling Q&A

Q&A is where presentations are won or lost for senior audiences.

**Q&A principles**:

| Situation | Response |
|---|---|
| You know the answer | Answer directly in 2–3 sentences. Don't expand unless asked. |
| You're not sure | "I don't have the exact number, but I'll send it by EOD." Never guess a statistic. |
| The question is off-topic | "That's a good question. It's slightly outside today's scope — can I follow up?" |
| The question reveals a gap | "You've identified something I hadn't fully considered. Let me think about that and come back to you." |
| Hostile or politically charged | Acknowledge the concern: "That's a real concern." Ask a clarifying question before responding. |
| "Can you just build it in X instead?" | "What problem would that solve? I want to make sure I've understood the concern." |

**What to avoid**:
- Defensive body language (arms crossed, looking down) when challenged
- "That's a great question!" — removes credibility, sounds performative
- Repeating the question back at length before answering
- Going over time to finish your slides — respect the Q&A time

---

## Self-Assessment Checklist

```
Structure
  [ ] My proposals lead with the problem, not the solution
  [ ] I use SCR structure (Situation-Complication-Resolution) for proposals
  [ ] My executive summary can be read alone in under 3 minutes

Audience adaptation
  [ ] I adjust depth and vocabulary for the specific audience before presenting
  [ ] Non-technical audiences receive business-impact framing first
  [ ] I have a specific ask at the end of every presentation

Demo readiness
  [ ] I have run through demos at least 3× before any live presentation
  [ ] I have a fallback (screenshots/recording) for live demos
  [ ] My demos end on the outcome, not on UI chrome

Q&A
  [ ] I say "I don't know; I'll find out" when I don't know
  [ ] I can acknowledge gaps or challenges without becoming defensive
  [ ] I stay within time on my main presentation to preserve Q&A time
```
