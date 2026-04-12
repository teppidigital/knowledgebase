# Building Team Culture

## Context

Culture is not the values on the wall. It is the sum of what behaviours are rewarded, tolerated, and punished — and what people do when no one is watching. Culture is built or eroded by hundreds of small decisions every day.

Team culture is not the responsibility of the engineering manager alone. Every team member, especially senior engineers and tech leads, shapes it through their daily actions.

**What weak culture produces**:
- Attrition concentrated among your best performers (they have options)
- Underperformance that stays hidden because no one feels safe raising it
- Knowledge hoarding and blame cultures
- Long onboarding times because culture is implicit and unwritten
- Poor incident response because people are afraid to speak up or take ownership

---

## Psychological Safety

Amy Edmondson's research on psychological safety is the strongest evidence-based model for high-performing teams. Psychological safety is the belief that you will not be punished for speaking up — for questions, mistakes, ideas, or concerns.

**Psychological safety ≠ comfort**: Safe teams are not conflict-free or challenge-free. They are teams where people feel safe to engage in productive conflict.

**7 factors that indicate psychological safety**:
1. Asking basic questions without fear of judgment
2. Admitting mistakes without fear of blame
3. Raising concerns without being seen as negative or disloyal
4. Proposing unconventional ideas without ridicule
5. Critiquing processes and decisions without fear of reprisal
6. Being honest about uncertainty ("I don't know")
7. Seeking help without feeling like a burden

**What leaders and senior engineers can do**:

| Behaviour | Why it matters |
|---|---|
| Ask questions publicly — including "I don't know" | Models that uncertainty is acceptable |
| Acknowledge your own mistakes in public | Most powerful safety signal available to senior people |
| Thank people for raising concerns | Reward the raising, not just the quality of the concern |
| React to bad news with curiosity, not blame | "What happened?" not "Who did this?" |
| Protect people who speak up from backlash | Safety is only real if it's enforced when tested |
| Invite dissent explicitly | "What am I missing? What might go wrong here?" |

---

## Running Effective Retrospectives

The retrospective is the primary mechanism for team culture improvement. A low-quality retro is worse than no retro — it signals that feedback doesn't lead to change.

**What makes a retro fail**:
- Same action items every sprint, never completed
- Only the loudest voices contribute
- Runs over time on venting; no decisions reached
- Facilitator also the team lead (people soften feedback)
- Nothing changes between retros

**Retro formats**:

| Format | Best for | Structure |
|---|---|---|
| **Start / Stop / Continue** | Teams new to retros; simple and clear | 3 columns; vote on top items |
| **4 Ls** (Liked, Learned, Lacked, Longed for) | Deeper exploration; captures learning | 4 columns; discuss themes |
| **Sailboat** (wind / anchor / sun / rocks) | Systems thinking; seeing what's slowing or helping | Visual metaphor; group discussion |
| **Timeline retro** | After a major release or incident | Map events on a timeline; identify patterns |
| **DAKI** (Drop / Add / Keep / Improve) | Specific practice changes needed | Explicit about what to stop, not just what went wrong |

**High-quality retro facilitation**:
1. Set the context: "This is a blameless space. We're looking at systems and processes, not people."
2. Timeboxed individual reflection before group discussion (prevents anchoring)
3. Dot-vote to surface shared themes rather than loudest voice
4. Every action item: named owner + deadline. No owner = no action.
5. Start next retro by reviewing previous action items first.

---

## Team Charter

A team charter makes the implicit explicit. It is most valuable when a team forms, changes significantly, or has recurring friction.

**Team charter template**:
```markdown
## [Team Name] Working Agreement

### Our mission
[One sentence — what does this team exist to do?]

### What we value
[3–5 values specific and actionable enough to test behaviour against]

### How we communicate
- Async-first for: [categories of communication]
- Sync for: [categories of communication]
- Response time expectations: Slack: [X hours]; PagerDuty: [Y minutes]
- Meeting norms: [agenda required / decisions documented / etc.]

### How we make decisions
- Minor technical decisions: [who decides]
- Major architectural decisions: [process, e.g. RFC with 48h review]
- Team process decisions: [consensus / lead decides / etc.]

### How we handle conflict
[1–2 sentences on expected escalation path]

### How we balance deep work and collaboration
- Quiet hours: [e.g. 9–12 no meetings]
- On-call rotation: [how it works]
- Review SLA: [PRs reviewed within X hours]

### How we grow
- Learning time: [dedicated time per sprint/week]
- Knowledge sharing: [format and cadence]
- Feedback: [retro cadence, 1:1 cadence]

### What good looks like here
[Describe the 3 behaviours that best exemplify this team's culture]
```

---

## Celebrating Wins (Without It Feeling Hollow)

Recognition that is vague, delayed, or proportionate to the giver's relationship with the recipient does not build culture — it creates cynicism.

**Principles for meaningful recognition**:
1. **Specific**: Name what they did, not just that they did "great work"
2. **Timely**: The closer to the behaviour, the stronger the reinforcement
3. **Public where appropriate**: Some people prefer private acknowledgment; ask first
4. **Proportionate**: Shipping a major project deserves more than passing a code review
5. **Peer-driven**: Recognition from peers often lands harder than from managers

**Recognition that works**:
> "I want to call out [name]'s work on the incident last week. They noticed the metric trending 30 minutes before the alert would have fired, wrote the fix in 45 minutes, and had the post-mortem doc in the shared folder within 24 hours. That kind of instinct and discipline is exactly what makes this team work."

**Team rituals for recognising wins**:
- Weekly "shoutouts" at the end of standup or team sync
- A dedicated Slack channel (#kudos, #wins) where anyone can post
- Sprint reviews that open with "what did we ship and who did it"
- Explicit crediting in written communications to leadership

---

## Remote and Distributed Teams

Distributed teams need extra intentionality around the informal behaviours that happen naturally in co-located teams.

**Distributed culture practices**:

| Problem | Solution |
|---|---|
| Invisible relationships | Structured social time (virtual coffee, non-work channel) |
| Timezone isolation | Async-first; rotating meeting times; record synchronously |
| Information silos | Over-document decisions; everything searchable; no Slack-only agreements |
| Onboarding coldness | Pair-programming culture; buddy system; first-week schedule |
| Burnout invisible to others | Team norms on working hours; explicit "no response expected after X" |
| Contribution not visible | Written summaries of async contributions; PRs reviewed publicly |

---

## Self-Assessment Checklist

```
Psychological safety
  [ ] I publicly acknowledge mistakes and uncertainty (modelling the behaviour)
  [ ] When someone raises a concern, my first response is curiosity, not defence
  [ ] I actively invite dissent in design and planning discussions
  [ ] People on my team raise concerns to me before they raise them elsewhere

Retrospectives
  [ ] Retros have rotating facilitators (not just the manager)
  [ ] Action items have a named owner and a date — not "the team will"
  [ ] I start each retro by reviewing last retro's actions
  [ ] People on the team speak in proportion to their share of the team, not their seniority

Team charter and norms
  [ ] My team has written communication and decision-making norms
  [ ] New team members receive the working agreement in their first week
  [ ] The team charter is reviewed at least annually

Recognition
  [ ] My recognition is specific and timely, not generic
  [ ] I recognise contributions from quieter team members, not just the loudest
  [ ] The team has at least one regular ritual for celebrating wins
```
