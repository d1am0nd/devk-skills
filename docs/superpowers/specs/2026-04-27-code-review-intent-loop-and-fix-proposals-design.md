# Code review skill — intent loop + fix proposal upgrade

**Date:** 2026-04-27
**Scope:** `skills/code-review/SKILL.md` (step 5 + step 8 references), `skills/code-review/references/clarifying-intent.md`, `skills/code-review/references/synthesizing-findings.md`

## Why

Two related-but-independent improvements to the existing `code-review` skill:

1. **Intent step is too rigid.** Current step 5 forces a 1-shot prose hypothesis + "match?" exit. When the agent has genuine ambiguity in the WHY, it's told to either fold uncertainty into the prose or just commit. That's fine when the agent is confident, bad when it isn't — the human has to redirect a wrong read instead of being asked one good question up front.
2. **Fix proposals in the report are too thin.** Current report shape uses `→ <fix in one phrase>` — a single phrase per finding. Often not enough for the human to act without re-reading the diff. Upgrading to 1-2 lines (prose or a code snippet, agent's choice per finding) gives the human something they can paste into their editor.

## Change 1 — Intent step becomes a two-mode loop

### Shape

Each turn the agent picks one of two modes:

- **Confident mode.** Agent can state the WHY in 1–2 sentences with reasonable certainty → posts prose hypothesis + "Match?". (Same as today.)
- **Uncertain mode.** Agent has a genuine WHY-level ambiguity → posts a single a/b/c question targeting that ambiguity, one per turn.

After the human answers an uncertain-mode question, the agent re-evaluates. If now confident, switch to confident mode and post the prose + match. If still uncertain on a different axis, ask the next a/b/c.

**Always exit through prose + "Match?".** Even after a few a/b/c rounds, the wrap-up is the prose statement of the WHY. That's the final confirmation gate before subagents fire — keeps the exit shape uniform and gives the human one synthesized intent to confirm.

### Loop rules

- No hard cap on a/b/c iterations, but the bar is high: every question must materially change the WHY. Most reviews exit in 0 a/b/c (confident from the start). The expected case for uncertain mode is 1–2 a/b/c.
- "Materially change the WHY" test: would the agent's review *focus* differ depending on the answer? If no, don't ask.
- One question per turn. Never stack.

### a/b/c rules in the intent step (new)

- Question must be about the **WHY of the change**: motivation, scope of intent, what's being unlocked, what was wrong before.
- a) and b) must both be plausible, concrete answers — not strawmen. Human should be able to pick without explanation in most cases.
- c) is always "Let me explain." Never replace with a third concrete option — the human always needs an escape hatch.
- Banned (carry-over): focal-area questions ("which file matters most"), review-work questions ("is this line a bug"), commit-message commentary, padding/preamble.

### Threshold for switching modes

The agent flips to uncertain mode when there's a binary or ternary unknown whose answer would change the WHY itself. Examples:

- "Is the goal correctness or speed?" — yes, ask.
- "Is this scoped to feature X, or all of Y?" — yes, ask.
- "Which file matters most?" — no, that's focal area, not WHY.
- "Is `parseAuthHeader` line 42 intentional?" — no, that's review work.

### Diagram

A mermaid flowchart goes in `references/clarifying-intent.md` showing:

```
form intent hypothesis
     ↓
confident on WHY?
   yes → prose + "Match?" → human responds → exit step 5
   no  → one a/b/c question on WHY-level unknown → human responds → re-evaluate
```

