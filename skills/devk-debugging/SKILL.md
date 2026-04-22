---
name: devk-debugging
description: Use when encountering any bug, test failure, stuck section, or unexpected behavior - before trying more fixes. Especially triggered by devk-section-tdd when a test won't pass after a few iterations. Enforces hypothesis-driven debugging (form hypothesis → test it → learn → iterate) instead of shotgun fixes. Prevents the "just try stuff" failure mode that makes bugs worse. Also triggered directly when the user says "this isn't working", "X is broken", "why is Y happening".
---

# devk-debugging — Hypothesis-driven debugging, not shotgun fixes

You are debugging a problem. The biggest failure mode when debugging with an LLM is **shotgun patching**: changing things hoping something sticks. This skill exists to prevent that. Debugging is a science: form a hypothesis, test it, learn, iterate.

## Core principles

- **Understand before fixing.** If you don't know why it's broken, you don't know if your fix works. You just know the symptom changed.
- **Quality over speed. No hacks.** Do not fix a bug with a silent try/except, a commented-out assertion, a hardcoded workaround, or by catching a broad exception to make the error go away. These are not fixes; they're hiding the problem.
- **One variable at a time.** If you change three things and the test passes, you don't know which one did it. That's a bad fix.
- **Trust the evidence, not your mental model.** When behavior contradicts your expectation, the expectation is wrong — investigate, don't rationalize.

## The loop

### Step 1: State the problem precisely

Write it down (in the conversation, or a scratch file). Concretely:

- What did you expect to happen?
- What actually happened?
- What's the exact error / incorrect output?
- When did it start failing? (If you have a before/after, that's gold.)
- What have you already tried? (So you don't loop.)

If you can't state the problem precisely, you're not ready to debug. Go back and gather data.

### Step 2: Form a hypothesis

What do you think is happening? Be specific:

- ❌ Bad: "Something's wrong with the API call."
- ✅ Good: "The response is being parsed as JSON before the Content-Type header is checked, so an HTML error page silently becomes `null`."

A good hypothesis is falsifiable — there's a specific observation that would confirm or refute it.

### Step 3: Design a test for the hypothesis

How would you know if your hypothesis is right? The test can be:

- **Observation** — add a log, print, or debugger breakpoint to see the actual value at a specific point.
- **Isolation** — remove components to see if the problem persists (binary search).
- **Contradiction** — construct an input that would produce a different result IF the hypothesis is right.

Pick the cheapest test that gives you the most information.

### Step 4: Run the test, record what happened

Actually run it. Observe the result. Record it. Do not skip this and guess.

### Step 5: Update your understanding

Three possibilities:

- **Hypothesis confirmed.** Now you know the cause. Proceed to step 6 (design the fix).
- **Hypothesis refuted.** You learned something. Go to step 2 with a new hypothesis informed by this observation.
- **Hypothesis inconclusive.** The test didn't actually answer the question. Design a better test (back to step 3).

### Step 6: Design the fix

ONLY after you understand the cause. The fix should:

- Address the root cause, not the symptom.
- Be minimal — just enough to fix the cause.
- Not introduce hacks. If you're tempted to silent-try/except, STOP and fix the real issue.
- Include a regression test: a test that would fail without the fix and passes with it. Write it first.

### Step 7: Apply the fix and verify

- Apply the fix.
- Run the regression test — confirm it passes.
- Run the broader test suite — confirm nothing else broke.
- If something else broke, you don't understand the cause as well as you thought. Back to step 2.

## Anti-patterns to avoid

- **Shotgun debugging.** Changing multiple things hoping one works. Always change one variable at a time.
- **Rationalizing.** "That shouldn't be happening, so maybe it's flaky" → no. If it happened once, understand why.
- **Symptom suppression.** `try: ... except: pass` to make the error go away. The bug is still there; you just can't see it.
- **Copy-paste fixes.** Pasting Stack Overflow code without understanding why your situation is the same.
- **Commenting out the failing test.** If the test reflects real behavior, fix the code, not the test. If the test is wrong, fix the test — but you need to know which.
- **Hardcoding the expected value.** "Let me just hardcode X for now" — no. Fix the logic that produces X.
- **Catching broad exceptions.** `except Exception:` tells you nothing and hides the real bug. Catch specifically, or don't catch.

## When you're genuinely stuck after several iterations

Signs:
- Every hypothesis gets refuted and you're running out of ideas.
- The bug only reproduces intermittently and you can't pin down the trigger.
- The code behaves in a way that seems to violate the language's semantics (this usually means your mental model is wrong).

Stop. Step back and either:

1. **Rubber-duck it.** Explain the full problem to the user as if they've never seen it. The act of explaining often surfaces the wrong assumption.
2. **Check your environment.** Is the test actually running the code you think? (Wrong module imported, cached bytecode, stale dependency, wrong branch, Docker container with different code.)
3. **Read the docs or source** of the library / framework. Your assumption about its behavior might be wrong.
4. **Ask the user.** Present your hypotheses, what you've ruled out, what evidence you have. The user may know context you don't.

## When invoked as a subagent

If the caller is `devk-section-tdd` or `devk-executing-plan`, return a clear report:

```markdown
# Debugging report: <short description>

## Root cause
<1-3 sentences on what was actually wrong>

## Evidence
<what you observed that confirmed the cause>

## Fix applied (or proposed)
<what you did, or what should be done>

## Regression test
<what test was added to prevent recurrence>

## Status
<resolved / still stuck + hypothesis>
```

If still stuck, say so honestly. Don't declare victory with a hack.

## Hard rules

- Hypothesis first, THEN change code.
- One variable at a time.
- No silent try/except. No commented tests. No hardcoded symptoms. Fix the cause.
- Every fix comes with a regression test.
- If genuinely stuck after real effort, escalate — don't ship a hack.
