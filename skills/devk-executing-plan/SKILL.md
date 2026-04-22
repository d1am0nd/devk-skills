---
name: devk-executing-plan
description: Use after devk-writing-plan has produced an approved plan at .devk/plan.md. This skill orchestrates execution - it dispatches each plan section to a fresh subagent running devk-section-tdd, runs parallel-safe groups in parallel, spawns devk-reviewing-section after each section completes, incorporates fixes, and handles the mid-flight replan escape hatch when something fundamental turns out to be wrong. Ends by invoking devk-final-review.
---

# devk-executing-plan — Dispatch sections, review, advance, replan if needed

The plan is approved. Execute it section by section (with parallelism where the plan marked it safe), review each section's output, fix blockers, and proceed. Escalate to the human only when the plan itself needs to change.

## Core principles

- **Quality over speed. No hacks. Project lives for years.** If a section can't be completed properly, you pause and reassess — you do NOT hand-wave past a failure.
- **One subagent per section.** Fresh context for each. Subagents run `devk-section-tdd`.
- **Parallel groups run in parallel**, in a single message with multiple Agent tool calls. Sequential sections run one after the other.
- **Per-section review is mandatory**, not optional. Each completed section → `devk-reviewing-section` subagent before the next starts (or the next group starts).
- **Each section commits independently** once it passes review. This gives a clean, bisect-friendly history and lets the human inspect any checkpoint.
- **Replan is always on the table.** If execution reveals the plan is wrong, stop and loop in the human with options.

## Output discipline (critical)

You are orchestrating — potentially dispatching many subagents whose individual outputs could flood the conversation. **Be terse to the human. Be silent during tool use.**

Only surface messages to the user at these moments:
- **Starting execution** — one line: "Starting execution: N sections, <K parallel groups>. Committing per section."
- **Section complete** — one line per section: "✓ S<ID> done (N tests passing, N files, committed)."
- **Parallel group complete** — "✓ Group [S3a, S3b] complete."
- **Blocker / replan signal** — always surface in full.
- **Final review ready** — "Running final review."
- **All done** — concise summary (see "When all sections are done").

Do NOT narrate:
- Each Agent call you make
- Tool results from subagents (read silently, summarize only if relevant)
- Your internal reasoning about which section is next
- `git` operations (do them silently; mention the commit SHA on completion if useful)
- Progress tracking file updates

Treat this as: *the human wants a status line, not a transcript.* If something is going fine, stay quiet. If something is broken, speak up immediately and clearly.

## Setup before dispatching

1. Read `.devk/plan.md` carefully — sections, parallel groups, dependencies, TDD outlines.
2. Create `.devk/progress.md` (if not present) to track section status. Schema:
   ```markdown
   # Progress
   - [x] S1: <title> — done, reviewed, 0 blockers
   - [ ] S2: <title> — in progress
   - [ ] S3a: <title> — pending (parallel with S3b)
   - [ ] S3b: <title> — pending (parallel with S3a)
   ```
   Update this after each section completes.

## The main loop

For each section (or parallel group) in execution order:

### 1. Dispatch

**Sequential section:** one Agent call.

- `subagent_type`: `"general-purpose"`
- `model`: the default (or upgrade to Opus for complex sections — use judgment; Sonnet is fine for most)
- `description`: "S<ID>: <short title>"
- `prompt`:
  ```
  Load and follow the `devk-section-tdd` skill via the Skill tool.

  The plan is at `.devk/plan.md`. Your section is **S<ID>**. Execute ONLY that section.

  Context you have:
  - Spec: `.devk/spec.md`
  - Requirements: `.devk/requirements.md`
  - Plan: `.devk/plan.md` (read YOUR section fully)
  - Material decisions (carried forward): <paste them here from the plan>
  - Prior completed sections: <list IDs or "none yet">

  Follow TDD strictly. Announce material decisions in your return summary.
  Return a concise summary: what you did, files touched, test results, any announcements.
  ```

**Parallel group:** send multiple Agent tool calls in ONE message. Each subagent gets its own section. This is how parallelism works — don't dispatch them serially.

### 2. Wait and collect

Wait for the section(s) to complete. Read each subagent's summary. Don't assume the section went perfectly — the subagent's summary is a claim, not proof.

### 3. Verify the section's claims

Before proceeding to review, do a quick sanity check yourself:
- Run the project's test suite (or at least the tests the section added) and confirm they pass.
- If tests don't pass, the section is NOT done. Go to step 5 (stuck handling).

