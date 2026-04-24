---
name: devk-tech-debt-audit
description: Autonomous, hands-off audit that surveys a codebase, identifies up to 3 most critical tech-debt items with high confidence, and proposes concrete refactoring plans. Use when the user says "audit tech debt", "find tech debt", "what should we refactor", "what's rotting", "review the project for issues", "propose refactorings", "what's slowing us down", or invokes /devk-tech-debt-audit explicitly. The human triggers this once and walks away; you return a report at ./devk/TECH_DEBT_REPORT_{YYYY-MM-DD_HH}.todo.md plus a short in-chat summary. You do NOT write code - exploration and proposals only. Quality of the findings is entirely your responsibility; the human will not verify your code-level reasoning. "Everything is fine" is a valid outcome if nothing passes the confidence bar.
---

# devk-tech-debt-audit — Autonomous tech-debt exploration and proposal

You are running a **hands-off tech-debt audit**. The human triggered this, is not watching, and will only come back to read a report. Internalize this before anything else:

- **You own the quality bar completely.** The human will not review your code-level reasoning — they will only read your conclusions. If you cry wolf, they lose trust. If you miss something critical, the project rots. Calibrate accordingly.
- **High confidence is mandatory.** Only surface findings you would stake your reputation on. If you're unsure whether something is debt, it isn't — or at least it doesn't belong in this report.
- **"Everything looks good" is a valid, honorable outcome.** If the project is in decent shape, say so and stop. Padding the report with borderline concerns is worse than an empty report.
- **No code changes. Ever.** This skill is exploration + proposal. Not a single line of application code is written during this run. (Writing the report file itself is fine.)
- **Size of the proposed refactor is irrelevant.** If the right answer is "rewrite the entire service layer," propose it. But then be honest about cost, risk, and the test design needed to do it safely.
- **Liberal use of subagents is mandatory.** Parallel discovery is how you cover ground without drowning your context. Independent verification is how you avoid fooling yourself. Both are non-negotiable — see the subagent patterns below.
- **The report is for a human who does not want to read code.** Write for impact, cost, and risk — not implementation detail. File:line references belong in an appendix-style evidence block, not in the prose.

## Model selection for subagents

Set the `model` parameter on `Agent` calls according to this table. For "inherit," **omit the `model` parameter** — the subagent will use whatever model the parent is running. This matters: on an Opus-run audit, the hard reasoning subagents should get Opus.

| Subagent | Model | Why |
|---|---|---|
| Phase 1 — Project survey | `sonnet` | Factual scraping, no judgement |
| Phase 2 — Dimension scans (×N) | `sonnet` | Many, parallel, pattern-spotting; cost matters |
| Phase 4a — Scope explorer | `sonnet` | Dependency tracing is mechanical |
| Phase 4a — Devil's advocate | **inherit** | This is the confidence gate — worth the stronger model |
| Phase 4b — Refactor proposal | **inherit** | Hardest synthesis: scope + risk + migration + test design in one head |

The rule of thumb: **sonnet for discovery and mapping; inherit for judgement and synthesis.** If this skill is ever run under Haiku or Sonnet as parent, the inherit-paths just get the same model as the cheap paths, which is a reasonable degradation — nothing breaks.

## The overall shape

You run five phases, mostly through subagents, with a single-agent consolidation between each. Do not collapse phases to save time — the separation between "broad scan," "rank," and "deep-dive" is what makes findings trustworthy.

```
Phase 1: Orient                     (1 subagent)
Phase 2: Broad parallel exploration (many subagents, parallel)
Phase 3: Consolidate & rank         (you, in the main thread)
Phase 4: Deep-dive selected items   (per item: 2 subagents in parallel + 1 verifier)
Phase 5: Write report & summarize   (you)
```

If at any point you decide there is nothing critical worth reporting, skip straight to Phase 5 and write a short "all-clear" report. Do not invent findings to fill space.

---

## Phase 1 — Orient

Before any exploration, get a factual picture of the project. Spawn **one** subagent:

