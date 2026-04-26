---
description: high-confidence refactoring proposals (max 3, with test design)
---

# Lens: Tech debt — high-confidence refactoring proposals

## Scope

Survey the codebase for tech debt, then surface up to **3** items the user should genuinely act on. Each surfaced item gets a refactoring proposal with size, risk, and test design.

This lens cares about:

1. **Architectural / structural debt** — god-modules, layer violations, mixed concerns, broken abstractions, circular dependencies, modules that "know too much."
2. **Duplication & coupling** — copy-pasted logic, parallel implementations of the same thing, tight coupling that should be an interface, hidden shared mutable state.
3. **Test quality & coverage gaps** — untested critical paths, tests that assert nothing useful, mocks that defeat the test's purpose, flaky-looking patterns, integration seams with no tests.
4. **Dead / vestigial code** — unreachable code, feature flags that never flip, commented-out blocks, obsolete migration paths, "legacy" modules still imported but unused.
5. **Error handling sanity** — silent try/except, swallowed errors, inconsistent error propagation, retries without backoff, missing error paths on boundaries.
6. **Dependency health** — outdated / deprecated / abandoned packages, version sprawl, security-relevant versions lagging, transitive dependency risk.
7. **Configuration & environment sprawl** — config read from many places, env vars undocumented, dev/prod divergence, secrets handling smells.
8. **Performance & scalability hotspots** — N+1 queries, unbounded loops over user input, blocking calls on hot paths, obvious O(n²) where O(n) would do.
9. **Type / contract safety** (if language supports it) — `any`/`Object`/`interface{}` leaks at module boundaries, stringly-typed APIs, missing schema validation at external edges.
10. **Security smells** — input reaching sinks without validation, auth checks that look bypassable, secrets in source, insecure defaults.

This lens does NOT cover:
- Code changes (proposal-only, like every lunch-break lens)
- A formal security audit (touches security smells but is not a substitute)
- PR-level review of recent commits — `code-review` skill is for that

## The quality bar (read this first)

This lens is different from other lunch-break lenses because **the cost of being wrong is high**: a confidently-wrong refactor proposal can waste weeks. Internalize:

- **High confidence is mandatory.** Only surface items you would stake your reputation on. If you're unsure whether something is debt, it isn't — or it doesn't belong in this report.
- **"All clear" is an honorable outcome.** If nothing passes the bar, write a short all-clear report and stop. Padding with borderline concerns is worse than empty.
- **Max 3 items.** Even on a debt-ridden codebase, more than 3 dilutes attention. Pick the 3 most critical.
- **Independent verification is required.** Every surfaced item must survive a devil's-advocate pass before it earns deep-dive treatment. No exceptions — this is what makes the report trustworthy without human code review.
- **The reader does not want to read code.** Write for impact, cost, and risk. File:line refs go in an Evidence block, not the prose.

## Phase 1 — Scout (orient on the project)

Dispatch **one** subagent (`sonnet`) to produce a factual project survey. No opinions, no debt calls — downstream work judges.

