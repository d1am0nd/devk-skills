# Clarifying intent — the Q&A loop

You have an intent hypothesis and possibly a few suspicious spots in mind. The point of this step is for *you* to come out the other side knowing exactly what the change is for and where the human wants you to focus.

## Step 1 — Confirm intent (always, in prose)

Always do this, even on small or self-explanatory-looking changes. The deal is: you write down the WHY of the change in your own words, the human tells you if you got it right.

This is the WHY of the *PR / change*, not the WHY of the review. Cover:

- What problem is being solved, or what capability is being added.
- The motivation — why this matters, what was wrong before, what the change unlocks.
- Roughly how the change accomplishes it (one or two sentences, not a file-by-file walkthrough).

Format:

```
**My read of this change:**

<one short paragraph — a few sentences — stating the intent and motivation.>

Does that match? Anything I got wrong or missed?
```

Rules:

- **Prose, not a/b/c.** The human needs to be able to correct you freely; binary options force a wrong shape onto a freeform answer.
- Be specific. "Refactors the auth flow" is useless; "Replaces the bespoke session-token check in the auth middleware with the new central `verifyToken` helper, so we stop diverging from the rest of the codebase" is what we want.
- Don't hedge. Commit to a real hypothesis. If you're guessing on a key piece, say *"I'm not sure whether X or Y — what's the actual driver?"* in the same paragraph.
- Don't pad with what you'll *do* in the review. Just state what you think the change is *for*.
- Keep it short. A few sentences. Not a wall of text.

If the human's response confirms your read, move on. If they correct you, internalize the correction and don't ask the same thing again — trust the human.

## Step 2+ — Follow-up clarifying questions (a/b/c)

After intent is locked in, you may have one or two specific things still worth asking — a suspicious spot, a focal-area choice, a constraint. Those *do* use a/b/c, one question per turn:

```
**Q: <single question, one short sentence>**

a) <concrete answer you think is most likely>
b) <concrete alternative answer>
c) Let me explain.
```

Rules for follow-ups:

- One question per turn. No stacked "also..." follow-ups.
- a) and b) must be plausible, concrete answers — not strawmen. The human should be able to pick one without explanation in most cases.
- c) is always "let me explain". Never replace it with a third concrete option; the human always needs an escape hatch to redirect you.
- Keep options short. One line each is ideal.
- Don't justify the question at length. A one-liner of context above the Q is fine; a paragraph isn't.

## What follow-ups are worth asking

Pick the question whose answer changes the most about your review. Usually one of:

- **A specific suspicious spot** — "Is `<file:line>` intentional or a mistake?" — when one spot looks off and the answer would change whether you investigate it deeply.
- **Scope/priority** — "What matters most here: correctness, performance, or readability?" — when the change is broad and review focus matters.
- **A constraint** — "Should I assume backward compat matters here?" — when an answer would meaningfully shape what you flag.

## What NOT to ask

- Anything you can answer by reading more diff. Read first.
- "What does this code do?" — your job is to figure that out.
- Style preferences — the standards enforcer handles those.
- Filler. If you have nothing meaningful to ask, say so and move on.

## After each answer — reconsider, then exit when ready

Each answer often kills 2–3 of your planned follow-ups. Re-rank what's still worth asking and stop fishing.

You're done when:

- You can state the change's intent in one or two sentences and the human would agree.
- You know which suspicious spots (if any) the human wants you to dig into.
- You know what *kind* of bugs would matter most (correctness, security, perf, UX).

When done, say something like *"Got it — kicking off the deep review now. Should be a couple of minutes."* Then move to step 6.

## Edge cases

- **Human's first message already states intent clearly.** Still write down your read in prose and ask "matches?". Skipping verification entirely defeats the skill; a short paragraph is cheap and surfaces misreads you didn't know you had.
- **Human corrects your intent.** Internalize the correction silently and proceed. Don't re-ask to verify — trust the human.
- **Human says "just review it, you figure it out".** Accept, lock in your hypothesis as the working intent, and proceed. Note in the final report that you reviewed against an unconfirmed intent.
- **Human says "I'm reviewing someone else's PR, I'm not sure of the intent either".** Treat the commit messages and PR description as the intent source. Still post your hypothesis and ask the human to sanity-check it (they may spot something the commits don't say). Mention in the final report that intent came from the commits, not the human.
