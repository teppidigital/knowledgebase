# Engineering Communication

## Context

Communication is the highest-leverage skill an engineer can develop. Technical output — no matter how correct — is invisible if it isn't understood by the people who need to act on it. As engineers grow from individual contributors to senior and staff roles, the ratio of time spent writing and talking vs. coding increases dramatically.

**The most common failure modes**:
| Failure mode | Observable symptom | Root cause |
|---|---|---|
| Explaining the "how" before the "why" | Listeners disengage; questions miss the point | Assuming shared context |
| Burying the headline | Decision-makers read halfway and make the wrong call | Fear of seeming blunt |
| Passive voice accountability | "The deployment failed" instead of "I deployed the wrong version" | Discomfort with ownership |
| Over-detailed async messages | Multiple replies needed just to extract the point | Not considering the reader's time |
| Speaking in acronyms to non-engineers | Room goes quiet; follow-up 1:1 carries the real conversation | Failure to context-switch |

**Communication channels and when to use them**:
| Signal needed | Channel | Why |
|---|---|---|
| Urgent and complex | Synchronous (call / meeting) | Real-time adjustment; read the room |
| Documented decision | Written async (RFC, design doc, ADR) | Searchable, reviewable, challengeable |
| Quick acknowledgement | Slack/Teams thread | Low friction; short lived |
| Sensitive feedback | 1:1 face-to-face or video | Non-verbal feedback; safety |
| Status update to stakeholders | Email or Confluence page | Non-intrusive; skimmable |
| Incident communication | Dedicated incident channel + status page | Single source of truth; time-stamped |

---

## Writing Frameworks

### The Pyramid Principle (Minto)

Lead with the conclusion. Support it with evidence. All technical writing should follow this shape:

```
Level 1 — Bottom line up front (BLUF):
  "We should migrate from REST polling to WebSockets.
   Current polling costs $12K/month in API Gateway calls and adds 800ms latency."

Level 2 — Supporting arguments (3 max):
  1. Cost: 40M polling calls/day at current scale → $12K/month
  2. Latency: Polling interval = 500ms + 300ms processing overhead
  3. Alternative verified: WebSocket prototype achieves < 50ms with 90% cost reduction

Level 3 — Evidence / appendix:
  → Cost breakdown sheet (linked)
  → Load test results (linked)
  → Migration risk assessment (linked)
```

**Anti-pattern (inverted pyramid)**:
> "We looked at the architecture and noticed polling frequency was high. After running some tests and checking the billing dashboard, and looking at alternatives including long polling and SSE and WebSockets, we think maybe WebSockets could help..."

### BLUF for Slack/Email

Every message that requires a response should state the request in the first sentence:

```
❌ "Hey, I've been looking into the DB performance issues we discussed last week.
    I ran EXPLAIN ANALYZE on the slow queries and found some missing indexes.
    I think we should add three indexes. Let me know what you think."

✅ "Action needed: Approve adding 3 DB indexes before Friday's release.
    Reason: EXPLAIN ANALYZE shows 3 queries doing full table scans (attached).
    Impact: ~80% reduction in P99 latency on the checkout flow.
    Risk: < 5 min migration, tested on staging. No rollback needed."
```

### Design Doc / RFC structure

```markdown
# Title: <What decision is being made>

## Status: Draft | In Review | Accepted | Superseded by [link]

## Problem
One paragraph. What is broken, missing, or won't scale? What is the forcing function (deadline, incident, SLA)?

## Context
What does the reader need to understand to evaluate this? Link don't repeat — reference ADRs, existing systems, etc.

## Proposal
What are you proposing to do? Be specific enough that an engineer could implement it from this section.

## Alternatives considered
| Option | Pros | Cons | Why rejected |
|---|---|---|---|
| Keep current approach | No migration cost | Won't scale past 10M events/day | Violates 6-month growth target |
| Option B | ... | ... | ... |

## Consequences
What changes? What becomes harder? Who is affected? What new operational burden is created?

## Open questions
- [ ] Does team X need to be consulted?
- [ ] What is the rollback plan if production adoption fails?

## Decision
<Fill in after review>
```

### Incident communication template

**Initial (within 5 min of detection)**:
```
[INCIDENT STARTED] SEV-2 — Checkout API elevated error rate
Time: 14:32 UTC
Impact: ~15% of checkout requests returning 500. ~800 users/minute affected.
Current status: Investigating
IC (Incident Commander): @your-name
Next update: 14:45 UTC
```

