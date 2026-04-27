# Clarifying intent

You've formed an intent hypothesis from the survey + smart read. This step is short: write your read in 1–2 prose sentences and ask the human to confirm. That is the whole step.

## What to write

The WHY of the *change*, not the WHY of the review. In one or two sentences, cover:

- What problem is being solved or what capability is being added.
- Roughly how the change accomplishes it (high-level — not file-by-file).

## Format

```
**My read of this change:**

<one or two sentences — just the WHY and the rough mechanism.>

Match? Anything I got wrong?
```

That is the entire message. Nothing else.

## Hard rules

- **1–2 sentences. Not a paragraph. Not two paragraphs.** If you can't fit it, you don't yet understand the change well enough — go read a bit more diff before posting.
- **Prose only. No a/b/c here.** No bulleted lists, no "things I'd dig into", no "focal areas", no "let me know if you want me to look at X". The intent step ends at "Match?". The a/b/c shape is for resolving ambiguous findings later — see `synthesizing-findings.md`.
- **No stacked follow-up questions.** Don't tack on a question about a specific spot, scope, focus, or a constraint. Those belong in the review itself, or as ambiguous-finding questions later.
- **No commentary on commit subjects.** "This commit says 'addresses review' — sounds like a previous review pass" is noise. The change is what it is now.
- **Don't hedge.** Commit to a real read. If you genuinely can't tell, fold the uncertainty into the same one or two sentences: *"I can't tell from the diff whether this is X or Y — which is it?"*
- **Don't pad.** No preamble ("Looking at the diff..."), no trailing "I'll then dispatch subagents to...".

## Examples

### Good — concise

> **My read of this change:** Adds gzip compression to HTTP responses and bundles `syntax.css` into the main hashed CSS file, both to cut cold-load latency.
>
> Match? Anything I got wrong?

### Good — concise

> **My read of this change:** Replaces the bespoke session-token check in the auth middleware with the new central `verifyToken` helper, so we stop diverging from the rest of the codebase.
>
> Match?

### Good — genuine uncertainty folded in

> **My read of this change:** Reworks the export pipeline — but I can't tell whether the goal is correctness (the old path silently dropped rows) or speed (parallel writes are new). Which is it?

### Bad — too long

> **My read of this change:** These two commits add gzip compression + bundle `syntax.css` into the main hashed CSS file to cut cold-load latency. The implementation has two non-obvious pieces:
>
> 1. A `defaultHTMLContentType` middleware that pre-sets `text/html` so chi's `Compress` middleware...
> 2. `cat syntax.css >> output.css` in both `Dockerfile` and `Taskfile`...
>
> The follow-up commit `ab77721` "addresses review" — sounds like a previous review pass already happened.

The implementation walkthrough is review work, not intent. Save it for the report. Commit-message commentary is noise.

### Bad — stacked focal-area questions as fake a/b/c

> **My read of this change:** ... [intent paragraph] ...
>
> A few specific things I'd dig into unless you steer me elsewhere:
>
> - a) Do other non-HTML handlers properly override the pre-set `text/html`?
> - b) Is the gzip middleware actually triggering for the routes you expect?
> - c) The new test does `os.Chdir("../..")` — is that going to bite us?

All three are review work. Do them in the review; don't ask the human to triage your scope before you've started. Even if one of them really were worth asking, a/b/c is "alternative answers to one question", not "three different questions".

## After the human responds

- **Confirms:** one sentence — *"Got it — kicking off the deep review now."* — then move to step 6.
- **Corrects:** internalize silently. Don't re-confirm. Trust them.
- **Says "just review it, you figure it out":** lock in your hypothesis as the working intent. Note in the final report that intent was unconfirmed.
- **They're reviewing someone else's PR and don't know intent:** treat commit messages as the intent source. Post your hypothesis as before. Note in the final report that intent came from commits, not the human.

## Where a/b/c questions DO belong

Not in the intent step. The a/b/c shape is for **resolving ambiguous findings during/after the review** — when something the review surfaced could plausibly be a bug *or* a deliberate choice, and the answer changes whether it gets flagged. Documented in `synthesizing-findings.md`.
