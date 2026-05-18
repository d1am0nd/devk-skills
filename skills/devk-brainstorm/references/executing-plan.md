# Executing the plan — dispatch waves in parallel, review, advance, replan if needed

> Reference loaded by `devk-brainstorm` after `references/writing-plan.md` has produced an approved plan. Follow these instructions as if they replaced the main skill.

The plan is approved. It's a directed graph of sections grouped into **waves**: everything in a wave is parallel-safe by design (sections touch disjoint files). You execute wave-by-wave — dispatching all sections in a wave in parallel, reviewing them in parallel, committing them once the wave passes review, then moving on.

## How subagent dispatch works here

This phase spawns subagents for: section implementation, per-section review, debugging when stuck, and final review. Subagents do NOT load skills themselves — you read the relevant subagent reference file and **inline its content into the subagent's prompt**.

- `references/subagents/section-tdd.md` — implementing one section (used per section, every wave)
- `references/subagents/reviewing-section.md` — reviewing one section (used per section, every wave)
- `references/subagents/final-review.md` — end-of-work holistic review (used once)
- `references/subagents/researching-docs.md` — dep/API verification (used if a new dep appears mid-execution)

`devk-debugging` stays a top-level skill (the user also invokes it directly for unrelated bugs). When you need it, instruct a subagent to load it via the Skill tool — that's the one exception to the inlining pattern.

## Core principles

- **Quality over speed. No hacks. Project lives for years.** A wave that doesn't pass review pauses the line — you do NOT advance.
- **Waves dispatch in parallel.** All sections in a wave fire in ONE message with multiple Agent tool calls. Same for review subagents. Sequential dispatch within a wave defeats the entire point.
- **One subagent per section per role.** Each section gets its own impl subagent and its own review subagent. Fresh context every time. That's how quality stays high at scale.
- **Per-section review is mandatory.** Never skipped. The reviewer subagent reads the section's diff, fixes in-scope issues, reports anything needing your call.
- **Per-section commits.** Even within a wave, commit each section independently in section-ID order once it passes review. Clean bisect-friendly history.
- **Replan is always on the table.** If review surfaces a spec or plan-level issue, stop and loop in the human.

## Output discipline (critical)

You are orchestrating potentially many parallel subagents. **Be terse to the user. Be silent during tool use.** The user is in product mode now — they want progress, not transcripts.

Surface messages only at:
- **Starting execution** — one line: "Starting — `<N>` pieces across `<M>` waves. I'll report per wave."
- **Wave starting** — one line per wave if it has more than one section: "Wave `<N>`: `<short human-readable list>`."
- **Wave complete** — one short line: "✓ Wave `<N>` done (`<short descriptions>`)."
- **Blocker / replan signal** — surface in full, plain language first, technical detail second.
- **Final review running** — "Doing a last-pass review before handing back."
- **All done** — concise PM-friendly wrap-up (see end of file).

Avoid jargon in status lines. Section IDs like `S3a` are for internal tracking; the user sees "the registration endpoint" or "the email-validation piece". Artifacts (`plan.md`, commits) stay technical; the human-facing voice softens.

Do NOT narrate:
- Every Agent call you make
- Tool results from subagents (read silently, summarize only if useful)
- Internal reasoning about which wave is next
- `git` operations (do them silently)
- Progress file updates

Rule of thumb: *the user wants a status line, not a transcript.* If something's going fine, stay quiet. If something's broken, speak up clearly.

## Setup before dispatching

1. Read `.devk/plan.md` carefully — the mermaid graph, the wave list, each section's entry.
2. **Record the pre-execution baseline.** If git repo: `git rev-parse HEAD` and remember it — the final-review subagent needs it to diff the full changeset. Else note "no baseline (not a git repo)."
3. Create `.devk/progress.md` if not present:
   ```markdown
   # Progress
   Baseline: <sha or "no baseline">

   ## Wave 1
   - [ ] S1: <title>

   ## Wave 2
   - [ ] S2: <title>
   - [ ] S3a: <title>
   - [ ] S3b: <title>

   ## Wave 3
   - [ ] S4: <title>
   ```
   Update after each section commits.

## The wave loop

For each wave in order:

### 1. Dispatch impl subagents — one message, multiple Agent calls

Read `references/subagents/section-tdd.md` once into working memory on the first wave; you'll reuse it across every section in every wave.

Send ONE message with N Agent tool calls, where N is the number of sections in this wave. Each call:

- `subagent_type`: `"general-purpose"`
- `model`: default (Opus for complex sections — use judgment; most sections are fine on the default)
- `description`: `"S<ID>: <short title>"`
- `prompt`:
  ```
  You are implementing one section of a plan with strict TDD. Follow these instructions exactly:

  <<< paste the full content of references/subagents/section-tdd.md here >>>

  Context:
  - The plan is at .devk/plan.md. Your section is S<ID>. Execute ONLY that section.
  - Spec: .devk/spec.md
  - Requirements: .devk/requirements.md
  - Material decisions (carried forward): <paste from plan>
  - Prior completed sections in earlier waves: <list IDs or "none yet">
  - Sections running in parallel with you in this wave: <list IDs> — they touch disjoint files per the plan, so coordinate by staying inside YOUR files only.

  Follow TDD strictly. Announce material decisions. Return the structured summary.
  ```

