# Executing the plan — dispatch sections, review, advance, replan if needed

> Reference loaded by `devk-brainstorm` after `references/writing-plan.md` has produced an approved plan. Follow these instructions as if they replaced the main skill.

The plan is approved. Execute it section by section (with parallelism where the plan marked it safe), review each section's output, fix blockers, and proceed. Escalate to the human only when the plan itself needs to change.

## How subagent dispatch works here

This phase spawns several kinds of subagent — section implementers, section reviewers, a debugging helper, and a final-review agent. Subagents do NOT load skills themselves. You read the relevant subagent reference file and **inline its content into the subagent's prompt**. That pattern repeats for every dispatch in this document. The reference files live at:

- `references/subagents/section-tdd.md` — for implementing a single section (used per section)
- `references/subagents/reviewing-section.md` — for per-section review (used per section)
- `references/subagents/final-review.md` — for the end-of-work holistic review (used once)
- `references/subagents/researching-docs.md` — for dep/API verification (used if a new dep appears mid-execution)

`devk-debugging` stays as a top-level skill (users also invoke it directly). When you need it, instruct the subagent to load it via the Skill tool — that's an exception to the inlining pattern for this one skill.

## Core principles

- **Quality over speed. No hacks. Project lives for years.** If a section can't be completed properly, you pause and reassess — you do NOT hand-wave past a failure.
- **One subagent per section.** Fresh context for each. Subagents follow the inlined contents of `references/subagents/section-tdd.md`.
- **Parallel groups run in parallel**, in a single message with multiple Agent tool calls. Sequential sections run one after the other.
- **Per-section review is mandatory**, not optional. Each completed section → reviewer subagent (inlining `references/subagents/reviewing-section.md`) before the next starts (or the next group starts).
- **Each section commits independently** once it passes review. This gives a clean, bisect-friendly history and lets the human inspect any checkpoint.
- **Replan is always on the table.** If execution reveals the plan is wrong, stop and loop in the human with options.

## Output discipline (critical)

You are orchestrating — potentially dispatching many subagents whose individual outputs could flood the conversation. **Be terse to the human. Be silent during tool use.** The human is in product mode now — they want to know progress, not implementation mechanics.

Only surface messages to the user at these moments:
- **Starting execution** — one line: "Starting — <N pieces of work> ahead. I'll report back per piece and commit as I go."
- **Section complete** — one short line per section: "✓ <short human-readable description> done."
- **Parallel group complete** — "✓ <two descriptions> done in parallel."
- **Blocker / replan signal** — always surface in full. Frame the problem in plain language first, then the technical detail.
- **Final review running** — "Doing a last-pass review before handing back."
- **All done** — concise PM-friendly wrap-up (see "When all sections are done").

Avoid jargon in status lines. Section IDs like "S3a" are for internal tracking; the user sees "the registration endpoint" or "the email-validation piece". Keep the artifacts (`plan.md`, commits) technical — the *human-facing voice* is what softens.

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

**Sequential section:** one Agent call. Before dispatching, read `references/subagents/section-tdd.md` once into working memory — you'll reuse it across every section dispatch in this execution.

