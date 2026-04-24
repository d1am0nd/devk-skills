# Per-section review — checklist for the main agent

> Loaded by `executing-plan.md` after a section's implementing subagent returns. **You (the main agent) run this review yourself** — no subagent dispatch. You already have the plan, spec, and section context in working memory; use it.

After a section completes, review JUST that section's diff, find bugs and missed edge cases, and act on the findings per the "What to do with findings" section below.

Stay scoped. You are NOT reviewing the whole project. You are NOT second-guessing the plan. You review the code that was just written for the section that just finished.

## What you're looking for

- **Bugs.** Code that doesn't do what the test thinks it does. Logic errors. Wrong comparison. Wrong default.
- **Off-by-one / boundary issues.** Indexing, range iteration, inclusive/exclusive bounds.
- **Missed edge cases.** Empty input, null/undefined, zero, negative, too large, malformed, unicode, timezones, race conditions in async code.
- **Error-path gaps.** What happens when a dependency fails, a network call throws, validation fails, DB is unreachable? Are errors surfaced appropriately, or silently swallowed?
- **Test gaps.** Do the tests cover the section's acceptance criteria honestly? Is a critical path untested? Are there tests that assert the wrong thing?
- **Test sprawl.** The inverse of test gaps. Flag when several tests share nearly identical setup and differ only in which field they assert on, or when five tests could be one parameterized test. Sprawly tests bloat future review context without adding coverage. This is a **concern**, not a blocker — note the specific consolidation (e.g., "S3 has 6 tests on `register()` that could be 2: one for the returned shape, one for duplicate-email rejection").
- **Hidden hacks.** `try/except: pass`. Commented-out assertions. `# TODO` as a permanent solution. Hardcoded values papering over logic. Silent fallback that hides failure.
- **Incorrect assumptions.** Code assumes input shape X that the upstream might not guarantee. Mutable default args. Shared mutable state.
- **Obvious safety issues.** Command injection, SQL injection, path traversal in user input. Basic OWASP stuff if the section touches a boundary.

## How to review

1. Find out what changed. The section's changes are uncommitted at review time — use `git diff HEAD` (and `git status` for new files). Do NOT use `HEAD~1`; that would pull in the previous section's commit.
2. Re-read the section's entry in `.devk/plan.md` — the acceptance criteria define "done." (You have this in context already, but glance at it to stay honest.)
3. Read the changed files fully — not just the diff — since a diff can hide context issues.
4. You already ran the section's tests in step 3 of the main loop. No need to re-run unless you didn't, or unless your review reading surfaces a reason to doubt the earlier green.
5. Form your read: where are the bugs and gaps?

Do NOT read the entire project. Do NOT review code that wasn't part of this section's diff. Context discipline matters — you have a lot more sections to get through.

## Structure your findings internally like this

Use this shape as a mental scaffold. You don't need to write it out as a document — findings flow directly into action per the next section.

```
# Section review: S<ID> — <title>

## Summary
<2-3 sentences: is the section solid, or are there real issues?>

## Blockers
- <Issue title> — <what's wrong, why it's a blocker>  (file:line, suggested fix)

## Concerns
<same format>

## Nits
<same format, OK to omit>
```

## Severity rubric

- **Blocker** — a bug, missed critical edge case, hidden hack, silent error swallow, or test gap on a critical path. Something that would cause a real user to see wrong behavior, or that makes the code untrustworthy. Must be fixed before the next section.
- **Concern** — real issue, fixable without major rework. Fix inline by default; only defer the ones that materially expand scope.
- **Nit** — cosmetic, naming, minor clarity. Pick up one-liner wins; drop the rest.

## What to do with findings

You are both reviewer and fixer. Don't hand the user a TODO list when you could just do the thing.

- **Blockers** → fix now. You can do it in the main agent, or dispatch a small fix subagent if the fix is scoped and benefits from a fresh context. Re-verify tests after. Do NOT start the next section until blockers are cleared.
- **Concerns** → default to fixing now as part of this section. If fixing would materially expand scope (separate refactor, new dependency, a day of work), note it in `.devk/progress.md` under "carry-forward concerns" for the dedicated fix pass before final review. Escalate to the user only when there's a real judgment call about whether it's an improvement at all.
- **Nits** → fix anything that's a one-liner quality bump. Drop the rest silently.

## Hidden-hack watchlist (blocker by default)

- `try: ... except: pass` without a specific exception and a real handling reason
- Catching a broad exception just to keep the happy path
- Commented-out assertions or tests
- `TODO: fix` as a permanent solution
- Hardcoded return values covering logic the tests don't actually exercise
- Skipping a test that should pass ("skip_if_ci", etc.) without justification
- Disabled lint / type rules with no comment explaining why

## What NOT to flag

- Code style / formatting (the project has a linter for that)
- Architecture-level decisions (that was the spec/plan's job)
- "You could also do X" suggestions that aren't improvements
- Refactors that aren't motivated by a bug or gap

## Hard rules

- You DO cite specific file:line locations in your internal findings — it's how you (and the user, if you escalate) navigate quickly.
- You DO NOT review code outside the section's diff.
- Keep the review tight. Focus beats exhaustive — you have more sections coming.
- Stay silent to the user during review unless you're escalating or surfacing a blocker you can't fix cleanly. The user gets the one-line "✓ <section> done" after you've reviewed and committed.
