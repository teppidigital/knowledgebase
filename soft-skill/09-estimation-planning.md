# Estimation and Planning

## Context

Estimation is not prediction. It is the process of communicating uncertainty in a way that enables good decisions. The goal is not to be right — it is to be usefully wrong by giving stakeholders enough information to plan, prioritise, and set expectations.

Poor estimates damage trust not because they are wrong, but because the communication around them implies a certainty that doesn't exist.

**Failure modes**:
| Pattern | What it looks like | Cost |
|---|---|---|
| **Single-point estimates** | "It'll take 2 weeks" with no range | When it takes 4, trust collapses |
| **Padding without acknowledgment** | Secretly doubling estimates to "be safe" | Wastes time; erodes team capacity visibility |
| **Scope drift** | Estimate doesn't track scope changes | Original estimate becomes meaningless mid-delivery |
| **Anchoring** | First number said in a meeting becomes the commitment | Managers can anchor you down; be aware of who speaks first |
| **Estimation by confidence** | Estimate what you think they want to hear | Short-term harmony; long-term missed deadlines |

---

## Estimate Types and When to Use Each

| Method | What it produces | Best for | Weakness |
|---|---|---|---|
| **Story points** | Relative complexity score | Velocity tracking over sprints | Meaningless to external stakeholders |
| **T-shirt sizing** | XS/S/M/L/XL | Early roadmap discussion; investment sizing | Imprecise; can mask real complexity |
| **Time-based (range)** | "3–5 days" | Scheduling; cross-team coordination | Risk of being held to the low end |
| **Three-point estimation** | Optimistic / Most likely / Pessimistic | High-stakes planning; external commitments | Requires more thinking upfront |
| **#NoEstimates / throughput** | Cycle time and lead time from history | Mature teams with stable flow | Requires historical data; doesn't work well for new work types |

---

## Three-Point Estimation

Three-point estimation forces you to think about the range of outcomes, not just the single most likely scenario.

```
OPTIMISTIC (O): Everything goes well; no blockers; dependencies arrive on time
MOST LIKELY (M): Normal level of friction; some rework; minor dependency slips  
PESSIMISTIC (P): One major unknown turns out to be hard; a dependency is late; rework required

EXPECTED VALUE = (O + 4M + P) / 6    [weighted toward Most Likely]
STANDARD DEVIATION = (P - O) / 6     [measure of uncertainty]
```

**Example**:
- Optimistic: 3 days
- Most likely: 6 days
- Pessimistic: 15 days (unknown auth dependency resolved badly)

```
Expected = (3 + 4×6 + 15) / 6 = (3 + 24 + 15) / 6 = 42 / 6 = 7 days
StdDev   = (15 - 3) / 6 = 2 days
```

**Communication**: "My best estimate is 7 days. With a tailwind, we could hit 5. With a headwind (auth dependencies not resolved), we could be at 15. I'll have a clearer fix by end of next week once I've done a spike."

---

## The Cone of Uncertainty

Estimates improve as more is known. The cone of uncertainty is the model that explains why early estimates are wide and get more precise over time.

```
          Wide
         ╱────╲
        ╱      ╲     ← Project start: 4× variance
       ╱        ╲
      ╱ High-    ╲
     ╱  level    ╲
    ╱   design   ╲    ← Requirements baseline: 2× variance
   ╱               ╲
  ╱  Detailed       ╲  ← Architecture: 1.5× variance
 ╱   design          ╲
╱─────────────────────╲  ← Implementation started: 1.1× variance
        Narrow
```

**Implication for communication**: Never give a precise estimate before enough is known. The correct answer to "how long will this take?" during a discovery phase is: "I can give you a range today: 2–8 weeks. I'll narrow that to ±20% after the initial spike (2 days of work)."

Stakeholders who push back on ranges are asking you to commit to certainty you don't have. The correct response is to explain what would need to be true for the estimate to narrow.

---

## Communicating Estimates

**Estimate communication rules**:

1. **Always give a range, not a single number** for any estimate over 1 day
2. **State your assumptions** — the estimate is conditional on those assumptions
3. **Name the unknowns** — what you don't know and when you'll know it
4. **Give a decision point** — "By [date] I'll have done a spike and can give you a ±20% estimate"
5. **Don't accept anchor pressure** — if someone says "we need it in 2 weeks", respond: "I can tell you what we'd need to cut to make that work. Can we talk about scope?"

**Estimate template**:
```
My estimate for [feature/task] is [range] under the following assumptions:
- [Assumption 1]
- [Assumption 2]

The biggest uncertainties are:
- [Unknown 1] — I'll have a clearer view by [date]
- [Unknown 2] — requires spike of [duration]

If we hit the pessimistic case, the most likely cause would be [risk factor].
```

---

## Managing Scope Creep

Scope creep is usually not malicious — it is well-intentioned addition. The problem is that each addition is small and feels reasonable; accumulated scope creep causes the original estimate to become meaningless.

**Scope creep patterns**:
- "While you're in there, can you also…"
- Undocumented acceptance criteria discovered at review
- Stakeholder sees demo and requests changes to completed work
- Dependencies delivered in a different form than expected

**Handling scope addition requests**:
```
"Yes, we can add [thing]. Adding that will move the estimate
from [X] to [Y] and push the delivery from [date] to [date].
Do you want to add it, or should I create a follow-on ticket?"
```

This is not resistance — it is communication. Every addition deserves an explicit trade-off conversation.

**Scope change log** (maintain during any delivery over 2 weeks):

| Date | Change | Requested by | Impact | Decision |
|---|---|---|---|---|
| 2024-01-15 | Add OAuth support | PM | +3 days | Accepted; delivery moved to Feb 2 |
| 2024-01-22 | Remove rate limiting | Tech lead | -1 day | Accepted; deferred to v2 |

---

## Retrospective: Tracking Estimation Accuracy

If you never look back at your estimates, you cannot improve them.

**Estimation retrospective (quarterly or after each major delivery)**:

1. **Collect the data**: For each item, record: estimated range, actual time, assumptions made
2. **Identify the bias**: Were estimates consistently too optimistic? By how much? (Most teams are optimistic by 2×)
3. **Identify the common surprises**: What unknowns repeatedly materialised? Can any of them be systematised?
4. **Adjust the system**: If team is consistently 2× optimistic, adjust velocity, add buffer, or change estimation method

**Simple tracking table**:
| Feature | Estimate | Actual | Ratio | Main cause of variance |
|---|---|---|---|---|
| User auth | 5d | 9d | 1.8× | OAuth provider docs outdated |
| Notifications | 3d | 2.5d | 0.8× | Library did more than expected |
| Export CSV | 2d | 6d | 3× | Data volume edge cases |

---

## Self-Assessment Checklist

```
Estimation quality
  [ ] I give ranges, not single points, for estimates over 1 day
  [ ] My estimates explicitly state the assumptions they depend on
  [ ] I name the top 2 unknowns in every estimate and when I'll know more
  [ ] I use three-point estimation for high-stakes work

Communication
  [ ] I notify stakeholders proactively when an estimate changes (not at delivery)
  [ ] I make scope changes explicit: name the addition, name the trade-off
  [ ] I have a scope change log for deliveries longer than 2 weeks

Retrospective
  [ ] I review my estimation accuracy after each significant delivery
  [ ] I know my team's typical optimism bias (ratio of actual to estimate)
  [ ] I adjust estimates based on what I've learned, not just gut feel
```
