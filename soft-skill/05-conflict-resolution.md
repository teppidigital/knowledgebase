# Conflict Resolution

## Context

Conflict in engineering teams is not a failure — it is evidence that people care about outcomes. Managed well, productive conflict generates better decisions; mismanaged, it damages trust and drives good people to disengage or leave.

The goal is not the absence of conflict. It is the capacity to surface disagreements early, work through them constructively, and reach a conclusion — even when full consensus isn't possible.

**Failure modes**:
| Pattern | What it looks like | Cost |
|---|---|---|
| **Conflict avoidance** | Issues never raised directly; silently shipped the wrong thing | Resentment builds; problems go underground |
| **Escalating to position** | "I've been here longer" / "I'm the lead" | Silences better ideas; people stop speaking up |
| **Technical tribalism** | Team/tech stack loyalty, not argument quality | "Not invented here" syndrome; duplicated work |
| **Passive resistance** | Agreeing in the meeting, then not delivering | Trust erodes; becomes a management problem |
| **Personal escalation** | Criticism of the person, not the decision | Psychological safety damaged; legal risk |

---

## Productive vs Destructive Disagreement

**Productive disagreement addresses the idea**. It is specific, considers tradeoffs, references shared goals, and ends with a decision.

**Destructive disagreement addresses the person**, their intent, or their character. It is non-specific ("this approach is just wrong"), comparative ("the old way was better"), or silent (expressed through inaction or gossip).

**Test: can you write the other person's best argument?**
If you cannot articulate the strongest version of the opposing view, you have not understood it yet. Steelman the other side before responding.

---

## De-escalation Language

When a conversation is getting heated, use neutral language to slow it down and return to shared ground.

**De-escalation phrases**:

| Situation | Instead of | Try |
|---|---|---|
| Disagreement on approach | "That won't work" | "What problem is that approach solving? I want to make sure I'm understanding the tradeoff" |
| Feeling dismissed | "You're not listening" | "I want to make sure I'm communicating clearly — can I restate what I think the concern is?" |
| Repeated disagreement | "We keep going in circles" | "We've covered the main arguments. Can we agree to time-box this and decide, even if we don't fully agree?" |
| Feeling attacked | "That's a personal attack" | "I'd like to keep this focused on the design decision. What specifically would you change?" |
| High emotion | "Calm down" (never say this) | "Let's take 10 minutes. I want to get this right, not fast." |

---

## Code Review Conflict

Code review is the most common site of interpersonal conflict in engineering teams. It is technical in form but often relational in substance.

**Root causes of code review conflict**:
- Reviewer assumes bad intent ("why would anyone do this?")
- Author feels attacked ("this person hates everything I write")
- No agreed team standards to reference
- Power imblance (senior reviewing junior)

**Principles for review that don't create conflict**:

| Principle | Example |
|---|---|
| Comment on the code, not the author | "This function has 3 responsibilities" not "You made this too complex" |
| Assume positive intent explicitly | "I think what you were going for here is X — one option is…" |
| Distinguish blocking vs non-blocking | Label: `[blocking]`, `[suggestion]`, `[nit]`, `[question]` |
| Ask questions before making statements | "What was the reason for the mutex here?" before "This mutex is wrong" |
| Acknowledge tradeoffs, don't just reject | "Both approaches have merit; given our latency target, I think X is safer" |

---

## Disagree and Commit

When a decision has been made and you disagree:

1. **Record your dissent clearly**: Write it in the decision doc or the relevant thread. This is not to win — it is to ensure the alternative was considered and to create a record if you need to revisit later.
2. **Stop the advocacy**: Once the decision is made, relitigating it privately or subtly undermining it through delayed action is a violation of team trust.
3. **Commit fully**: Don't half-heartedly implement a decision. If you commit, commit. If you cannot commit (ethical concern, technical impossibility), raise that explicitly at the moment of decision.

**The language of disagreeing and committing**:
> "I still think [alternative] would be the better choice because [reason], and I want that on the record. That said, I'll implement [decision] and make it work as well as I can. If we hit [specific risk I identified], I'd like us to review the decision at that point."

---

## Conflict Between People (Not Ideas)

When the conflict has become personal — about a specific relationship rather than a decision:

**Step 1: Private conversation first**
Before escalating, have a direct 1:1. Most relationship conflict is caused by a misread signal or a misunderstood comment. Ask: "I noticed something in our last interaction and I wanted to check in directly — is there something I've done that affected our working relationship?"

**Step 2: Name the pattern, not the instance**
Don't bring a catalogue of incidents. Identify the pattern: "I've noticed that in meetings, when I raise a concern, the conversation moves quickly past it. I'm not sure if that's intentional, but it's affecting how comfortable I feel contributing."

**Step 3: Escalate to a third party (manager/EM) only when**:
- You've had the direct conversation and it didn't resolve
- The relationship is too damaged for direct conversation
- There is a power imbalance that makes direct conversation unsafe
- There is a policy violation (harassment, discrimination)

**Step 4: When you're the manager being escalated to**:
- Don't take sides in the first conversation — listen to both stories separately
- Distinguish the relationship problem from the performance problem
- Facilitate a structured conversation if both parties are willing
- Be willing to make a decision if the parties cannot resolve it themselves

---

## Mediation Patterns

When facilitating a conflict between others:

**Structured conversation format**:
1. Set the ground rules: "We're here to reach a decision, not to establish who was right."
2. Each person states their view without interruption (5 minutes each max).
3. Each person reflects back the other's view: "What I heard you say is…"
4. Identify shared goals: "Both of you want [X]. Where you disagree is on [Y]."
5. Generate options together rather than defending positions.
6. Make a decision — even if it's "we'll try A for 4 weeks and revisit."

---

## Self-Assessment Checklist

```
Productive conflict
  [ ] When I disagree, I can state the other person's best argument before stating mine
  [ ] I use specific examples (the design doc, the PR comment) not generalisations
  [ ] I separate disagreement with an idea from distrust of a person

Code review
  [ ] My review comments distinguish blocking issues from suggestions / nits
  [ ] I ask questions before making statements about unfamiliar code
  [ ] I comment on the code, not the author

Disagreement and commitment
  [ ] I record dissent clearly and then commit fully
  [ ] I do not relitigate decided questions in private
  [ ] If I cannot commit, I raise that at decision time, not after

Mediation
  [ ] I don't take sides before hearing both perspectives
  [ ] I can facilitate a structured conversation with two people in conflict
  [ ] I know when to escalate and when escalation would make things worse
```
