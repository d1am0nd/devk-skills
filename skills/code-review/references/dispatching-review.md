# Dispatching the review

You have confirmed intent. Now fan out subagents to do the actual review work. The main agent's job from here is orchestration and consolidation, not reading code.

## Slicing the diff into sections

Group changed files into review sections by *cohesion*, not by directory. A good section:

- Is small enough that one subagent can hold it in context (target: ≤500 lines of diff per section, occasional exceptions).
- Has internal coherence — the files in a section relate to each other (same feature area, same module, same boundary).
- Doesn't artificially split tightly-coupled changes (a route handler + its service should be in the same section).

Examples:

- Auth changes touching `lib/auth/`, `api/middleware/auth.ts`, and `tests/auth.test.ts` → one section.
- A migration plus the model change that depends on it → one section.
- An unrelated UI tweak that snuck into the same PR → its own section.

Aim for **2–6 sections**. Fewer if the change is small. More only if the change is genuinely sprawling — and if you reach 8+, you're over-slicing; merge.

## Subagent roster

Dispatch all of these in parallel — single message, multiple Agent tool calls.

### Section reviewers — one per section

- Inline `references/subagents/section-reviewer.md` into the prompt.
- Include in the prompt:
  - The git revision range (e.g. `HEAD~4..HEAD`, `main..HEAD`, `--cached`).
  - The confirmed intent statement (1–2 sentences).
  - The list of files in this section.
  - Any specific concerns the human flagged for this area (from step 5).
  - **The main agent's noted suspicions for files in this slice** (from step 3 — `file:line — one-liner` entries). Frame as *"investigate or rule out"*, not as confirmed findings.
  - The "look one level deeper" mandate (already in the role file, but reinforce it inline).

### Standards enforcer — one for the whole changeset

- Inline `references/subagents/standards-enforcer.md`.
- Include in the prompt:
  - The git revision range.
  - The confirmed intent.
  - The full list of changed files (the enforcer decides what's worth checking).

### Optional: dedicated test reviewer

- Spawn ONE if tests are a large or risky part of the change (e.g., the diff includes a new test framework, a major test refactor, or a security-sensitive area where test quality is the main concern).
- Otherwise, let section reviewers handle their own tests as part of their slice.
- If you spawn one: inline `references/subagents/section-reviewer.md` but instruct it to focus on test honesty, coverage gaps, and test sprawl.

## Models

- Section reviewers: `sonnet` is fine for typical work. Bump to `inherit` (parent's model, omit the `model` parameter) when the section involves security, money, auth, crypto, or anything where wrong = expensive.
- Standards enforcer: `sonnet`.
- The "one level deeper" greps and reads happen inside each subagent's own context — that's the point of using subagents at all.

## What to put in each subagent prompt

Every subagent prompt should contain, in this order:

1. The role-specific instructions (inlined verbatim from `references/subagents/<role>.md`).
2. The git revision range.
3. The confirmed intent.
4. The slice of files this subagent owns (or the full file list, for the enforcer).
5. Any specific concerns the human flagged (if relevant to this slice).
6. The main agent's noted suspicions for files in this slice (section reviewers only — not the enforcer).
7. A reminder that output goes back to the main agent in the format the role file specifies.

## Don't dispatch

- A "look at the whole codebase" subagent. Scope discipline applies — subagents review the diff and what it touches, not the project.
- A "summarizer" subagent. You consolidate in the main agent — that's a judgment call, not delegate-able.
- More than ~6 section reviewers. If you've sliced finer than that, you're over-slicing.
- A duplicate of the standards enforcer per section. One enforcer for the whole change is enough; it'll see the project's standards once and apply them to everything.

## After dispatch

While subagents run, do nothing else. Don't pre-read more diff "just in case" — it pollutes the context you'll need for synthesis. Wait for results.

When all subagents have returned, read their outputs and proceed to step 7 (mid-review check-in), then step 8 (synthesizing-findings).