**Update (every 15 min until resolved)**:
```
[UPDATE 14:45] SEV-2 — Checkout API
Status: Root cause identified — DB connection pool exhausted after 14:20 deploy
Action: Rolled back to v1.42.3 at 14:43
Current: Error rate declining (12% → 4% and falling)
Next update: 15:00 UTC or when resolved
```

**Resolution**:
```
[RESOLVED 14:58] SEV-2 — Checkout API
Duration: 26 minutes
Impact: ~20,800 failed checkout attempts
Root cause: Connection pool size not updated for new DB; exhausted under load
Fix: Rollback to v1.42.3; pool size fix in PR #1287 (ships tomorrow)
Post-mortem: In draft, link by EOD tomorrow
```

---

## Stakeholder Communication

### Audience ladder — tune your message

| Audience | What they need | What they don't need |
|---|---|---|
| **CTO / VP** | Risk, cost, timeline, decision needed | Implementation details, stack choices |
| **Engineering Manager** | Progress, blockers, team impact | Line-by-line code reasoning |
| **Tech Lead peer** | Technical trade-offs, alternatives | Business justification (they trust you) |
| **Product Manager** | User impact, timeline, what's changing | Architecture diagrams |
| **Customer** | What broke, when it's fixed, what you're doing | Internal systems, team names |

### Status update template (weekly async)

```
## Week of [date] — [Team/Project name]

### Done this week
- Shipped feature X (PR #321) — reduces P95 latency from 450ms to 120ms
- Resolved SEV-3 DB migration issue — post-mortem published

### In progress
- Auth service migration (60% complete) — on track for [date]
- Performance profiling of batch job — blocked on staging data access

### Blocked
- [ ] Need sign-off from Security on new OAuth scope — requested [date], awaiting response from @person

### Next week
- Complete auth migration and cut to 5% canary
- Start design doc for event sourcing proposal

### Risks
- Batch job deadline is [date]; if staging access not resolved by [date], we miss it
```

---

## Written Communication Anti-Patterns

| Anti-pattern | Bad example | Better |
|---|---|---|
| **Weasel words** | "We kind of need to maybe look at this" | "We need to fix this before the Q3 launch" |
| **Blame without ownership** | "The pipeline was broken" | "I merged a breaking change to the pipeline; I'm fixing it now" |
| **Ambiguous "we"** | "We should review this" | "I'll review this and send notes by Wednesday. @Alex, can you review the security section?" |
| **Unanswered questions in updates** | "Something is wrong with the DB" | "DB write latency spiked from 5ms to 800ms at 13:40. Investigating; likely cause is missing index on orders table" |
| **Hedging everything** | "This might potentially cause some issues in certain cases" | "This causes a race condition when 2 concurrent requests hit the same session. Reproduced in staging." |

---

## Async Communication Principles

1. **Make it easy to say yes or no**: Provide options, not open-ended requests.
2. **One message, one ask**: Multiple asks in a single message lead to partial responses.
3. **Give context, not just a request**: "Can you review this?" → "Can you review the auth section of this PR by Thursday? I want to merge before the Friday freeze."
4. **State the deadline explicitly**: "Soon" is not a deadline.
5. **Summarise long threads**: If a thread exceeds 10 messages, write a summary at the top.
6. **Don't use Slack for decisions**: Important decisions that need to be searchable belong in documents.

---

## Self-Assessment Checklist

```
Writing
  [ ] My messages lead with the ask or conclusion, not the background
  [ ] I adjust technical depth based on the audience
  [ ] My RFCs and design docs follow a consistent structure
  [ ] I write incident updates that a non-engineer can understand

Meetings & verbal
  [ ] I can explain a technical decision to a VP in 2 minutes
  [ ] I speak clearly about uncertainty ("I don't know, I'll find out by x")
  [ ] I drive meetings to conclude with an action, owner, and deadline
  [ ] I don't dominate discussions — I create space for others

Async
  [ ] I respond to messages within the agreed team SLA
  [ ] I use threads, not channels, for discussions
  [ ] I close open loops explicitly ("Done", "Won't fix — reason", etc.)
  [ ] I summarise long email/Slack threads before adding my reply
```
