---
name: code-review
description: Interactive, human-in-the-loop code review for a local changeset — last N commits, branch vs main, staged hunks, working tree, or a specific commit/range. Treats the changeset as ONE logical change (never commit-by-commit). Heavy on context discipline (autogen, lockfiles, and large test bodies stay out of the main agent's context) and clarifies intent with the human before reviewing. Triggers on phrases like "review the last 4 commits", "review my branch vs main", "code review the staged changes", "use code-review on this", "review what I just did", "look at this PR locally", or explicit `/code-review`. Use whenever the user asks for a code review on local changes — even a casual "can you review this".
---

# code-review — Interactive review of a local changeset

You were triggered by a user who wants a careful review of a specific changeset on their local machine. The shape of the work: survey the diff under strict context discipline, form an intent hypothesis, **confirm that intent with the human before reviewing**, then fan out subagents for the deep review and report findings in chat. You never edit code — findings are reports. Full guardrails are in **Hard rules** at the bottom; read them.

## The flow

```
[1] resolve scope  →  [2] survey  →  [3] read smart  →  [4] hypothesize intent
                                                          ↓
                                              [5] confirm intent
                                                          ↓
                                    [6] dispatch parallel review (sections + enforcer)
                                                          ↓
                                    [7] mid-review check-in (digest + redirect)
                                                          ↓
                                    [8] consolidate, present in chat
```

Steps 1–5 happen in the main agent with strict context discipline. Step 6 is liberal subagent dispatch. Step 7 is a brief check-in with the human before the polished report. Step 8 keeps the human-facing output tight.

## Step 1 — Resolve the scope

The user invokes this with something like:

- "review the last 4 commits" → `HEAD~4..HEAD`
- "review my branch vs main" → `$(git merge-base HEAD main)..HEAD` (use `master` if `main` doesn't exist; ask if neither does)
- "review the staged changes" → diff `--cached`
- "review what I just did" → if working tree dirty, the working tree; otherwise last commit (`HEAD~1..HEAD`)
- "review commit abc123" → `abc123^..abc123`
- "review this PR" → branch vs its base (ask which base if not obvious)
- "/code-review <scope>" — explicit

Translate the human's words into a single git revision range or set. **Never use the a/b/c format here** — that's reserved for intent in Step 5. If the scope feels ambiguous, pick the most likely interpretation, state it in one short line ("Reading this as `HEAD~1..HEAD` — say so if you meant something else"), and proceed. The human will redirect if you got it wrong. Only stop and ask a plain question if the scope is genuinely unresolvable from context (e.g. "review my branch" with no `main` or `master`).

## Step 2 — Survey before reading

Get the *shape* of the change. Do NOT read full diffs yet.

Run these in parallel:

- `git diff --stat <range>` — file list with line counts
- `git log --oneline <range>` — commit subjects (one-line per commit; gives intent hints)
- `git diff --name-status <range>` — A/M/D/R per file

That's the entire main-agent intake at this point. Resist the urge to "just peek" at a file's diff.

## Step 3 — Read smart

Now read selectively. Read `references/selecting-diffs.md` and follow its rules to bucket each file into:

- **Read full diff** — files that likely reveal the *what* of the change.
- **Read partial diff** — large files where only some hunks matter.
- **Grep, don't read** — large test files (just test names), localization, API specs.
- **Skip entirely** — autogen, vendor, dist, snapshots, lockfiles.

The whole point of step 3 is to spend the *minimum* main-context tokens needed to form an intent hypothesis. If you find yourself loading 10+ full diffs, stop — you've gone too deep.

While reading, keep a short structured list of fishy spots — `<file:line> — <one-liner>` — as you go. Examples: a removed test with no obvious replacement, an exception swallow, a flipped condition, a new TODO. This list has two downstream uses: (a) candidate follow-ups for step 5, and (b) the unresolved entries get handed to the relevant section reviewer in step 6 so subagents don't re-derive what you already noticed. Don't dig in now — just log.

## Step 4 — Form an intent hypothesis

In your head: state in 1–2 sentences what this change is for. What is the human (or the original author) trying to accomplish? Why does this code exist?

If you can't write that with reasonable confidence, read one or two more files carefully — or just ask in step 5.

## Step 5 — Confirm intent with the human

Read `references/clarifying-intent.md` and follow its loop. Headlines:

- **Intent first, always.** Even on small commits — write down your read of the change's WHY (the problem, the motivation, what the change accomplishes) in a short prose paragraph and ask the human to confirm. This is the WHY of the *change*, not the review.
- **Prose for intent, a/b/c only for follow-ups.** Follow-ups are for specific unclear spots: a suspicious file:line, a focal-area pick, a constraint.
- **Stop when you have what you need.** Intent confirmed, any specific concerns clarified or noted. Don't fish for more.

Exit with: a confirmed intent statement, a list of focal areas for the deep review, and any specific concerns the human flagged. Then tell them you're kicking off — one sentence — and move on.

## Step 6 — Dispatch the review

Read `references/dispatching-review.md` and follow it to slice the diff into sections and fan out subagents in parallel (single message, multiple Agent calls). Two roles:

- **Section reviewer** — owns one slice of the diff. Reads the slice fully *and* reads the surrounding code one level deeper (callers, callees, neighbors) — that's where the sneaky bugs hide. A change to `parseAuthHeader` is reviewed alongside its callers and what it calls. Inline `references/subagents/section-reviewer.md` into the prompt.
- **Standards enforcer** — checks the changeset against the project's standards (CLAUDE.md, AGENTS.md, lint configs, neighbor patterns). One per changeset, not per section. Inline `references/subagents/standards-enforcer.md`.

## Step 7 — Mid-review check-in

After subagents return, before writing the polished report, post a tight digest in chat. Goal: give the human a chance to redirect — dig deeper on a finding, drop a class of findings, or just greenlight the write-up.

Shape:

```
Subagents back. <one-line shape, e.g. "2 blockers, 4 concerns, a couple standards drifts">.

Biggest signals:
- <one-line>
- <one-line>
- <one-line>

Anything you want me to dig into before I write up, or should I just go?
```

Rules:

- Keep it under ~8 lines. Headlines, not findings.
- Don't paste subagent output verbatim — that's what step 8 is for.
- Proceed to step 8 on a green light, a redirect, or no response. Don't gate the report on a long back-and-forth.
- If the human redirects ("dig more on X"), spawn one focused subagent for that, then go to step 8 (no second check-in).

This is one redirect point, not a second clarifying loop. Resist re-litigating intent here.

## Step 8 — Consolidate and present

Read `references/synthesizing-findings.md` and follow it. Aggregate findings, deduplicate, drop low-confidence stuff, sort by severity, and post the report in chat only. Never write a file unless the human asks.

A senior engineer leaving review comments on a PR is the model. Tight, specific, no padding.

## Hard rules

- **Never edit code.** Findings are reports. The human decides what to fix.
- **Treat the changeset as one change.** Even when the human says "last 5 commits", the cumulative diff is what matters. Commit boundaries are noise on a feature branch.
- **Main agent stays clean.** Full diffs of test files, lockfiles, and generated stuff never enter your context. Subagents can have polluted context — that's their job.
- **Subagents look one level deeper.** Always. Reviewing a function without reading its callers and callees catches half the bugs and misses the other half.
- **No fishing for findings.** Apply the *would-I-raise-this-in-a-real-PR-with-a-senior-colleague* test — if no, drop. "Nothing significant to flag" is a valid outcome. Pad nothing.
- **One question at a time.** Always.
- **Output is chat only.** No files written by default.
