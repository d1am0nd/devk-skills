---
name: lunch-break
description: Autonomous, hands-off "go explore the project" mode. The user picks a lens (UI, copy, SEO, a11y, what-next, competitive, etc.), walks away, and comes back to a written report of ideas and observations. Liberally fans out subagents in two phases (scout then deep-dive). Use when the user says "lunch break", "go explore <X>", "do a <X> review", "go wild on <X>", "audit <X> while I'm away", "look at the project for me", or invokes /lunch-break explicitly. The user picks ONE lens per run. Available lenses are auto-discovered from references/. If no lens is given, list the lenses and ask which.
---

# lunch-break — Autonomous, lens-driven project exploration

You were triggered by a user who is walking away from the computer. They want to come back to a written report containing observations and ideas about the project, viewed through a single chosen "lens". Internalize this before doing anything else:

- **The user is gone.** Don't pause for clarifications mid-run. If the lens is unambiguous and the intensity is decided, just go. If something is genuinely ambiguous and would change the whole run, ask once *before* dispatching subagents — never mid-run.
- **The deliverable is a report file plus a short chat summary.** Not code changes, not refactors, not even small drive-by fixes. The report is what the user comes back to read.
- **Liberal subagent use is the point.** This is not a token-saving exercise. Parallel exploration is how you cover ground while the user is away. Be generous.
- **Every lens has a reference file.** It encodes how that specific lens explores. Read the reference *first* — it is authoritative for that lens.
- **Ideas, not a Gantt chart.** The user is browsing for inspiration. Each finding should be a clear observation plus a concrete suggestion. They will pick what to act on later.

## Step 1 — Resolve the lens

Invocation patterns:

- `/lunch-break <lens>` — explicit
- "lunch break <lens>", "go explore <lens>", "do a <lens> review", "go wild on <lens>" — natural language with lens
- "lunch break", "go explore the project for me", "look at the project while I'm away" — natural language, no lens

If a lens is given (explicit or strongly implied — "go look at the UI" → `ui`), use it.

If no lens is given:
1. Use the `Glob` tool with pattern `references/*.md` rooted at this skill's directory to list lens files. (Do NOT use Bash for this — Glob is permission-free for project files; shell wildcards trigger a permission prompt.)
2. For each result, use the `Read` tool with `limit: 5` to grab just the frontmatter, then parse the `description:` field.
3. Show the user something like:
    ```
    Available lenses:
      - ui          → audit UI/UX rough edges across the app
      - copy        → review user-facing text for quality and personality
      - seo         → discoverability and search audit
      - a11y        → accessibility audit (WCAG, semantics, contrast, keyboard, screen reader)
      - what-next   → brainstorm features (LLM-powered ok, no manual human work)
      - competitive → research what similar products do that we don't
    Which lens should I run?
    ```
4. Wait for the user's pick. Picking the lens *is* the trigger — this is the only acceptable mid-trigger pause.

## Step 2 — Resolve the intensity

Two modes:

- **liberal** (default): single deep-dive wave. Roughly 8–15 subagents in Phase 2.
- **maximum**: multi-wave, roughly 20+ subagents total. Use when the user said anything like "wild", "maximum", "really thorough", "go nuts", "no holds barred", "all out", "take all the time you need".

If unclear, default to **liberal**. Tell the user in chat which mode you picked so they know what to expect before they walk away.

## Step 3 — Load the reference and execute

Read `references/<lens>.md` from this skill's directory. The reference defines:

- **Scope** — what the lens looks at and explicitly does not look at
- **Phase 1 scout** — initial broad survey, with how many subagents and what to ask each
- **Phase 2 deep dive** — how to fan out based on scout results, with subagent prompt templates
- **Finding shape** — what a useful observation looks like for this lens (each lens is shaped a little differently)

Follow the reference. If something in the reference doesn't fit this specific project (e.g. the project has no frontend and the lens is `ui`), adapt — but say so honestly in the report. Don't fake findings to fill a section.

## Step 4 — Synthesize and write the report

Write the report to: `.lunch-break/<lens>-<YYYY-MM-DD-HHMM>.md` (relative to the project working directory).

Create the `.lunch-break/` directory if it doesn't exist. If the project has a `.gitignore` and `.lunch-break` isn't in it, mention this in the chat summary so the user can add it if they want.

Use this structure:

```markdown
# Lunch Break: <lens>
Generated: <YYYY-MM-DD HH:MM>
Intensity: liberal | maximum
Subagents dispatched: <n total> (scout: <s>, deep dive: <d>)
Project: <project name from package.json / repo / cwd>

## TL;DR
- 3–5 bullet headline findings, each one short

## Findings
### <Title>
- **Where:** <files / routes / components / external sources>
- **Observation:** what you noticed (factual, brief)
- **Why it matters:** impact on user, business, or developer
- **Suggested change:** concrete proposal (no full code unless a tiny snippet helps)
- **Effort guess:** small / medium / large (rough, not load-bearing)

(Repeat per finding. Order: highest-impact first.)

## Stuff that didn't fit
Loose threads, hunches that didn't pan out into full findings, things worth a follow-up lunch break.

## What I looked at
Brief — what areas/files/dirs the subagents covered, and what was deliberately out of scope.
```

After writing the report, post a short summary in the chat: the same TL;DR bullets plus the path to the full report. Do not paste the full report into chat.

## Subagent model selection

Use the `model` parameter on `Agent` calls:

| Subagent role | Model | Why |
|---|---|---|
| Phase 1 — scout | `sonnet` | Broad mapping, low judgement |
| Phase 2 — deep dive | `sonnet` | Many parallel, pattern-spotting |
| Final synthesis (you) | inherit (omit) | Judgement-heavy work stays with the parent |

For maximum mode, the second wave can also be sonnet — what makes maximum more thorough is the *number* of subagents, not stronger models per agent.

## Hard rules

- **Never edit application code.** Only write the report file (and create `.lunch-break/`).
- **Never paste full file contents into the report.** Reference paths and quote short snippets only.
- **Never invent file paths or quotes.** If a subagent reports something specific, verify the path exists before citing it. Hallucinated references kill trust.
- **Never pad the report.** A short report with 3 sharp findings beats a long report with 12 weak ones. "Nothing critical found in this lens" is an honorable outcome — say so plainly if true.
- **Always run subagents in parallel within a wave.** Issue them in a single message with multiple Agent tool calls. Sequential dispatch defeats the purpose.
