---
name: code-review
description: Interactive, human-in-the-loop code review for a local changeset — last N commits, branch vs main, staged hunks, working tree, or a specific commit/range. Treats the changeset as ONE logical change (never commit-by-commit). Heavy on context discipline (autogen, lockfiles, and large test bodies stay out of the main agent's context) and clarifies intent with the human before reviewing. Triggers on phrases like "review the last 4 commits", "review my branch vs main", "code review the staged changes", "use code-review on this", "review what I just did", "look at this PR locally", or explicit `/code-review`. Use whenever the user asks for a code review on local changes — even a casual "can you review this".
---

# code-review — Interactive review of a local changeset

You were triggered by a user who wants a careful review of a specific changeset on their local machine. Internalize this before doing anything:

- **The changeset is ONE change.** Whatever the human points at — last 4 commits, branch vs main, staged hunks — review the cumulative diff as a single logical change. Never review commit-by-commit. Commit boundaries on a feature branch are usually checkpoint noise.
- **Context discipline is the headline feature.** Diffs can be huge. Lockfiles and snapshot files alone can be tens of thousands of lines. You will be ruthless about what enters the main agent's context. The deep review is what subagents are for; the main agent stays clean.
- **The human collaborates, especially up front.** Step 1 is *understanding what the change is for*. You confirm that intent with the human before doing anything else. Only after the intent is locked in do you fan out subagents for the actual review.
- **Findings are reports, not fixes.** This skill never edits code. The human decides what to do with what you find. You are reviewing — for the human's own work or someone else's. Be objective either way.
- **One question at a time. Always.** This applies to every clarifying step in this skill.

## The flow

```
[1] resolve scope  →  [2] survey  →  [3] read smart  →  [4] hypothesize intent
                                                          ↓
                                    [5] confirm intent (Q&A, a/b/c)
                                                          ↓
                                    [6] dispatch parallel review (sections + enforcer)
                                                          ↓
                                    [7] consolidate, present in chat
```

Steps 1–5 happen in the main agent with strict context discipline. Step 6 is liberal subagent dispatch. Step 7 keeps the human-facing output tight.

## Step 1 — Resolve the scope

The user invokes this with something like:

- "review the last 4 commits" → `HEAD~4..HEAD`
- "review my branch vs main" → `$(git merge-base HEAD main)..HEAD` (use `master` if `main` doesn't exist; ask if neither does)
- "review the staged changes" → diff `--cached`
- "review what I just did" → if working tree dirty, the working tree; otherwise last commit (`HEAD~1..HEAD`)
- "review commit abc123" → `abc123^..abc123`
- "review this PR" → branch vs its base (ask which base if not obvious)
- "/code-review <scope>" — explicit

Translate the human's words into a single git revision range or set. If genuinely ambiguous, ask once with a/b/c options (see Step 5 format) before proceeding. Don't ask if you can read it from context.

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

While reading, mentally note any spots that look fishy (a removed test with no obvious replacement, an exception swallow, a flipped condition that might be wrong, a new TODO). Don't review them yet. They become candidate questions for step 5.

## Step 4 — Form an intent hypothesis

In your head: state in 1–2 sentences what this change is for. What is the human (or the original author) trying to accomplish? Why does this code exist?

If you can't write that sentence with reasonable confidence, you either need to read one or two more files (carefully — context budget) or just ask the human directly in step 5.

You are NOT reviewing yet. You are forming a hypothesis.

## Step 5 — Confirm intent with the human

Read `references/clarifying-intent.md` and follow its loop. Headlines:

- One question per turn. Format is always a / b / c.
- a) and b) are concrete answers you propose. c) is always "let me explain".
- After each answer, *reconsider* what (if anything) you still need to ask. Most planned follow-ups become moot after the first answer.
- Continue until intent is clear AND any specific suspicious spots you noticed have been clarified or noted.
- Two questions is normal. Four is too many.

Exit this step with:

- A clear, confirmed intent statement (one sentence).
- A list of focal areas (files / modules / behaviors) the deep review must cover.
- Optional: specific concerns the human flagged or asked you to investigate.

Tell the human you're kicking off the review. One sentence. Then move on.

## Step 6 — Dispatch the review

Read `references/dispatching-review.md` and follow it to slice the diff into sections and fan out subagents in parallel (single message, multiple Agent calls). Two roles:

- **Section reviewer** — owns one slice of the diff. Reads the slice fully + reads relevant existing code one level deeper (callers, callees, neighbors). Inline `references/subagents/section-reviewer.md` into the prompt.
- **Standards enforcer** — checks the changeset against the project's standards (CLAUDE.md, AGENTS.md, lint configs, neighbor patterns). One per changeset, not per section. Inline `references/subagents/standards-enforcer.md`.

The "one level deeper" rule for section reviewers is the whole point. A change to `parseAuthHeader` is reviewed alongside its callers and what `parseAuthHeader` itself calls. Sneaky bugs hide in those seams.

## Step 7 — Consolidate and present

Read `references/synthesizing-findings.md` and follow it. Aggregate findings, deduplicate, sort by severity, and post the report in chat only. Never write a file unless the human asks.

A senior engineer leaving review comments on a PR is the model. Tight, specific, no padding.

## Hard rules

- **Never edit code.** Findings are reports. The human decides what to fix.
- **Treat the changeset as one change.** Even when the human says "last 5 commits", the cumulative diff is what matters. Commit boundaries are noise on a feature branch.
- **Main agent stays clean.** Full diffs of test files, lockfiles, and generated stuff never enter your context. Subagents can have polluted context — that's their job.
- **Subagents look one level deeper.** Always. Reviewing a function without reading its callers and callees catches half the bugs and misses the other half.
- **No fishing for findings.** "Nothing significant to flag" is a valid outcome. Pad nothing.
- **One question at a time.** Always.
- **Output is chat only.** No files written by default.