For a single-section wave, that's one Agent call. For a 3-section wave, three Agent calls in one message — the harness runs them concurrently.

### 2. Wait, then verify claims

When all impl subagents return, read their summaries. For each:
- Run the section's tests (or at least the section's test set) and confirm green. Don't trust the subagent's "tests passing" claim without checking.
- If any section returned STUCK or has failing tests → see step 5 (stuck handling).

If one section was STUCK, you can still dispatch reviews for the other sections that returned cleanly — review runs in parallel with debugging.

### 3. Dispatch review subagents — one message, multiple Agent calls

Read `references/subagents/reviewing-section.md` once into working memory on the first wave; reuse it after.

Send ONE message with one Agent call per completed section in the wave. Each call:

- `subagent_type`: `"general-purpose"`
- `model`: default (Sonnet is fine for most; upgrade to Opus only if the section is unusually subtle)
- `description`: `"Review S<ID>"`
- `prompt`:
  ```
  You are reviewing one section of a plan. Follow these instructions exactly:

  <<< paste the full content of references/subagents/reviewing-section.md here >>>

  Context:
  - Your section: S<ID> — <title>
  - Plan: .devk/plan.md (your section's entry has the acceptance criteria)
  - Spec: .devk/spec.md (read only what's relevant)
  - The section's changes are UNCOMMITTED. Use `git diff HEAD` and `git status` to see them.

  Read the diff, read the changed files in full, fix in-scope issues, report anything needing orchestrator action. Return the structured report.
  ```

### 4. Read review reports, route findings

For each review subagent's return:

- **Status: CLEAN** or **FIXED_IN_PLACE** — section is good. Re-run the section's tests to confirm the review's fixes didn't break anything. Move to commit (step 7).
- **Status: NEEDS_ORCHESTRATOR_ACTION** — there are open items. Look at each:
  - `BLOCKER` (in-section but reviewer didn't fix) → dispatch a focused fix subagent (inline `section-tdd.md`, instruct it to address only the listed blocker, no scope creep). Re-verify when it returns.
  - `OUT_OF_SCOPE` (touches other sections / spec / plan) → this is a replan signal. Stop the wave loop. Go to step 6.
  - `NEEDS_HUMAN` → escalate to the user with the finding + suggested resolution + a/b options.
- **Status: STUCK** — review couldn't complete. Investigate briefly yourself (read the diff). If you can resolve, do so. If not, dispatch debugging.

The point of the review subagent is to keep the main agent's context light. **Don't re-read the diff yourself unless the review report indicates a problem.**

### 5. Stuck handling (impl returned STUCK or tests fail)

Dispatch the debug subagent — the one exception to the inlining pattern, because `devk-debugging` is its own skill:

- `subagent_type`: `"general-purpose"`
- `description`: `"Debug S<ID>"`
- Prompt: `"Load and follow the devk-debugging skill via the Skill tool. Failing test: <…>. Section goal: <…>. What's been tried: <…>."`

If debugging resolves it → go back to step 3 to review the now-fixed section. If debugging also fails → replan signal (step 6).

### 6. Replan signal

You hit a replan signal when ANY of:
- Reviewer flags `OUT_OF_SCOPE` or `NEEDS_HUMAN` for a spec/plan-level issue
- Two sections in the same wave produce a conflict the plan missed
- Debugging can't resolve a blocker without changing assumptions the plan made

STOP. Update `.devk/progress.md` to reflect the partial state. Present to the human in plain language:

> ## Hit a snag — need your input
>
> **What happened:** <1-2 sentences in plain language. E.g., "The third-party API paginates instead of returning all results — that wasn't in the spec.">
>
> **Why the plan doesn't fit anymore:** <specific, non-technical if possible>
>
> **Options:**
> **a)** Small course correction — <minimal change, no re-planning needed>
> **b)** Rework the design a bit — <change that touches the spec; will need a quick re-review>
> **c)** Step back and rethink — <if the whole approach needs another look>
>
> **What's done so far:** <plain-language list of what's working>
> **What's blocked:** <plain-language list>
>
> Which way do you want to go?

**Do not silently fix spec-level problems with code hacks.** That's the core "no hacks" case.

### 7. Commit each section in the wave

Once all sections in the wave have passed review (or all blockers cleared, all tests green), commit each section in section-ID order. One commit per section.

Skip silently if not a git repo. Otherwise, for each section:

1. Stage only files changed by this section. Use `git diff --name-only HEAD` to confirm — sections in the same wave are file-disjoint by plan design, so this should be clean.
2. Match the project's commit convention if one is clear from `git log --oneline -10`. Default `devk: S<ID> <section title>`.
3. If pre-commit hooks fail: this is signal, not nuisance. Fix the underlying issue, re-stage, new commit. Never `--no-verify`.

