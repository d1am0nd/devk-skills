# Synthesizing findings

Subagents have returned. Now you produce the human-facing report. It goes in chat only — never to a file unless the human asks.

## Aggregate

- Read each subagent's findings.
- **Deduplicate.** If two subagents flagged the same `file:line` (e.g., a section reviewer saw a logic bug *and* the standards enforcer flagged the surrounding style), merge into one finding. Keep the most informative framing — usually the bug framing wins over the style framing.
- **Cross-check.** If Section A's reviewer flagged something that depends on a caller in Section B (or in untouched code), confirm Section B's reviewer (or a quick grep you do yourself) doesn't already explain it away. Don't surface a finding that's already neutralized elsewhere.
- **Drop out-of-scope findings.** If the human's intent statement explicitly puts something out of scope (e.g., "I'm not worried about perf here"), drop perf findings unless they cross into correctness.

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
## Code review — <scope description, e.g. "last 4 commits" or "feat-auth vs main">

**Intent:** <one sentence, the confirmed intent>

### Blockers
- **<title>** — <one sentence>. `<file:line>`
  Suggested: <one sentence>

### Concerns
- **<title>** — <one sentence>. `<file:line>`
  Suggested: <one sentence>

### Standards
- **<title>** — <one sentence>. `<file:line>` (rule: <source>)

### Nits
- <one-liner>. `<file:line>`

### What I looked at
- <N> files reviewed across <M> sections; standards check covered <X>.
- Skipped: <e.g. lockfile, generated migration, snapshot fixtures — list briefly>.

### Verdict
<one line — e.g. "Two blockers to fix." or "Clean — nothing significant to flag.">
```

## Don'ts

- Don't paste full subagent outputs into the report.
- Don't invent severities to fill empty sections — drop empty sections entirely. A report with only Concerns and a Verdict is fine.
- Don't add a separate "recommendations" section. Suggested fixes go inline with each finding; the human doesn't need a second list.
- Don't write to a file. Chat only, unless the human asks.
- Don't summarize what you just did at the end ("I reviewed X and found Y!"). The report is the summary.

## When findings are sparse

If everything came back clean:

```markdown
## Code review — <scope>

**Intent:** <sentence>

Nothing significant to flag. Reviewed <N> files across <M> sections. Standards check passed.

If you want me to dig deeper on a specific spot, say which.
```

That's the whole report. No padding.

## When findings are dense

If you have 15+ real findings, the change is either large or genuinely problematic. Either way:

- Cap each section at the top ~6 entries; mention the rest as a count ("plus 4 more nits — let me know if you want them listed").
- Lead with the verdict so the human knows the shape before reading details.
- Don't try to soften the report. A bad change deserves a clear "this needs work before merging".
