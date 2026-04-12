# Prioritisation and Time Management

## Context

Engineering time is the most expensive and least recoverable resource on a team. Poor prioritisation hurts in two distinct ways:
1. Wrong work gets done: correct execution of the wrong priority.
2. Right work gets delayed: the right thing identified but never resourced.

The goal is not doing more things. It is doing the right things and protecting the time required to do them well.

**Failure modes**:
| Pattern | What it looks like | Cost |
|---|---|---|
| **FIFO prioritisation** | Most recent request becomes top priority | Strategic work never happens |
| **Urgency bias** | Everything feels urgent; nothing is treated as important | Constantly reactive; technical debt compounds |
| **Invisible work** | Interruptions, reviews, toil never counted in capacity | Commitments missed; people burn out |
| **Inability to say no** | Every request accepted; backlog grows without bound | Half-finished work; trust damaged when things slip |
| **Planning without protecting** | Priorities set, but time not blocked | Important but not urgent work consistently bumped |

---

## The Eisenhower Matrix

Categorise every piece of work by urgency and importance:

```
                  URGENT              NOT URGENT
            ┌─────────────────────┬─────────────────────┐
IMPORTANT   │  DO NOW             │  SCHEDULE           │
            │  Incidents          │  Architecture work  │
            │  Production bugs    │  Tech debt paydown  │
            │  On-fire deadlines  │  Documentation      │
            ├─────────────────────┼─────────────────────┤
NOT         │  DELEGATE           │  ELIMINATE          │
IMPORTANT   │  Interruptions      │  Unnecessary reports│
            │  Most meetings      │  Low-value admin    │
            │  @-mentions         │  Meetings with no   │
            │                     │  agenda             │
            └─────────────────────┴─────────────────────┘
```

**Quadrant II is the leverage zone**: important, not urgent work is where long-term outcomes are shaped. The single greatest prioritisation skill is growing your time in Quadrant II by reducing Quadrant I crises (which are often caused by under-investment in Quadrant II) and eliminating Quadrant III/IV noise.

---

## Deep Work vs Shallow Work

**Deep work**: cognitively demanding, creates new value, requires uninterrupted concentration (architecture, complex debugging, writing, designing systems).

**Shallow work**: logistical, low cognitive load, replicable (responding to Slack, attending status meetings, admin tasks).

Both are necessary. The problem is that shallow work is interruptible and immediate, so it crowds out deep work unless deep work is actively protected.

**Daily time audit**:
Track your actual time for one week by category. Most engineers are shocked to discover they have fewer than 90 minutes per day of real deep work. The rest is context-switching and shallow tasks.

**Protecting deep work**:

| Strategy | Implementation |
|---|---|
| Time blocking | Calendar block 2–3 hour uninterrupted sessions 3–4× per week; treat like external meetings |
| Focus modes | Phone on DND; Slack status to "Deep work until 14:00" |
| Office hours | Announce specific times you're available for ad-hoc questions |
| Async-first defaults | Prefer async for non-urgent questions; don't expect immediate responses |
| Meeting-free windows | Agree as a team on no-meeting mornings (e.g., Mon/Wed before lunch) |

---

## Managing Interruptions

Interruptions compound. A 2-minute question costs 20+ minutes of recovery time due to context switching.

**Interrupt classification**:
| Interrupt type | Appropriate response |
|---|---|
| Production incident | Stop; respond immediately |
| Colleague stuck on a blocker | "I'm mid-task; can we do 15 minutes at [time]?" |
| Non-urgent Slack question | Reply when you surface, not in real time |
| Meeting request without agenda | "Can you send me what decision or outcome this meeting is for?" |
| Escalated to you via email | Check email 2× daily, not continuously |

**Saying no effectively**:

The goal is to decline the work, not the person. Give a reason, and offer an alternative where possible.

| Situation | Script |
|---|---|
| Too much on the plate | "I want to take this on properly. Right now I'm working on X until [date]. Can this wait, or should we talk about what comes off my plate?" |
| Wrong owner | "I think [person/team] is better placed for this. I'm happy to introduce you." |
| Out of scope | "This isn't in our current sprint focus. Can you create a ticket and I'll make sure it gets assessed at next planning?" |
| Underspecified request | "I want to help. Before I start, I need to understand [specific thing]. Can you get me that first?" |

---

## Energy Management

Time management without energy management is incomplete. You have 8 hours but not 8 hours of equal cognitive capacity.

**Simple energy model**:
- **Peak hours** (typically morning for most): Deep, creative, cognitively demanding work
- **Trough** (early afternoon): Administrative tasks, routine reviews, Slack replies
- **Recovery** (late afternoon): Collaborative work, meetings, lightweight reviews

Scheduling matters more than volume. One hour in peak hours is worth 3 in a trough.

---

## Weekly Planning Ritual

A consistent weekly review prevents reactive management and keeps strategic priorities visible.

**30-minute weekly review (recommend: Friday afternoon or Monday morning)**:

```
1. Capture (5 min)
   - Review notes, Slack, email for anything uncaptured
   - Add to backlog; don't keep anything in your head

2. Review last week (5 min)
   - What did I complete? What slipped? Why?
   - Did I actually do the high-priority items I planned?

3. Set this week's priorities (10 min)
   - Name 3 outcomes that would make this week a success
   - These are not task lists — they are results: "Architecture doc reviewed and signed off"
   - Check they match the team/product priorities

4. Plan the calendar (5 min)
   - Block deep work for each of the 3 priorities
   - Identify any meeting-heavy days and reschedule deep work accordingly

5. Identify blockers (5 min)
   - What could prevent the 3 outcomes?
   - Who do you need to speak with?
   - What can you do today to unblock next week?
```

---

## Estimation and Capacity

Work planned without accounting for actual capacity will always slip.

**Realistic capacity rule of thumb**:
A typical 5-day week has about 3 days of available engineering capacity after accounting for:
- Daily standups and team ceremonies (~2–3 hrs/week)
- Unplanned interruptions and support (~4–6 hrs/week)
- Reviews, async communication (~3 hrs/week)
- Admin and context-switching (~2 hrs/week)

**Total available = 5 days - ~15 hrs overhead ≈ 2.5–3 days net engineering time**

Never commit to 5 days of work in a 5-day sprint without acknowledging this.

---

## Self-Assessment Checklist

```
Prioritisation
  [ ] I can name my top 3 priorities for this week without looking at a list
  [ ] I know which of my current work is Quadrant II (important, not urgent)
  [ ] I regularly push back on requests that don't match team priorities

Time protection
  [ ] I have blocked deep work time on my calendar this week
  [ ] I have communicated availability hours to reduce reactive interruptions
  [ ] I check Slack/email on a schedule, not continuously

Saying no
  [ ] I can decline work with a reason and alternative without guilt
  [ ] I have had at least one conversation this month about what gets deprioritised

Weekly review
  [ ] I have a consistent weekly review ritual (even 20 minutes counts)
  [ ] Last week's planned priorities matched what I actually worked on
```