- `subagent_type`: `"general-purpose"`
- `model`: the default (or upgrade to Opus for complex sections — use judgment; Sonnet is fine for most)
- `description`: "S<ID>: <short title>"
- `prompt`:
  ```
  You are implementing one section of a plan with strict TDD. Follow these instructions exactly:

  <<< paste the full content of references/subagents/section-tdd.md here >>>

  Context:
  - The plan is at .devk/plan.md. Your section is S<ID>. Execute ONLY that section.
  - Spec: .devk/spec.md
  - Requirements: .devk/requirements.md
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

Before the first review dispatch, read `references/subagents/reviewing-section.md` once into working memory. Reuse it across every review dispatch.

For each completed section (or each of a parallel group), spawn a reviewer:

- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: "Review S<ID>"
- `prompt`:
  ```
  You are reviewing a single section's diff. Follow these instructions exactly:

  <<< paste the full content of references/subagents/reviewing-section.md here >>>

  Context:
  - Section reviewed: S<ID> from .devk/plan.md.
  - The section's changes are in the current working tree (git diff against the last known-good point will show what changed).
  - Return the structured findings the instructions specify.
  ```

Read the findings.

**Default stance: if a finding is a confirmed improvement, fix it. Don't file it as a TODO for the user.** The user expects you to own quality. Handing back a list of "things I noticed but didn't do" is not the job.

**Blocker findings:** must be fixed before proceeding. You fix them in the main agent OR dispatch a small fix subagent. Re-verify tests. Do NOT start the next section until blockers are cleared.

**Concern findings:** *default to fixing them now*. If the fix is clearly a win (tightens a contract, plugs an edge case, removes a hack, simplifies code) — do it as part of this section. If fixing would materially expand scope (separate refactor, new dependency, a day of work), *then* add to `.devk/progress.md` under "carry-forward concerns" and handle in the dedicated fix pass before final review. Escalate to the user only when the fix is ambiguous — when there's a real judgment call about whether it's an improvement at all.

**Nit findings:** fix anything that's a one-liner quality bump. Drop the rest silently.

### 5. Stuck handling (a section fails or spins)

If a section can't be completed — tests won't pass after reasonable iteration, or the subagent reports it's stuck — **do NOT ship the failure**.

Step 1: Spawn a debugging subagent. This is the one exception to the inlining pattern — `devk-debugging` stays a top-level skill (users also invoke it directly), so the subagent loads it via the Skill tool. Prompt: "Load and follow the `devk-debugging` skill via the Skill tool. Failing test: <…>. Section goal: <…>. What's been tried: <…>."

Step 2: If debugging resolves it, continue. If debugging also fails to resolve, that's a replan signal (step 6).

### 6. Replan signal (when the plan itself is wrong)

You hit a replan signal when ANY of:
- A section's blockers can't be fixed without changing assumptions the plan made.
- Two sections turn out to conflict in a way the plan missed.
- A reviewer flags that the approach (not the code) is wrong.
- Debugging reveals the spec-level assumption is broken (e.g., "the API we're integrating with doesn't work the way the spec assumed").

**STOP.** Mark progress accordingly. Present to the human in plain language:

> ## Hit a snag — need your input
>
> **What happened:** <1-2 sentences in plain language. E.g., "The third-party API we're integrating with works differently than we assumed — it paginates instead of returning all results at once.">
>
> **Why the plan doesn't fit anymore:** <specific, but non-technical if possible>
>
> **Options:**
> **a)** Small course correction — <minimal change that fixes it, no re-planning needed>
> **b)** Rework the design a bit — <change that touches the spec; will need a quick re-review>
> **c)** Step back and rethink — <if the whole approach needs another look>
>
> **What's done so far:** <plain-language list of what's working>.
> **What's blocked:** <plain-language list>.
>
> Which way do you want to go?

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

### 1. Carry-forward concern pass

Run through any concerns you deferred. Fix them. Commit: `devk: carry-forward fixes`.

### 2. Final review

Read `references/subagents/final-review.md` into working memory, then dispatch one subagent:

- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"` (or Opus for large changesets — use judgment)
- `description`: "Final review"
- `prompt`:
  ```
  You are doing a holistic end-of-work review. Follow these instructions exactly:

  <<< paste the full content of references/subagents/final-review.md here >>>

  Context:
  - .devk/requirements.md, .devk/spec.md, .devk/plan.md have the intent.
  - The changeset is the commit range from before execution started to HEAD.
  - Return the structured report the instructions specify.
  ```

When it returns, **apply the same default as per-section reviews: fix confirmed improvements inline, don't file them as TODOs.**

- **Critical findings** → fix. No discussion needed.
- **Worth-addressing findings** → if any of them are clearly correct improvements (a real bug, a missed edge case, an inconsistency, a hidden hack, a missed integration), fix them inline as part of this pass. Default to doing, not asking.
- **Only escalate to the user when:** a finding requires a spec-level change, introduces a new dependency, is clearly out of scope of the feature, or is genuinely ambiguous (it might not be an improvement). In those cases, state the finding plainly and ask.

Commit everything fixed in this pass: `devk: improvements from final review`. That's the last quality gate before handoff.

### 3. Wrap-up and .devk/ cleanup offer

Once final review is clean and all improvements are committed, present the wrap-up.

Keep the voice PM-friendly — describe what was built in product terms, not technical ones. Example tone: "Login now works end-to-end with Google sign-in and remembers sessions for a week" rather than "Implemented `AuthController.googleCallback` with JWT rotation".

Use this shape:

> ## Done
>
> <1-3 sentences describing what the user can now do, in plain language.>
>
> <Optional: one line about anything user-visible you want to call out — a new config flag, a migration they need to run, anything worth knowing.>
>
> **Working notes** in `.devk/`:
> - `requirements.md` — what we set out to build
> - `spec.md` — the technical design
> - `plan.md` — the N sections we broke it into (all done, all reviewed)
> - `progress.md` — section-by-section progress log
>
> What should I do with these? They've served their purpose.
>
> **a) Archive** — move to `.devk/archive/<YYYY-MM-DD>-<slug>/` (keeps a paper trail, easy to look back on). *Default.*
> **b) Keep only a slim summary** — one file with the feature title + any residual TODOs; delete the rest.
> **c) Delete everything** — clean slate. I'll confirm before removing.
> **d) Leave as-is.**

Act on the answer:

- **a)** `mkdir -p .devk/archive/<date>-<slug>/` and `git mv` (or `mv` if untracked) the four `.md` files into it. Commit: `devk: archive working notes for <feature title>`.
- **b)** Write a single `.devk/summary.md` with: feature title, one-paragraph description of what shipped, and any TODOs worth remembering (drawn from carry-forward concerns / final review items you chose not to fix). Delete the four original files. Commit: `devk: summarize and clean up working notes for <feature title>`.
- **c)** Show the file list, ask "delete these? yes/no", then `rm` on confirmation. Commit: `devk: remove working notes for <feature title>`.
- **d)** Do nothing.

Skip the commit step silently if not a git repo.

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
- **Default to fixing confirmed improvements. Don't hand the user a TODO list when you could just do the thing.**
- Human-facing voice is PM-friendly; technical artifacts stay technical.
- Replan is a first-class outcome, not a failure.
- Final review is the next step after all sections pass review (dispatch using `references/subagents/final-review.md`).
- After final review: auto-fix, commit, then present wrap-up + `.devk/` cleanup offer.
