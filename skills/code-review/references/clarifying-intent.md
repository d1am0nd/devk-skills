# Clarifying intent — the Q&A loop

You have an intent hypothesis and possibly a few suspicious spots in mind. The point of this step is for *you* to come out the other side knowing exactly what the change is for and where the human wants you to focus.

## Format — one question at a time

```
**Q1: <single question, one short sentence>**

a) <concrete answer you think is most likely>
b) <concrete alternative answer>
c) Let me explain.
```

Rules:

- One question per turn. No stacked "also..." follow-ups.
- a) and b) must be plausible, concrete answers — not strawmen. The human should be able to pick one without explanation in most cases.
- c) is always "let me explain". Never replace it with a third concrete option; the human always needs an escape hatch to redirect you.
- Keep options short. One line each is ideal. No multi-paragraph framing of an option.
- Don't justify the question at length. A one-liner of context above the Q is fine; a paragraph isn't.

## What to ask first

Ask the question whose answer changes the most about your review. Usually that means one of:

- **Intent ambiguity** — "Is this change about <X> or <Y>?" — when you have two genuinely plausible intent hypotheses.
- **Scope/priority** — "What's the priority here: correctness, performance, or readability?" — when the change is broad and review focus matters.
- **A specific suspicious spot** — "Is `<file:line>` intentional or a mistake?" — when one spot looks off and the answer would change whether you investigate it deeply.

## What NOT to ask

- Anything you can answer by reading more diff. Read first.
- "What does this code do?" — your job is to figure that out, not ask.
- Multi-part questions. Split them or pick the most important.
- Filler. If you have nothing meaningful to ask after the first answer, *say so* and move on.
- Style preferences ("should I look at naming?"). The standards enforcer handles that anyway.

## After each answer — reconsider

The first answer often kills 2–3 of your planned follow-ups. After reading the human's answer, mentally re-rank what's still worth asking. Most of the time, the second question is much smaller than the first — or doesn't exist.

A common shape:

- Q1 confirms the high-level intent.
- Q2 (sometimes) clarifies one specific suspicious spot.
- Q3 (rarely) confirms a constraint for the review (e.g., "should I assume backward compat matters?").

Two questions is normal. Four is too many. If you reach for a fourth, you're stalling — go review.

## Exit signals

You have enough when:

- You can state the change's intent in one sentence and the human would agree.
- You know which suspicious spots (if any) the human wants you to dig into vs. accept as intentional.
- You know what *kind* of bugs would matter most for this change (correctness? security? perf? UX?).

When done, say something like: *"Got it — kicking off the deep review now. Should be a couple of minutes."* Then move to step 6.

## Edge cases

- **Human's first message already states intent clearly.** Skip Q1. Confirm with one short statement: *"Sounds like the goal is X — moving to deep review."* Only ask if you genuinely have a doubt.
- **Human picks (c) and explains in a way that contradicts your hypothesis.** Update your intent statement and proceed. Don't re-ask just to verify; trust the human.
- **Human says "just review it, you figure it out".** Accept, lock in your hypothesis as the working intent, and proceed. Note in the final report that you reviewed against an unconfirmed intent.
- **Human says "I'm reviewing someone else's PR, I'm not sure of the intent either".** Treat the commit messages and PR description as the intent source. Ask only if those leave a real ambiguity. Mention in the final report that intent came from the commits, not the human.