After all sections are committed, update `.devk/progress.md` and commit it (or fold it into the last section's commit — minor preference).

Then output one line to the user: `"✓ Wave <N> done (<short descriptions>)."`

### 8. Advance

Move to the next wave. Loop.

---

## When all waves are done

### 1. Carry-forward concern pass

Most concerns get fixed inline by review subagents under the new model, so this pass is often empty. If anything was deferred, fix it now. Commit: `devk: carry-forward fixes`.

### 2. Final review

Read `references/subagents/final-review.md` into working memory, then dispatch one subagent:

- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"` (or Opus for large changesets — use judgment)
- `description`: `"Final review"`
- `prompt`:
  ```
  You are doing a holistic end-of-work review. Follow these instructions exactly:

  <<< paste the full content of references/subagents/final-review.md here >>>

  Context:
  - .devk/requirements.md, .devk/spec.md, .devk/plan.md have the intent.
  - Pre-execution baseline SHA: <paste the baseline SHA from Setup step 2>. The changeset is <baseline>..HEAD. (If "no baseline" was recorded, review the working tree + uncommitted diff.)
  - Return the structured report the instructions specify.
  ```

When it returns:

- **Critical findings** → fix. No discussion needed.
- **Worth-addressing findings** → if any are clearly correct improvements (a real bug, a missed edge case, an inconsistency, a hidden hack, a missed integration), fix them inline now. Default to doing, not asking.
- **Escalate to the user only when:** a finding requires a spec-level change, introduces a new dependency, is clearly out of scope of the feature, or is genuinely ambiguous. Then state the finding plainly and ask.

Commit everything fixed in this pass: `devk: improvements from final review`.

### 3. Wrap-up and `.devk/` cleanup offer

Present the wrap-up. PM-friendly voice — describe what was built in product terms, not technical ones.

> ## Done
>
> <1-3 sentences describing what the user can now do, in plain language.>
>
> <Optional: one line about anything user-visible worth knowing — a new config flag, a migration to run, etc.>
>
> **Working notes** in `.devk/`:
> - `requirements.md` — what we set out to build
> - `spec.md` — the technical design
> - `plan.md` — the waves we broke it into (all done, all reviewed)
> - `progress.md` — section-by-section progress log
>
> What should I do with these? They've served their purpose.
>
> **a) Archive** — move to `.devk/archive/<YYYY-MM-DD>-<slug>/` (keeps a paper trail, easy to look back on). *Default.*
> **b) Keep only a slim summary** — one file with the feature title + any residual TODOs; delete the rest.
> **c) Delete everything** — clean slate. I'll confirm before removing.
> **d) Leave as-is.**

Act on the answer:

- **a)** `mkdir -p .devk/archive/<date>-<slug>/`, `git mv` (or `mv` if untracked) the four `.md` files into it. Commit: `devk: archive working notes for <feature title>`.
- **b)** Write `.devk/summary.md`: feature title, one-paragraph description, any TODOs worth remembering (from carry-forward concerns or final-review items not fixed). Delete the four originals. Commit: `devk: summarize and clean up working notes for <feature title>`.
- **c)** Show the file list, ask `"delete these? yes/no"`, then `rm` on confirmation. Commit: `devk: remove working notes for <feature title>`.
- **d)** Do nothing.

Skip commits silently if not a git repo.

---

## What you do NOT do

- Skip review on a section. Every section gets reviewed.
- Mark a section "done" if its tests don't pass.
- Silently fix spec-level problems with code hacks.
- Advance past blockers by rationalizing "we can fix it later."
- Run wave `N+1` before wave `N` is fully reviewed and committed.
- Bundle multiple sections into one subagent to "go faster" — one subagent per section per role.
- Dispatch sections in a wave one at a time. Always one message, multiple Agent calls.

## Reminders

- Waves dispatch in ONE message with N Agent calls. Same for reviews.
- Quality over speed. A failed wave stops the line.
- Default to fixing confirmed improvements; don't hand the user a TODO list.
- Human-facing voice is PM-friendly; technical artifacts stay technical.
- Replan is a first-class outcome, not a failure.
- Per-section commits in section-ID order for clean bisect history.

## If the user wants to stop here

If the user signals they want to pause or drop the work mid-execution ("stop", "let's pause", "actually nevermind"), acknowledge and offer to tidy up `.devk/`.

> Got it — pausing this.
>
> Working notes in `.devk/`:
> - `requirements.md`, `spec.md`, `plan.md`, `progress.md` — `<N>` of `<M>` sections done.
>
> What should I do with them?
>
> **a) Leave as-is** — I'll pick up next time. *Default if you might come back to this.*
> **b) Archive** — move to `.devk/archive/<YYYY-MM-DD>-<slug>/`.
> **c) Delete** — clean slate. I'll confirm first.

Act on the answer the same way the other phases do.