Prompt:
```
Survey the project at the current working directory. Produce a FACTUAL summary only — no opinions, no recommendations, no tech-debt calls.

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

Keep this in working memory — quote relevant pieces when briefing later subagents so they don't rediscover the basics.

## Phase 2 — Broad parallel exploration (one subagent per dimension)

Dispatch **one subagent per dimension in a single message with parallel `Agent` calls**. Use the 10 dimensions in Scope, adapting to the project (drop "type safety" for Ruby; add a project-specific dimension like "SQL query sprawl" for a data-heavy service).

Model: `sonnet`. These are many, parallel, mostly pattern-spotting.

**Per-dimension subagent template** (substitute `{DIMENSION}` and `{GUIDANCE}`):

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
  * Initial sense of severity (low / medium / high / critical) with a one-line why
  * Blast radius: how many other parts of the code appear to depend on this
  * Your confidence the finding is real (low / medium / high)
- Err on the side of surfacing candidates — downstream ranking will filter. Do NOT fabricate. If the dimension has nothing, return an empty list and say so.
- Do NOT propose solutions. Do NOT rank across dimensions. Do NOT edit code.
- Be specific. "Error handling is inconsistent" is useless. "`src/api/handlers/*.ts` mix throwing, returning Result types, and logging-and-returning-null — examples at handlers/users.ts:42, handlers/orders.ts:88" is useful.

Guidance specific to this dimension:
{GUIDANCE}

Target 0–8 candidates. Under 600 words total.
```

Wait for all dimension subagents to return.

## Consolidate & rank (in main thread, you)

Do this yourself — full context of all candidates is needed.

1. **Merge duplicates** across dimensions (e.g. "god module" found under both architectural and duplication).
2. **Group related candidates** that are really one underlying issue presenting multiple symptoms.
3. **Drop thin-evidence or low-confidence candidates.**
4. **Score remaining candidates** across qualitative bands (low / med / high) — no false precision:
   - **Blast radius** — how much of the codebase touches this?
   - **Velocity drag** — how much does this slow down new work *today*? (Evidence: awkward workarounds in recent code, frequent nearby changes, comments that complain.)
   - **Risk of harm** — potential for correctness, security, or data-integrity bugs to ship because of this.
   - **Compounding cost** — does this get materially worse if left for another 6 months? (Not all debt does.)
   - **Your confidence the problem is real** — be honest. This is the gate.
5. **Select up to 3** items. Apply the bar: only candidates where confidence is **high** AND combined severity (blast radius × risk × drag) clears "this is materially hurting the project."

If fewer than 3 clear the bar, select fewer. **If zero clear the bar, select zero** and skip Phase 3 — go straight to writing an all-clear report. Do not lower the bar to fill slots.

Keep your scoring table in working notes. It does not need to appear in the report unless useful.

## Phase 3 — Per-item deep dive

For each selected item, run two subagents in parallel, then a third.

### 3a. Parallel: scope explorer + devil's advocate

**Subagent A — Scope explorer** (`sonnet` — mechanical dependency tracing):
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
- Existing test coverage (honest read: real coverage or ceremony?)
- Anything surprising that would complicate a refactor

Do NOT propose solutions. Do NOT edit code.
```

**Subagent B — Devil's advocate** (omit `model` to inherit from parent — this is the quality gate for the whole audit and deserves the strongest reasoning available):
```
A prior pass flagged a tech-debt item. Your job is to CHALLENGE it independently.

Claim: {title}
Evidence offered: {paste evidence}
Claimed severity: {severity}
Claimed reasons it matters: {why}

Investigate skeptically:
- Is the evidence actually representative, or cherry-picked?
- Is there context (a comment, design doc, PR, CLAUDE.md note) that justifies the current design?
- Is this a real problem or a stylistic preference dressed up as debt?
- Would a competent engineer familiar with this codebase agree it's debt, or push back?
- Are there reasons NOT to refactor (e.g. stable interface, well-tested, about to be replaced anyway)?

Return a verdict under 400 words:
- CONFIRM (real debt, evidence holds) / REFINE (real but framing is off) / REJECT (not debt, here's why)
- Specific evidence for your verdict
- If CONFIRM: one or two points the original finding missed

Do NOT edit code.
```

Wait for both. Read both.

- **REJECT with a compelling case:** drop the item. Either pick an alternate from your ranked list if one qualifies, or accept a shorter report. Do not argue with the devil's advocate — its whole point is to catch overconfidence.
- **REFINE:** update your framing before proceeding.
- **CONFIRM:** proceed using Subagent A's scope as the foundation.

### 3b. Refactor proposal

Dispatch **one subagent** (omit `model` to inherit — refactor design must hold scope, risk, migration path, and test design in one head):

```
Design a refactoring plan for this tech-debt item.

Finding: {title}
Why it matters: {severity reasoning}
Full scope (from prior exploration): {paste Subagent A's output}
Verification notes (from devil's advocate): {paste Subagent B's output}

Produce a refactoring proposal with:

1. Target end-state — describe what the code should look like after, at the architectural level. No code.
2. Migration path — step-by-step sequence to get from here to there. Prefer incremental, mergeable steps over one big-bang change. Identify natural checkpoints where the code compiles and tests pass.
3. Size estimate — S / M / L / XL with rough eng-days (honest order-of-magnitude: hours vs days vs weeks vs months).
4. Risk of breakage — low / medium / high. List the specific things most likely to break silently (wrong behavior, not compile errors).
5. Test design to safeguard the refactor — what characterization tests / integration tests / property tests / golden-file tests / manual verification would catch regressions? Be specific about which behaviors to pin down BEFORE starting.
6. Escape hatches — if the refactor goes sideways mid-way, how do we stop safely? (Feature flags, parallel implementations, branch-by-abstraction.)

Do NOT write code. Do NOT edit files. Conceptual plan only.
Target 400–700 words.
```

### 3c. Solo sanity-check (you)

Read the proposal. Sanity-check:
- Does the migration path actually remove the debt, or just shuffle it?
- Is the test-design concrete enough that a future executor could follow it, or vague hand-waving?
- Is the size/risk estimate honest? Be especially suspicious of "Medium" — it's the lazy default.
- Could this refactor make things *worse* (introduces a new abstraction in the wrong shape)?

If anything feels off, dispatch a quick follow-up subagent with specific questions rather than papering over it.

## Finding shape (extends the default per-finding template)

Replace the default per-finding template with this richer one. Keep the rest of the report structure (header, Verdict, TL;DR, Findings section, Stuff that didn't fit, What I looked at) as defined in `SKILL.md`.

```markdown
### Finding {N}: {Title}

**Criticality:** Low / Medium / High / Critical
**Why it's critical:** 2–3 sentences focused on impact — velocity, risk, compounding cost. Not code detail.

**Scope of the problem:**
2–4 sentences describing what's affected and how broadly. Name modules/features, not line numbers.

**Proposed solution:**
3–6 sentences describing target state and migration path conceptually. No code. Mention specific techniques (branch-by-abstraction, characterization tests, parallel implementations) where relevant.

**Size of refactor:** S / M / L / XL — rough eng-time ("a few hours", "1–2 days", "1–2 weeks", "multi-week effort")

**Risk of breaking something:** Low / Medium / High
1–3 sentences about what specifically is most likely to break silently. Be concrete.

**Test design to safeguard the refactor:**
- Specific tests to write BEFORE touching the code
- What existing behavior needs to be pinned down first
- What integration / characterization / property / golden-file tests to add
- What manual verification is unavoidable, if any

**Confidence this is real debt:** High / Very high — one sentence including the devil's-advocate verdict.

**Evidence (for reference):**
- file:line refs, brief quotes, pattern descriptions
- anything else useful for whoever executes the refactor
```

In the report's `Stuff that didn't fit` section, list candidates that were considered and rejected — title plus a one-line reason. Keeps the audit honest and helps future audits avoid re-raising the same things. Skip the section entirely if nothing was rejected.

In `What I looked at`, include a one-paragraph "What was NOT audited" — anything you explicitly couldn't cover (runtime behavior in production, infra in a separate repo, frontend perf needing browser profiling). Manages expectations.

## Notes on writing the report well

- **Write for impact, not mechanics.** The reader decides "do we refactor X?" Help them decide. Don't make them reconstruct your reasoning.
- **Be specific about size and risk.** "Medium" with no concrete claim is useless. Claim something falsifiable: "~3 engineer-days; the risky part is the cache-invalidation change in `billing/cache.ts`."
- **Test design is the most important section after Criticality.** A refactor with no test-design plan is one that will regress silently. Treat this as the safeguard that makes the proposal actionable.
- **File:line refs belong in Evidence**, not in the prose.

## Lens-specific hard rules

These are in addition to the parent SKILL.md hard rules:

- **Independent verification is mandatory for every surfaced finding.** No item enters the report without a CONFIRM (or REFINE) from a devil's advocate.
- **Max 3 findings.** Even on a debt-ridden project, dilution kills the report.
- **No partial execution.** Do not "start fixing the easy one while you're here." Different skill.
- **Subagents do not spawn further subagents.** Keep the dispatch tree flat — main agent spawns, subagents report.

## When this lens is a bad fit

- The user wants code changes now → `devk-brainstorm`.
- The user wants a review of a specific PR or recent commits → `code-review`.
- The user wants a security audit specifically → this lens touches security smells but is not a substitute.
- The project is empty or near-empty → no meaningful debt to find. Write a quick all-clear and stop.