### 4. Dispatch per-section review

For each completed section (or each of a parallel group), spawn a reviewer:

- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: "Review S<ID>"
- `prompt`:
  ```
  Load and follow the `devk-reviewing-section` skill via the Skill tool.

  Section reviewed: S<ID> from `.devk/plan.md`.
  The section's changes are in the current working tree (git diff against the last known-good point will show what changed).
  Return the structured findings the skill specifies.
  ```

Read the findings.

**Blocker findings:** must be fixed before proceeding. You fix them in the main agent OR dispatch a small fix subagent. Re-verify tests. Do NOT start the next section until blockers are cleared.

**Concern findings:** if easy, fix inline. If substantial, add to `.devk/progress.md` under "carry-forward concerns" and address in a dedicated fix pass before `devk-final-review`.

**Nit findings:** ignore unless the fix is a one-liner that improves clarity.

### 5. Stuck handling (a section fails or spins)

If a section can't be completed — tests won't pass after reasonable iteration, or the subagent reports it's stuck — **do NOT ship the failure**.

Step 1: Spawn a debugging subagent running `devk-debugging` with the specific failure context. Give it the failing test, the section's goal, and what's been tried.

Step 2: If debugging resolves it, continue. If debugging also fails to resolve, that's a replan signal (step 6).

### 6. Replan signal (when the plan itself is wrong)

You hit a replan signal when ANY of:
- A section's blockers can't be fixed without changing assumptions the plan made.
- Two sections turn out to conflict in a way the plan missed.
- A reviewer flags that the approach (not the code) is wrong.
- Debugging reveals the spec-level assumption is broken (e.g., "the API we're integrating with doesn't work the way the spec assumed").

**STOP.** Mark progress accordingly. Present to the human:

> ## Replan needed — section S<ID>
>
> What happened: <1-2 sentences>
>
> Why the plan doesn't work as-is: <specific>
>
> Options:
> **a)** Amend the plan: <minimal change that fixes it>
> **b)** Amend the spec: <change that might need re-review>
> **c)** Pause and redirect: <if the whole thing should rethink>
>
> What's done so far: <list>. What's blocked: <list>.
>
> Which path?

**Do not silently fix a spec-level problem with a code hack.** That's exactly the "no hacks" principle's core case.

### 7. Commit the section

Once the section passes review (blockers cleared, tests green), commit the section's changes before moving to the next section. This keeps each section as a clean checkpoint in history.

Skip silently if not a git repo. Otherwise:

1. Stage only files changed by this section. Use `git diff --name-only HEAD` to see what's touched; confirm nothing unrelated is included. Include `.devk/progress.md` in the commit too.
2. Match project commit convention if one is clear from `git log --oneline -10`. Default to `devk: S<ID> <section title>`.
3. If pre-commit hooks fail: this is signal, not nuisance. Fix the underlying issue, re-stage, new commit. Never use `--no-verify`.

Example:
```
git add <section's touched files> .devk/progress.md
git commit -m "devk: S<ID> <section title>"
```

Surface only the one-line completion message to the user ("✓ S<ID> done (N tests passing, N files, committed <sha>)."). Don't paste the diff or narrate the commit.

### 8. Update progress and proceed

After each section + review + commit:
- Update `.devk/progress.md` (already committed with the section)
- Move to next section or group
- If a parallel group's siblings finished, dispatch the next sequential section

**Note on parallel groups + commits:** sibling sections in a parallel group produce a single combined commit once *all* siblings have passed review (they share a checkpoint point). Commit message: `devk: [S3a, S3b] <group description>`.

## When all sections are done

Run the carry-forward concern pass (the substantive concerns you deferred). Fix them. Commit them: `devk: carry-forward fixes`.

Then invoke `devk-final-review` via the Skill tool. If it finds critical issues, fix them and commit: `devk: fixes from final review`. That's the last quality gate before handoff.

## What you do NOT do

- Skip a section's review.
- Mark a section "done" if tests don't pass.
- Silently fix spec-level problems with code hacks.
- Advance through blockers by rationalizing "we can fix it later."
- Bundle multiple sections into one subagent to go faster — one subagent per section is how we protect context.

## Reminders

- Quality over speed. A broken section stops the line.
- Parallel groups in one message, serial sections one at a time.
- Per-section review is mandatory.
- Replan is a first-class outcome, not a failure.
- `devk-final-review` is the next skill after all sections pass review.