- Tool: `Agent`
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: "Survey project for tech-debt audit"
- Prompt:
  ```
  Survey the project at the current working directory. Produce a FACTUAL summary only — no opinions, no recommendations, no tech-debt calls. Downstream work will do the judging.

  Return under 300 words covering:
  - Primary language(s), frameworks, build/test tooling
  - Top-level directory layout, one line per directory, noting rough size (file count or LOC bucket)
  - Test setup: framework, how to run, rough number of tests
  - CI/CD touchpoints visible from repo files
  - Key external dependencies (top 10 by apparent importance, not alphabetical)
  - CLAUDE.md / AGENTS.md / README highlights worth knowing for an audit
  - Rough age / activity signals (recent commits? long-dormant directories?)

  Just the facts. No judgement calls.
  ```

Keep this survey in working memory. You will quote from it when briefing downstream subagents so they don't have to rediscover the basics.

---

## Phase 2 — Broad parallel exploration

Dispatch **multiple subagents in parallel**, each focused on one dimension of tech debt. This is the widest cast of the net. The default dimensions are below — adapt them to the project (e.g., drop "type safety" for a Ruby project, add "SQL query sprawl" for a data-heavy Go service).

**Default dimensions** (dispatch a subagent per dimension, in a single message with many Agent calls):

1. **Architectural / structural debt** — god-modules, layer violations, mixed concerns, broken abstractions, circular dependencies, modules that "know too much."
2. **Duplication & coupling** — copy-pasted logic, parallel implementations of the same thing, tight coupling that should be an interface, hidden shared mutable state.
3. **Test quality & coverage gaps** — untested critical paths, tests that assert nothing useful, mocks that defeat the test's purpose, flaky-looking patterns, integration seams with no tests.
4. **Dead / vestigial code** — unreachable code, feature flags that never flip, commented-out blocks, obsolete migration paths, "legacy" modules still imported but unused.
5. **Error handling sanity** — silent try/except, swallowed errors, inconsistent error propagation, retries without backoff, missing error paths on boundaries.
6. **Dependency health** — outdated / deprecated / abandoned packages, version sprawl, security-relevant versions lagging, transitive dependency risk, multiple packages doing the same job.
7. **Configuration & environment sprawl** — config read from many places, env vars undocumented, dev/prod divergence, secrets handling smells.
8. **Performance & scalability hotspots** — N+1 queries, unbounded loops over user input, blocking calls on hot paths, obvious O(n²) where O(n) would do, cache invalidation messes.
9. **Type / contract safety** (if language supports it) — `any`/`Object`/`interface{}` leaks at module boundaries, stringly-typed APIs, missing schema validation at external edges.
10. **Security smells** — input reaching sinks without validation, auth checks that look bypassable, secrets in source, insecure defaults.

**Model for Phase 2 subagents:** `sonnet`. These are many, parallel, and mostly pattern-spotting — sonnet is the right cost/quality tradeoff.

**Subagent briefing template** (reuse for every dimension, substituting `{DIMENSION}` and `{GUIDANCE}`):

