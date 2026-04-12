# Giving and Receiving Feedback

## Context

Feedback is the primary mechanism for professional growth. Without it, engineers repeat the same patterns for years and are blindsided during performance reviews. With it, improvement is rapid and continuous.

The two most common failure modes:
1. **Feedback withheld**: "It's not worth the awkwardness" — the person never improves and eventually fails in a higher-stakes situation.
2. **Feedback delivered poorly**: Vague, overly personal, or timed badly — the receiver becomes defensive and nothing changes.

**Feedback is not the same as criticism**. Criticism evaluates the person. Feedback describes observable behaviour and its impact, then suggests an alternative.

---

## The SBI Model (Situation, Behaviour, Impact)

SBI is the most practical feedback framework for engineers. It keeps feedback specific, non-judgmental, and actionable.

| Component | Definition | Example |
|---|---|---|
| **Situation** | When and where did it happen? (specific, not general) | "In yesterday's architecture review…" |
| **Behaviour** | What exactly did the person do or say? (observable, not interpreted) | "…you interrupted Ana three times before she finished her point…" |
| **Impact** | What was the effect on you, the team, or the outcome? | "…which made her disengage for the rest of the session, and we didn't hear her proposal." |

**Full example — constructive**:
> "In yesterday's architecture review, you interrupted Ana three times before she finished her point. As a result, she disengaged for the rest of the session and we didn't hear her proposal. I'd encourage you to wait for a pause before adding your perspective."

**Full example — positive**:
> "In today's incident call, you calmly summarised the situation every 15 minutes for stakeholders who weren't technical. That kept panic low and let the engineers focus. That was really effective."

### SBI applied to code review

Code review is feedback — apply the same principles:

```
❌ "This is wrong."
❌ "Why would you do it this way?"
❌ "We don't write code like this."

✅ "This approach works, and I want to flag one concern:
    in the loop on line 42, we're calling findById() on every iteration.
    With 10K records that's 10K DB calls. Consider extracting this to a
    batch query before the loop. [link to docs]"

✅ (positive) "Nice use of a discriminated union here — makes the exhaustive
    check in the switch explicit. Good pattern to keep."
```

---

## Radical Candour (Kim Scott)

A complementary framework to SBI. It describes feedback quality along two axes:

```
                     Care Personally
                          ↑
                          │
   Ruinous Kindness       │        Radical Candour
   (nice but not honest;  │        (honest AND caring;
    problems fester)      │         the goal)
                          │
 ──────────────────────────────────────────────────→
                          │              Challenge Directly
   Manipulative           │
   Insincerity            │        Obnoxious Aggression
   (neither caring nor    │        (honest but not caring;
    honest; the worst)    │         burns trust)
                          │
```

**Radical Candour in practice**:
- Praise publicly when warranted; deliver criticism privately.
- Give feedback as soon as possible after the event — not at the end of the quarter.
- Deliver it in person or over video — not Slack.
- Make it clear you're giving feedback *because* you think the person can grow, not to judge them.

---

## Receiving Feedback

Receiving feedback well is a skill as important as giving it. Most defensive reactions are physiological — the brain treats criticism as a threat.

### The ACCEPT model

| Step | In practice |
|---|---|
| **Acknowledge** | "Thank you for telling me this." (Don't immediately justify.) |
| **Clarify** | "Can you give me a specific example?" (SBI applied in reverse.) |
| **Consider** | Let it sit. Don't respond to the substance immediately. |
| **Evaluate** | Is it accurate? From a trusted source? Pattern or one-off? |
| **Plan** | What will you do differently? |
| **Thank** | Follow up: "I've been thinking about what you said — here's what I'm changing." |

### Signals that you're being defensive

- Immediately providing counter-examples
- Explaining your reasoning before asking a clarifying question
- Going silent and withdrawn
- Seeking validation from a third party before considering the feedback

**The test**: After receiving feedback, can you accurately repeat back what the person said — without your interpretation of their intent? If not, you haven't listened yet.

---

## Performance Reviews

### Writing your own review

A self-review that lands well follows the same principles as a design doc — evidence-based, specific, tied to impact:

```
❌ "I worked on the payments team and contributed to multiple projects."

✅ "Reduced checkout P95 latency from 650ms to 120ms by introducing Redis
    caching (PR #312). This directly resolved the SLA breach flagged in Q1
    and avoided estimated £40K/year in SLA credits.
    Mentored two junior engineers on the observability tooling — both are
    now independently writing and responding to alerts."
```

**Framework: Situation → Action → Result (SAR)**:
- **Situation**: What was the context or problem?
- **Action**: What specifically did you do?
- **Result**: What measurable outcome followed?

### Common self-review mistakes
- Describing activities, not impact: "I attended design reviews" instead of "I challenged the proposed pattern in 3 reviews, preventing 2 architectural decisions that would have required rework within 6 months"
- Listing skills instead of evidence: "Strong communication skills" vs. "Wrote the incident post-mortem template adopted by all 4 teams"
- Underselling deliberately: False modesty harms your trajectory and wastes your manager's political capital

### Writing peer reviews

**Useful peer review**:
- Specific and behavioural (SBI)
- Includes at least one growth area — a review with only positives is not trusted
- Calibrated: Does the level of praise match what hiring this person at a higher level would look like?

**Template**:
```markdown
## What [Name] does well (2–3 specific examples)

## Where [Name] has the most growth opportunity

## What [Name] would need to demonstrate to operate at the next level

## One thing that would have improved our collaboration
```

---

## Feedback Culture Norms

Agree these as a team explicitly — don't assume them:

| Norm | Why it matters |
|---|---|
| Feedback is given with the intent to help, not judge | Creates safety to receive it openly |
| Positive feedback is delivered publicly and promptly | Reinforces the right behaviour; models what good looks like |
| Critical feedback is private; praise is public | Protects dignity; avoids public embarrassment |
| Feedback includes a suggested alternative | "Stop doing X" is not actionable without "try Y instead" |
| Feedback on code is about the code, not the person | "This function is hard to follow" not "You write unclear code" |

---

## Self-Assessment Checklist

```
Giving feedback
  [ ] I give feedback within 48 hours of the triggering event, not at review time
  [ ] I use SBI structure: I can state the situation, behaviour, and impact specifically
  [ ] My positive feedback is as specific as my critical feedback
  [ ] I offer a suggested alternative, not just what went wrong
  [ ] I deliver critical feedback 1:1, not in group settings

Receiving feedback
  [ ] I thank the person before responding or justifying
  [ ] I ask for an example if the feedback is vague
  [ ] I wait at least 24 hours before deciding whether feedback is valid
  [ ] I follow up with the person to say what I've changed

Culture
  [ ] I normalise feedback by giving it regularly, not just at reviews
  [ ] My code review comments are about the code, not the author
  [ ] I have explicitly asked "how can I give you better feedback?" to at least one report
```
