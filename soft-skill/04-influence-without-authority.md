# Influence Without Authority

## Context

Most of the important problems engineers need to solve span team or organisational boundaries. You need another team to adopt your API standard, change their deployment process, or prioritise a dependency you need — and you have no authority over them.

Influence without authority is the skill of moving people and organisations toward a goal when you can't order them to move. At senior and staff engineer level, this is a core daily activity.

**Why authority-based influence fails**:
Even when you do have authority, mandating behaviour generates compliance, not commitment. People comply while you're watching and revert when you're not. Influence creates durable change because people agree with the direction.

**The two most common failure modes**:
1. **Being right but unpersuasive**: You have the correct technical answer but no one adopts it. Your RFC is technically excellent and goes nowhere.
2. **Over-engineering the argument**: More data and longer docs actually delay decisions — decision-makers experience them as risk, not clarity.

---

## Building Credibility

Influence is borrowed against a credibility account. Before you can persuade, you need credibility in the space.

**Credibility sources**:
| Source | How to build it | How to lose it |
|---|---|---|
| **Technical competence** | Ship things that work; diagnose incidents correctly; review PRs that improve code | Confidently wrong; hype without substance |
| **Track record** | Commitments kept; follow-through; predictable delivery | Missing deadlines without notice; promising then underdelivering |
| **Intellectual honesty** | Acknowledge uncertainty; admit when you were wrong; update positions with evidence | Defending a position you know is wrong; hiding bad news |
| **Domain knowledge** | Be the person others consult; write the authoritative docs | Having opinions on everything without depth |
| **Relationships** | Known, trusted, approachable; people hear about your reputation before they meet you | Being known as difficult, dismissive, or a bottleneck |

**Minimum credibility requirement**: Before proposing a change that affects another team, deliver at least one thing they asked for without being asked twice.

---

## The Proposal Framework

### Structure for a persuasive technical proposal

```
1. Lead with the problem they recognise — not the solution you want
   "Deployment failures on the shared pipeline are causing 4–6 hours of blocked
   work per week across 8 teams."

2. Make the cost of the status quo explicit
   "At current team size, that's ~32 person-hours/week — roughly one engineer's
   sprint wasted per week."

3. Propose the simplest effective solution
   "If we add a required canary stage with a 5% rollout gate, we catch 90% of
   these failures before they impact the main line. Prototyped in 3 hours."

4. Acknowledge the cost of your proposal
   "This adds ~8 minutes to the pipeline. I've already spoken to the 3 largest
   teams — all consider it a worthwhile trade."

5. Make the ask specific and bounded
   "I'm asking for 30 minutes this week to walk through the prototype with the
   platform team. I'm not asking for a commitment yet."
```

### Common mistakes in proposals
- Leading with your solution before establishing the problem
- Overwhelming with data (3 key numbers, not 30)
- Not acknowledging the cost of your proposal (signals you've thought about the other side)
- Making the ask too large for an initial conversation

---

## Stakeholder Mapping

Before starting any cross-team initiative, map the stakeholders:

```
High interest / High power → MANAGE CLOSELY: key decision-makers; keep in the loop; meet 1:1
High interest / Low power  → KEEP INFORMED: implementers; enthusiastic allies; communicate frequently
Low interest / High power  → KEEP SATISFIED: can block you; brief regularly; never surprise them
Low interest / Low power   → MONITOR: inform at milestones; don't over-invest
```

**Power mapping exercise**:
For each stakeholder, answer:
1. What do they care about most? (their priority, not yours)
2. What would make them say no?
3. Who influences them that you can speak with first?
4. What's the minimum ask that gets you started?

---

## Navigating Organisational Resistance

### The 5 types of resistance and how to respond

| Resistance type | What it sounds like | Response |
|---|---|---|
| **Technical disagreement** | "That approach won't work because…" | Engage the argument; propose a spike to validate |
| **Priority conflict** | "This isn't in our roadmap" | Quantify the cost of not doing it; find the smallest piece they can do |
| **Risk aversion** | "What if this breaks something?" | Demonstrate safety; propose a reversible pilot |
| **Political / territorial** | "We should be the ones building that" | Acknowledge their ownership; offer to build it together |
| **Inertia** | "We've always done it this way" | Find an ally who's frustrated with the current state; show a working alternative |

### "Disagree and commit"

When a decision goes against your recommendation:
- State your disagreement clearly and on record: "I think this is the wrong call because [specific reason]."
- Then commit fully: "That said, I'll support the decision and make it work."
- Don't revisit it after the fact unless new evidence arrives.

What you must NOT do: say nothing during the decision, then undermine the outcome after.

---

## Building Alliances

Technical proposals succeed because people trust the proposer, not because the proposal is optimal. Build alliances before you need them.

**Alliance-building habits**:
- Attend other teams' demos and planning reviews occasionally — just to listen
- Help with something off your critical path when asked (builds goodwill credit)
- Mention others' contributions publicly: "The API design was shaped significantly by @person's input"
- Consult before you propose — people support what they helped build
- Walk proposals past key influencers informally before the formal review

**The pre-flight conversation**:
Before any significant proposal goes to a review, have a 20-minute informal conversation with the 2–3 most influential people:
> "I'm working on a proposal for X. I wanted to get your perspective before I write anything formal. What would the biggest concern be from your team's point of view?"

This achieves:
1. You discover objections while you can still address them
2. They feel consulted and are less likely to be surprised
3. You may hear something that improves the proposal significantly

---

## Influence Upward

Influencing peers is easier than influencing leaders. With leaders, you have less time, higher stakes, and they're evaluating you while you're trying to persuade them.

**Rules for influencing upward**:
1. **Lead with the business problem**, not the technical solution. Leaders care about outcomes, not implementations.
2. **Bring a recommendation, not a question**. "Here's what I think we should do and why" opens a better conversation than "What should we do?"
3. **Quantify the risk of inaction**. Urgency beats importance for getting time on the agenda.
4. **Keep it to 3 options max**. More than 3 options signals you haven't done the analysis.
5. **Make the ask explicit**. "I need your decision on whether to proceed by Friday so we can hit the Q2 deadline."

---

## Self-Assessment Checklist

```
Credibility
  [ ] I have delivered on my last 3 commitments to other teams
  [ ] I acknowledge uncertainty openly ("I don't know; I'll confirm by x")
  [ ] I update my position publicly when I'm wrong
  [ ] I am known as someone who helps before they ask

Proposals
  [ ] My proposals lead with the problem, not my solution
  [ ] I have a named owner for every significant cross-team initiative I'm driving
  [ ] I have pre-flighted key proposals informally before formal review

Relationships
  [ ] I can name the engineers on adjacent teams whose work intersects with mine
  [ ] I have spoken to at least one person in each impacted team before proposing a change
  [ ] I give credit to others in public forums when credit is due

Upward influence
  [ ] I bring recommendations, not questions, to senior leadership
  [ ] I frame technical problems in terms of business outcomes
  [ ] I make my asks explicit: decision needed, by when, why it matters
```