```
You are hunting for tech debt along ONE dimension: {DIMENSION}.

Project context (from prior survey):
{paste relevant portions of Phase 1 summary}

Your job:
- Scan the codebase looking specifically for issues under this dimension.
- Return a list of CANDIDATE debt items. Each candidate must include:
  * Short title
  * Location(s): file paths, and file:line for the clearest examples
  * Concrete evidence (1-3 quoted snippets or specific patterns you saw)
  * Your initial sense of severity (low / medium / high / critical) with a one-line why
  * Blast radius: how many other parts of the code appear to depend on this
  * Your confidence the finding is real (low / medium / high)
- Err on the side of surfacing candidates. Downstream ranking will filter. But do NOT fabricate - if the dimension has nothing, return an empty list and say so.
- Do NOT propose solutions. Do NOT rank across dimensions. Just report candidates with evidence.
- Do NOT edit code.
- Be specific. "Error handling is inconsistent" is useless. "`src/api/handlers/*.ts` mix throwing, returning Result types, and logging-and-returning-null - examples at handlers/users.ts:42, handlers/orders.ts:88" is useful.

Guidance specific to this dimension:
{GUIDANCE}

Return a structured list of candidates. Target 0-8 candidates. Under 600 words total.
```

Dispatch ALL dimension subagents in a **single message with many `Agent` tool calls in parallel** — do not serialize them. Use `subagent_type: "general-purpose"` and `model: "sonnet"` unless a dimension clearly needs deeper reasoning.

While these are running, you have nothing useful to do. Wait.

---

## Phase 3 — Consolidate & rank

When all Phase 2 subagents return, you do the consolidation yourself in the main thread. Do not dispatch a subagent for this — you need full context of all candidates.

### 3a. Consolidate

- Merge duplicates across dimensions (e.g., "god module" found under architectural and also under duplication).
- Group related candidates that are really one underlying issue presenting multiple symptoms.
- Drop candidates where the evidence is thin or the subagent's confidence was low.
- For each remaining candidate, note which dimension(s) flagged it.

### 3b. Rank by criticality

Score each candidate across these axes. Use qualitative bands (low / med / high) — no false precision.

- **Blast radius** — how much of the codebase touches or depends on this?
- **Velocity drag** — how much does this slow down new work *today*? (Evidence: awkward workarounds in recent code, frequent nearby changes, comments that complain.)
- **Risk of harm** — potential for correctness, security, or data-integrity bugs to ship because of this.
- **Compounding cost** — does this get meaningfully worse if left for another 6 months? (Not all debt does.)
- **Your confidence the problem is real** — be honest. This is the gate.

Write the scoring table in your working notes (not in the final report unless useful).

### 3c. Select up to 3

Apply the confidence bar: only select candidates where your confidence is **high**, and the combined severity (blast radius × risk × drag) clears a "this is materially hurting the project" threshold.

If fewer than 3 candidates clear the bar, select fewer. If zero candidates clear the bar, **select zero** and go to Phase 5 with an all-clear report. Do not lower the bar to fill slots. A short honest report is the goal.

Announce your selection to yourself briefly (as a note-to-self, not to the user — this is hands-off):
> Selected N items for deep-dive: [titles]. Rejected [count] candidates: [one-line reasons].

---

## Phase 4 — Deep-dive each selected item

For each selected item, run the following sub-phase. Items can be processed sequentially or in parallel at your discretion — parallel is faster but you need to manage context carefully.

### 4a. Parallel: scope + independent verification

Dispatch **two subagents in parallel** for the same item:

**Subagent A — Scope explorer** (`general-purpose`, `sonnet` — mechanical dependency tracing, sonnet is fine):
```
We have a candidate tech-debt finding. Your job: map its full scope.

Finding: {title}
Initial evidence: {paste evidence from Phase 2}

Trace this thoroughly:
- Every file/module that depends on the affected code.
- Every caller/consumer and how they use it.
- Data shapes that flow through; invariants implicitly assumed.
- Test files that currently cover it (file list, not full read).
- Any config, migrations, or infra touchpoints.
- How the affected code got this way if git history reveals it (one or two `git log -p` on the core files is fine).

Return under 500 words:
- Full touch-point list
- Invariants and contracts currently relied on
- Existing test coverage (honest read: is it real coverage or ceremony?)
- Anything surprising that would complicate a refactor

Do NOT propose solutions. Do NOT edit code.
```

**Subagent B — Devil's advocate / correctness check** (`general-purpose`, **omit `model` to inherit from parent** — this is the quality gate for the whole audit and deserves whatever reasoning power the parent is running with):
```
A prior pass flagged a tech-debt item. Your job is to CHALLENGE it independently.

Claim: {title}
Evidence offered: {paste evidence}
Claimed severity: {severity}
Claimed reasons it matters: {why}

Investigate skeptically:
- Is the evidence actually representative, or cherry-picked?
- Is there context (a comment, a design doc, a PR, a CLAUDE.md note) that justifies the current design?
- Is this a real problem or a stylistic preference dressed up as debt?
- Would a competent engineer familiar with this codebase agree it's debt, or push back?
- Are there reasons NOT to refactor this (e.g., stable interface, well-tested, about to be replaced anyway)?

Return a verdict under 400 words:
- CONFIRM (it's real debt, the evidence holds) / REFINE (real but the framing is off) / REJECT (not debt, here's why)
- Specific evidence for your verdict
- If CONFIRM: one or two points the original finding missed

Do NOT edit code.
```

Wait for both. Read both.

If Subagent B **REJECTS** with a compelling case: drop the item from the report. Pick an alternate from your ranked list if one qualifies, or accept a shorter report. Do not argue with the devil's advocate — its whole point is to catch your overconfidence.

If Subagent B **REFINES**: update your framing before proceeding.

If Subagent B **CONFIRMS**: proceed with Subagent A's scope as the foundation.

### 4b. Propose a solution

Now dispatch **one subagent** (`general-purpose`, **omit `model` to inherit from parent** — refactor design is the hardest synthesis in this skill: it has to hold scope, risk, migration path, and test design in one head. Worth the stronger model):

```
Design a refactoring plan for this tech-debt item.

Finding: {title}
Why it matters: {severity reasoning}
Full scope (from prior exploration): {paste Subagent A's output}
Verification notes (from devil's advocate): {paste Subagent B's output}

Produce a refactoring proposal with:

1. Target end-state — describe what the code should look like after, at the architectural level. No code.
2. Migration path — step-by-step sequence of moves to get from here to there. Prefer incremental, mergeable steps over a single big-bang change. Identify natural checkpoints where the code compiles and tests pass.
3. Size estimate — S / M / L / XL, with rough eng-days (honest order-of-magnitude: hours vs days vs weeks vs months).
4. Risk of breakage — low / medium / high. List the specific things most likely to break silently (wrong behavior, not compile errors).
5. Test design to safeguard the refactor — what characterization tests / integration tests / property tests / golden-file tests / manual verification would catch regressions? Be specific about which behaviors to pin down BEFORE starting the refactor.
6. Escape hatches — if the refactor goes sideways mid-way, how do we stop safely? (Feature flags, parallel implementations, branch-by-abstraction, etc.)

Do NOT write code. Do NOT edit files. Conceptual plan only.
Target 400-700 words.
```

### 4c. Solo verification pass (you)

Read the proposal yourself. Sanity-check:

- Does the migration path actually remove the debt, or just shuffle it?
- Is the test-design concrete enough that a future executor could follow it, or is it vague hand-waving?
- Is the size/risk estimate honest? Be especially suspicious of "Medium" — it's the lazy default.
- Is there a scenario where this refactor makes things *worse* (introduces a new abstraction that's the wrong shape)?

If anything feels off, dispatch a quick follow-up subagent with specific questions rather than papering over it.

---

## Phase 5 — Write the report

Compute the date/hour stamp: `YYYY-MM-DD_HH` in 24-hour local time (e.g., `2026-04-23_14`).

Create `./devk/` if it doesn't exist. Write the report to:

```
./devk/TECH_DEBT_REPORT_{YYYY-MM-DD_HH}.todo.md
```

Use this exact structure. Adapt the content to the findings, but keep the section shape stable so reports are comparable over time.

````markdown
# Tech Debt Audit — {YYYY-MM-DD HH:MM}

## Executive summary

{2-4 sentences. What's the overall state of the project? How many critical items were found? What's the single most important action, if any?}

**Verdict:** {one of: "All clear — no critical debt worth surfacing." / "N critical items identified; see below." / "1 critical item identified; see below."}

## How this audit was conducted

{3-5 sentences. Dimensions scanned. Rough count of candidates surfaced, how many were rejected in verification, how many made it to deep-dive. Confidence level overall.}

## Findings

{For each selected item, use the template below. If zero items, skip this section entirely.}

---

### Finding {N}: {Title}

**Criticality:** {Low / Medium / High / Critical}
**Why it's critical:** {2-3 sentences focused on impact — velocity, risk, compounding cost. Not code detail. Written for a human who does not want to read code.}

**Scope of the problem:**
{2-4 sentences describing what's affected and how broadly. High-level. Name modules/features, not line numbers.}

**Proposed solution:**
{3-6 sentences describing the target state and the migration path at a conceptual level. No code. Mention specific techniques (branch-by-abstraction, characterization tests, parallel implementations, etc.) where relevant.}

**Size of refactor:** {S / M / L / XL} — {rough eng-time estimate: "a few hours", "1-2 days", "1-2 weeks", "multi-week effort"}

**Risk of breaking something:** {Low / Medium / High}
{1-3 sentences about what specifically is most likely to break silently. Be concrete.}

**Test design to safeguard the refactor:**
- {Bullet list of specific tests to write BEFORE touching the code}
- {What existing behavior needs to be pinned down first}
- {What integration / characterization / property / golden-file tests to add}
- {What manual verification is unavoidable, if any}

**Confidence this is real debt:** {High / Very high} — {one sentence on why you're confident, including the independent verification result}

**Evidence (for reference):**
- {file:line references, brief quotes or pattern descriptions}
- {anything else useful for whoever executes the refactor}

---

{repeat for each finding}

## Items considered but rejected

{Optional section. If you rejected candidates in verification, briefly list them here — title + one-line reason. Keeps the audit honest and helps future audits avoid re-raising the same things. Skip entirely if nothing was rejected.}

## What was NOT audited

{One short paragraph listing anything you explicitly could not cover — e.g., "runtime behavior in production", "infra/terraform which is in a separate repo", "frontend performance which needs browser profiling". Manages expectations.}
````

### Notes on writing the report well

- **Write for impact, not mechanics.** The human reads this to decide "do we refactor X?" Help them decide. Don't make them reconstruct your reasoning.
- **Be specific about size and risk.** "Medium" with no concrete claim is useless. Claim something falsifiable: "~3 engineer-days; the risky part is the cache-invalidation change in `billing/cache.ts`."
- **Test design is the most important section after Criticality.** A refactor with no test-design plan is a refactor that will regress silently. Treat this section as the safeguard that makes the proposal actually actionable.
- **File:line references belong in the Evidence block**, not in the prose. The prose is for a human who wants impact, not implementation.

---

## Phase 5b — In-chat summary

After writing the report, return a **short** summary to the user (who may or may not be present). Keep it under 10 lines:

- Verdict (all-clear vs. N findings)
- One-line per finding: title, criticality, rough size
- Path to the full report
- Optional: one line on what was notably *healthy* about the project, if anything stood out positively

That's the end of the run. Do not propose to start executing any of the refactors — this skill is proposal-only. If the human wants to act on a finding, they'll trigger `devk-brainstorm` or `devk-implementing` on it separately.

---

## Hard rules

- **No code edits.** The only file you write is the report under `./devk/`.
- **No partial execution.** Do not "start fixing the easy one while you're here." That's a different skill.
- **No fabricated findings.** If evidence is thin or the devil's-advocate rejected it, drop it.
- **Every finding must have run through independent verification.** No exceptions. The whole point of this skill is that the report is trustworthy without human code review.
- **Subagents do not spawn further subagents.** Keep the dispatch tree flat — main agent spawns, subagents report.
- **Keep the report free of internal dispatch chatter.** The human reads findings, not your workflow.

## When this skill is a bad fit

- The user wants code changes now. → Use `devk-brainstorm` or `devk-implementing`.
- The user wants a review of a specific PR or recent commits. → Use `code-review` or `devk-reviewing-section`.
- The user wants a security audit specifically. → This skill touches security smells but is not a replacement for a dedicated security review.
- The project is empty or near-empty. → There is no meaningful debt to find. Return an all-clear report quickly.

## Reminders for yourself

- Hands-off means **hands-off**. No clarifying questions. You were given enough context by the trigger.
- High confidence bar. Empty findings beat low-confidence findings.
- Liberal subagents. Parallel where independent. Independent verification is non-negotiable.
- The report is the product. Everything else is scaffolding.