Mermaid choice over dot/graphviz (which superpowers' own skills use): there's no rendering audience here, but mermaid renders in more places if the file is ever viewed in a markdown viewer. Minor inconsistency with house style accepted.

### File-level changes

**`skills/code-review/SKILL.md`** — step 5 bullets rewritten:

- Bullet 1: from "1–2 sentences, prose only" → "Confident on the WHY → 1–2 prose sentences + 'match?'. Uncertain → single a/b/c question on the WHY, one per turn, until confident."
- Bullet 2: from "No a/b/c here. No stacked follow-up questions." → "a/b/c allowed only for genuine WHY-level uncertainty — never for focal areas or review work. Never stack questions."
- Bullet 3: from "Move on as soon as the human responds" → "Move on once you can write a confident WHY the human has confirmed."

**`skills/code-review/references/clarifying-intent.md`** — substantial rewrite:

- Header: change from "This step is short: write your read in 1–2 prose sentences and ask the human to confirm. That is the whole step." → reflects the two-mode shape.
- Add mermaid flow diagram near the top.
- Format section: split into "Confident mode format" (prose + match, current shape) and "Uncertain mode format" (a/b/c on the WHY).
- Hard rules: revise to allow a/b/c for genuine WHY-level uncertainty; keep all other bans (focal-area, review-work, commit-message commentary, padding, no-stacked-questions).
- Examples:
  - Keep: "Good — concise" examples (confident mode).
  - **Flip:** "Good — genuine uncertainty folded in" → becomes **Bad — don't fold uncertainty into prose**, with new framing: "ask a clean a/b/c instead".
  - **Add:** new "Good — uncertain mode a/b/c" example showing the same export-pipeline case rephrased as a/b/c.
  - Keep: "Bad — stacked focal-area questions as fake a/b/c" (focal-area is still banned even with a/b/c now allowed for WHY-level).
- "After the human responds" section: extend to cover loop case (re-evaluate → continue or exit).
- "Where a/b/c questions DO belong" section at the bottom: revise — a/b/c is now used in two places (intent uncertain mode + ambiguous findings), with different scopes.

## Change 2 — Fix proposals upgraded from one phrase to 1-2 lines

### Shape

Each Blocker, Concern, and Standards finding gets a 1-2 line fix proposal. Agent picks per finding:

- **Code snippet** when the fix is mechanical and small — fenced block with `-`/`+` lines or just the replacement.
- **Prose** when the fix needs explanation or refactoring guidance that doesn't fit cleanly in 2 lines of code.

Hard cap: 2 lines. If a fix needs more, the finding shape collapses (write a one-line description of the fix direction and let the human ask for more).

Nits stay collapsed (count only) — no fix proposal.

### Format

Replace current `→ <fix in one phrase>` with a multi-line shape:

```markdown
- `file:line` — **<title>**. <one sentence on what's wrong>.
  ```ts
  - if (token === storedHash) { ... }
  + if (await verifyToken(token, storedHash)) { ... }
  ```
```

Or prose form:

```markdown
- `file:line` — **<title>**. <one sentence on what's wrong>.
  Fix: Replace the bespoke check with `verifyToken` from `lib/auth.ts`. Signature matches what the middleware already passes.
```

### File-level changes

**`skills/code-review/references/synthesizing-findings.md`**:

- "Format the report" section: update the example block. Change `→ <fix in one phrase>` to a 1-2 line fix block (code or prose).
- Format rules section: replace "Suggested fixes go inline with each finding via `→`" with a rule about 1-2 line fixes (code snippet preferred when mechanical; prose when explanation needed; hard cap of 2 lines).
- "When findings are sparse" / "When findings are dense" sections: no change.
- "Resolve ambiguous findings (a/b/c)" — the embedded `a) Bug — <one-line proposed fix>` format gets the same 1-2 line treatment as the main report fixes (code snippet or prose, agent picks, max 2 lines).

## Out of scope

- Refactoring the rest of the synthesis flow (rank order, severity definitions, dedup logic).
- Touching steps 1–4, 6–7 of the SKILL.md flow.
- Changing the project's spec/docs directory conventions (this spec creates `docs/superpowers/specs/` because nothing existed there yet; future skills can use the same path).

## Verification

- After implementation, dogfood the change on a small recent commit in this repo. Trigger an uncertain-mode intent question on a deliberately ambiguous diff (e.g., a refactor where the goal could plausibly be readability or perf). Confirm the loop exits via prose+match.
- Confirm a Blocker finding renders with a 1-2 line code-or-prose fix.
- No code change to the skill's *runtime* — these are markdown reference + SKILL.md edits. Verification = re-read the skill end-to-end and check it still hangs together.
