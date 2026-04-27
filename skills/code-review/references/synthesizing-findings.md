# Synthesizing findings

You've completed the mid-review check-in (step 7) and either got a green light or a small redirect. Now produce the polished human-facing report. It goes in chat only — never to a file unless the human asks.

## Aggregate

- Read each subagent's findings.
- **Deduplicate.** If two subagents flagged the same `file:line` (e.g., a section reviewer saw a logic bug *and* the standards enforcer flagged the surrounding style), merge into one finding. Keep the most informative framing — usually the bug framing wins over the style framing.
- **Cross-check.** If Section A's reviewer flagged something that depends on a caller in Section B (or in untouched code), confirm Section B's reviewer (or a quick grep you do yourself) doesn't already explain it away. Don't surface a finding that's already neutralized elsewhere.
- **Drop out-of-scope findings.** If the human's intent statement explicitly puts something out of scope (e.g., "I'm not worried about perf here"), drop perf findings unless they cross into correctness.

## Resolve ambiguous findings (a/b/c)

Some findings come back from subagents flagged as **ambiguous** — the answer "is this a bug?" depends on whether the author meant to do it. Don't auto-classify these. Don't drop them as low-confidence. Ask the human, one at a time, with this exact shape:

```
**Q: `<file:line>` — <one-sentence observation>. Is this:**

a) Bug — <one-line proposed fix>
b) Intentional because <plausible reason in one short clause>
c) Something else
```

Rules:

- **One question per turn.** Wait for the answer, then ask the next one.
- **Cap at ~3** ambiguous questions per review. If you have more, you're fishing — apply the confidence filter harder and drop the rest.
- **a) and b) must both be plausible.** If b) ("intentional because…") sounds absurd, you don't actually have an ambiguous finding — you have a bug. Just flag it in the report.
- **c) is always "something else"** — the human's escape hatch. Don't replace it with a third concrete option.
- **Skip this step entirely if you have nothing genuinely ambiguous.** Don't manufacture an a/b/c question to look thorough.

Folding answers into the report:

- **a) Bug** → finding stays, classified per the answer (Blocker / Concern).
- **b) Intentional** → finding drops. Don't include it as a Note "for visibility" — that's just noise.
- **c) Something else** → ask one short follow-up to understand, then classify.

## Filter — drop the low-confidence stuff

Apply the same test the subagents should have applied: *would you raise this in a real PR review with a senior colleague?* If no — drop it. If a finding came back from a subagent and you can't articulate why a senior reviewer would care, it doesn't go in the report.

Confidence threshold: <70% confidence → drop, don't downgrade. Low-confidence findings clutter the report and erode trust in the rest of it. Better a tight report with three real issues than a sprawling one with three real issues buried in twelve maybes.

## Rank

Sort findings in this order:

1. **Blockers** (bugs, broken caller contracts, silent error swallow on a real path, security issue at a boundary).
2. **Standards violations explicitly documented** in CLAUDE.md / AGENTS.md or other authoritative sources. These are usually labeled Concern but for a project that documented them, they shouldn't be ignored.
3. **Concerns** (likely problem worth discussing).
4. **Drifts** (convention not followed; from the standards enforcer).
5. **Nits / Notes**.

## Format the report

Keep it tight. Follow this exact shape in chat:

```markdown
## Code review — <scope, e.g. "last 4 commits" or "feat-auth vs main">

**Verdict:** <one line — e.g. "Two blockers to fix." or "Clean.">
**Intent:** <one sentence, the confirmed intent>

### Blockers (N)
- `file:line` — **<title>**. <one sentence on what's wrong>. → <fix in one phrase>

### Concerns (N)
- `file:line` — **<title>**. <one sentence>. → <fix in one phrase>

### Standards (N)
- `file:line` — **<title>** (rule: <source>). → <fix>

### Nits
<N nits — say if you want them listed>

---
*Reviewed N files across M sections. Skipped: <one-liner, only if anything substantive was skipped>.*
```

Format rules:

- **Verdict at the top, then intent.** Human sees the shape immediately.
- **Counts in section headers.** `### Blockers (2)` lets the human size each list before reading.
- **`file:line` first, title second.** Humans navigate by path; the title is secondary.
- **Drop empty sections entirely.** No "Standards: none." If there are no Blockers, the section doesn't exist.
- **Nits are collapsed by default.** Show the count in the section, not the list. Human can ask to expand. If there are zero nits, drop the section.
- **Footer is one italic line** with files reviewed and anything substantive skipped (lockfiles don't count as substantive — don't list them).
- **No "recommendations" section.** Suggested fixes go inline with each finding via `→`.

## Don'ts

- Don't paste full subagent outputs into the report.
- Don't invent severities to fill empty sections — drop empty sections. A report with only Concerns is fine.
- Don't include findings that wouldn't survive the *real-PR-with-a-senior-colleague* test. The filter step is mandatory, not optional.
- Don't list nits inline by default. Footer count only; expand on request.
- Don't write to a file. Chat only, unless the human asks.
- Don't summarize what you just did at the end. The report is the summary.

## When findings are sparse

If everything came back clean:

```markdown
## Code review — <scope>

**Verdict:** Clean.
**Intent:** <sentence>

*Reviewed N files across M sections. Standards check passed.*

If you want me to dig deeper on a specific spot, say which.
```

That's the whole report. No padding.

## When findings are dense

If you have 15+ real findings (after the filter step), the change is either large or genuinely problematic. Either way:

- Cap each section at the top ~6 entries; mention the rest as a count ("plus 4 more concerns — say if you want them listed").
- The verdict at the top should reflect the shape ("Several blockers; this needs work before merging.").
- Don't try to soften the report. A bad change deserves a clear assessment.
