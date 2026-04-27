---
description: code-level perf and reliability smells (hybrid report — quick wins + max 3 structural)
---

# Lens: Performance & Reliability — code-level smells

## Scope

Audit the codebase for performance and reliability issues that are visible from reading code and config — no profilers, no load tests, no production telemetry. Surface a **hybrid report**: up to **3** structural findings (each with verification, like tech-debt) plus a flat list of **quick wins** (capped at ~15) that the user can act on in minutes.

This lens cares about:

**Perf dimensions (5)**

1. **Hot-path I/O** — blocking I/O on hot paths, sync calls in async contexts, file/network/DB calls inside loops, missing streaming where the data is large.
2. **Query patterns** — N+1, missing pagination, over-fetching, missing batching, missing indexes that are visible from query shape, repeated identical queries within one request.
3. **Render & bundle** (frontend) — heavy components without memoization where it matters, large or unsplit bundles, synchronous expensive work in render, unoptimized images, web fonts loaded synchronously, render-blocking scripts.
4. **Allocation & loop hygiene** — O(n²) where O(n) works, unbounded growth (lists/maps that only ever grow), expensive object creation in tight loops, work re-done that could be memoized at module/request scope.
5. **Async/parallelism gaps** — sequential awaits that could be `Promise.all` / `gather` / `errgroup`, missing batching at fan-out points, missing concurrency limits where unbounded parallelism would overwhelm a downstream.

**Reliability dimensions (5)**

6. **Timeouts & cancellation** — network/DB/RPC calls without timeouts, missing AbortController / cancellation tokens, unbounded waits, `Promise` never-settles patterns.
7. **Error handling** — swallowed errors (`catch {}`), catch-and-log-and-return-null, errors crossing boundaries without context, mixed error styles within one module (throw vs Result vs null).
8. **Retries & backoff** — missing retries on transient failures of idempotent operations, retries without backoff/jitter, retries on operations that aren't idempotent (worse than no retry), missing retry budgets / max attempts.
9. **Idempotency & ordering** — endpoints/jobs that mutate state without idempotency keys, race conditions on shared state, write-after-read patterns without locks/CAS, queue consumers that assume exactly-once.
10. **Input validation & resource cleanup** — missing validation at trust boundaries (HTTP handlers, queue consumers, file uploads), resource leaks (connections/files/timers/subscriptions not closed on the error path), unbounded queues/buffers/caches.

This lens does NOT cover:

- Observability gaps — logging, tracing, metrics, SLOs, alerts, dashboards. (Possibly a future lens.)
- Capacity planning, deploy safety, runbooks, on-call readiness.
- Security correctness — covered by the security smell dimension of `tech-debt`.
- Anything that needs a profiler, load test, or production telemetry to verify. The lens stays static-only.

## The quality bar (read this first)

This lens has two tiers, with two different bars:

- **Structural findings** are like tech-debt: high-confidence, max 3, devil's advocate verification mandatory. The cost of being wrong on a structural finding is high (days to weeks of refactor wasted).
- **Quick wins** have a lower bar but still a bar: the fix must be genuinely tiny (single-spot, mechanical, <1hr) AND the benefit must be real and verifiable from the code. "You could technically refactor X" papercuts do not belong in either tier.

Empty sections are honest. "No structural findings cleared the bar" or "No quick wins worth flagging" are valid outcomes — say so plainly. Padding either tier kills the report's signal.

## Phase 1 — Scout

Dispatch **one** subagent (`sonnet`) for a factual project survey. No opinions, no smell calls.

Prompt:
```
Survey the project at the current working directory. Produce a FACTUAL summary only — no opinions, no recommendations.

Return under 300 words covering:
- Primary language(s), runtime, framework(s), build/test tooling
- Project shape: is there a frontend? backend service? background workers? CLI? library? data pipeline? Note which.
- Where I/O happens: which modules talk to DB, HTTP, filesystem, message queues, third-party APIs
- Concurrency model: async/await? threads? event loop? worker pool? sync only?
- HTTP/RPC boundaries: where does external input enter? where do external calls leave?
- Data layer: ORM / query builder / raw SQL? caching layer? connection pooling visible?
- Frontend rendering (if any): SSR / SSG / SPA / hybrid? state management? bundler?
- Test setup: framework, types of tests (unit/integration/e2e), how to run
- Anything in CLAUDE.md / AGENTS.md / README about perf or reliability requirements (SLAs, latency budgets, throughput targets)

Just the facts. No judgement calls.
```

Quote the relevant pieces of this scout when briefing Phase 2 subagents — don't make them rediscover the basics.

## Phase 2 — Parallel dimension fan-out

Dispatch **one subagent per dimension in a single message with parallel `Agent` calls**. Up to 10 subagents (5 perf + 5 reliability). All `sonnet`.

**Adaptation:** drop dimensions that don't apply. CLI with no DB → skip "query patterns". Backend with no frontend → skip "render & bundle". Pure-function library with no I/O → may skip most dimensions. Don't dispatch a subagent with no realistic target.

**Per-dimension subagent template** (substitute `{DIMENSION}` and `{GUIDANCE}`):

```
You are hunting along ONE dimension: {DIMENSION}.

Project context (from prior survey):
{paste relevant scout output}

Find candidate issues. For each, return:
- Short title
- Location(s): file paths, file:line for clearest examples
- Concrete evidence (1–3 quoted snippets or specific patterns you saw)
- Severity guess: low / medium / high / critical, one-line why
- Blast radius: how many call sites / requests / jobs / users are affected
- Tier hint: "quick win" (single-spot, mechanical fix, <1hr)
              or "structural" (touches multiple files / changes a contract / >1 day)
- Confidence the finding is real: low / medium / high

Rules:
- Err toward surfacing candidates — downstream ranking filters.
- Do NOT fabricate. An empty result is fine; say so explicitly if the dimension has nothing.
- Do NOT propose fixes for structural candidates (a deep dive does that later).
- For quick-win candidates, a ONE-LINE fix sketch is OK ("add timeout: 5s to fetch call").
- Do NOT edit code.
- Be specific. "Some queries look slow" is useless. "users.findAll() is called inside the orders loop at handlers/orders.ts:42 with no eager-load — N+1 across the response" is useful.

Guidance specific to this dimension:
{GUIDANCE}

Target 0–8 candidates. Under 600 words.
```

Wait for all dimension subagents to return.

## Consolidate & rank (in main thread, you)

Do this yourself — full context across all dimensions is needed.

1. **Merge duplicates** across dimensions (a `fetch` with no timeout shows up under both "timeouts & cancellation" and "hot-path I/O").
2. **Group symptoms of one underlying issue.** Five swallowed-error sites in one module is one finding, not five.
3. **Drop thin-evidence and low-confidence candidates.**
4. **Split surviving candidates into two pools by tier hint:** structural pool, quick-wins pool. (You can override a subagent's tier hint if the evidence convinces you it's wrong — e.g. a "quick win" that actually touches 30 files is structural.)
5. **Score structural candidates** across qualitative bands (low/med/high) on:
   - **Blast radius** — how many call sites / requests / jobs are affected today
   - **User-visible impact** — latency, errors users see, dropped work, data loss potential
   - **Compounding cost** — does this get materially worse with traffic / feature growth?
   - **Confidence the problem is real** — be honest. This is the gate.
   
   **Select up to 3** that clear the bar (high confidence AND combined severity says "this is materially hurting the system"). Fewer than 3 is fine. If zero clear the bar, write all-clear for the structural section and skip Phase 3a entirely.
6. **Score quick-wins.** Lower bar, but still a bar. Drop "you could technically do X" papercuts. Keep wins where the cost-to-fix is genuinely tiny AND the benefit is real and visible from the code (e.g. "third-party HTTP call at `billing/stripe.ts:88` has no timeout — request can hang indefinitely; add `timeout: 5000`"). **Cap at ~15.** If more clear the bar, keep the highest-leverage 15 and note the trim in the report.

Keep your scoring notes — they don't go in the report unless useful.

## Phase 3 — Two paths

### Path A — Structural deep dive (per selected structural finding, max 3)

For each selected structural finding, run two subagents in parallel, then a third.

**3a. Parallel: scope explorer + devil's advocate**

**Subagent A — Scope explorer** (`sonnet`):
```
We have a candidate perf/reliability finding. Your job: map its full scope.

Finding: {title}
Initial evidence: {paste evidence}

Trace this thoroughly:
- Every file/module that depends on the affected code.
- Every caller/consumer and how they use it.
- Data shapes and invariants implicitly assumed.
- Test files that currently cover the behavior (file list, not full read).
- Any config, migrations, infra touchpoints.
- Brief git history on the core files if useful (one or two `git log -p`).

Return under 500 words:
- Full touch-point list
- Invariants and contracts currently relied on
- Existing test coverage (honest read: real coverage or ceremony?)
- Anything surprising that would complicate a refactor

Do NOT propose solutions. Do NOT edit code.
```

**Subagent B — Devil's advocate** (omit `model` to inherit from parent — this is the quality gate):
```
A prior pass flagged a perf/reliability item. Your job is to CHALLENGE it independently.

Claim: {title}
Evidence offered: {paste evidence}
Claimed severity: {severity}
Claimed reasons it matters: {why}

Investigate skeptically:
- Is the evidence representative, or cherry-picked?
- Is there context (a comment, design doc, PR, CLAUDE.md note) justifying the current design?
- For perf: is this actually on a hot path, or hypothetically slow but rarely hit?
- For reliability: is the failure mode actually reachable, or is the "missing timeout" on a call that always responds in <100ms?
- Would a competent engineer familiar with this codebase agree, or push back?
- Are there reasons NOT to change it (stable interface, well-tested, scheduled for replacement)?

Return a verdict under 400 words:
- CONFIRM (real, evidence holds) / REFINE (real but framing is off) / REJECT (not a real issue, here's why)
- Specific evidence for your verdict
- If CONFIRM: one or two points the original finding missed

Do NOT edit code.
```

Wait for both. Read both.

- **REJECT with a compelling case:** drop the item. **Do not promote alternates** to fill the slot — accept a shorter report. The discipline is what makes the structural section trustworthy.
- **REFINE:** update your framing before proceeding.
- **CONFIRM:** proceed using Subagent A's scope as the foundation.

**3b. Refactor proposal** (omit `model` to inherit — must hold scope, risk, migration, verification, and test design in one head):

```
Design a refactoring plan for this perf/reliability finding.

Finding: {title}
Why it matters: {severity reasoning}
Full scope (from prior exploration): {paste Subagent A's output}
Verification notes (from devil's advocate): {paste Subagent B's output}

Produce a proposal with:

1. Target end-state — what the code should look like after, at the architectural level. No code.
2. Migration path — step-by-step sequence to get from here to there. Prefer incremental, mergeable steps. Identify natural checkpoints where the code compiles and tests pass.
3. Size estimate — S / M / L / XL with rough eng-days (honest: hours vs days vs weeks).
4. Risk of breakage — low / medium / high. List specific things most likely to break SILENTLY (wrong behavior, not compile errors).
5. How to verify it actually helps — what to MEASURE, load-test, or characterize before/after to confirm the change has the intended effect. For perf: which metric, on which path, with what input shape. For reliability: which failure mode to inject (chaos / fault injection / unit-level error simulation), and what the system should do.
6. Test design to safeguard the refactor — characterization tests, integration tests, property tests, golden-file tests. Be specific about which behaviors to pin down BEFORE starting.
7. Escape hatches — if the refactor goes sideways mid-way, how do we stop safely? (Feature flags, parallel implementations, branch-by-abstraction.)

Do NOT write code. Do NOT edit files. Conceptual plan only.
Target 400–700 words.
```

**3c. Solo sanity-check** (you):
- Does the migration path actually remove the issue, or just shuffle it?
- Is the verification plan concrete enough that a future executor knows what to measure or inject? Or hand-waving?
- Is the test design specific, or vague?
- Is the size/risk estimate honest? Be especially suspicious of "Medium" — it's the lazy default.
- Could this refactor make things *worse* (introduces a new abstraction in the wrong shape; pessimizes a different path)?

If anything feels off, dispatch a quick follow-up subagent with specific questions.

### Path B — Quick-wins verification pass

**One subagent** (`sonnet`), given the consolidated quick-wins list:

```
You are auditing a list of proposed quick-win perf/reliability fixes. For each item:

- Verify the file path exists and the cited line(s) match the claim. (Use Read on the relevant span.)
- Verify the proposed one-line fix sketch is reasonable for the code at that location.
- Flag any item where the cited evidence is wrong, the fix wouldn't actually help, or the fix would break something at the call site (e.g. "add timeout 5s" on a call that legitimately needs to wait longer).

Return the list with each item marked KEEP / TRIM (with note on what to adjust) / DROP (with reason).

Do NOT edit code. Under 800 words.
```

Incorporate the verification into the report — DROPs vanish from the report, TRIMs update the wording, KEEPs go in as-is.

**Why one pass instead of per-item devil's advocate:** quick wins are individually low-stakes — a wrong "add timeout: 5s" suggestion costs the user a minute to evaluate. Per-item devil's advocate would be wildly out of proportion. One sanity-pass is enough.

### Maximum-mode addition

For the **top structural item only**, also dispatch a "measurement design" subagent (`sonnet`) that drafts what to instrument or load-test before/after to prove the refactor worked, in concrete terms.

For **quick-wins**, expand the pass into a "patch sketch" subagent that drafts the actual code change as a snippet (no file edits) for each KEPT item, so the reader can paste it in.

## Finding shape

Two finding sections in the report. Other sections (header, Verdict, TL;DR, "Stuff that didn't fit", "What I looked at") match `SKILL.md`.

### Structural findings (top section, max 3)

```markdown
### Finding {N}: {Title}

**Criticality:** Low / Medium / High / Critical
**Flavor:** Perf / Reliability / Both
**Why it's critical:** 2–3 sentences focused on user / system impact (latency, errors, lost work, data risk). Not code detail.

**Scope of the problem:**
2–4 sentences naming modules/features and how broadly affected.

**Proposed solution:**
3–6 sentences describing target state and migration path conceptually. No code. Mention specific techniques (branch-by-abstraction, characterization tests, parallel implementations) where relevant.

**Size of refactor:** S / M / L / XL — rough eng-time ("a few hours", "1–2 days", "1–2 weeks", "multi-week effort")

**Risk of breaking something:** Low / Medium / High
1–3 sentences about what specifically is most likely to break silently. Be concrete.

**How to verify it actually helps:**
What to measure, load-test, or inject to confirm the change has the intended effect. For perf: metric, path, input shape. For reliability: failure mode and expected system response.

**Test design to safeguard the refactor:**
- Specific tests to write BEFORE touching the code
- What existing behavior to pin down first
- Integration / characterization / property / golden-file tests to add
- What manual verification is unavoidable, if any

**Confidence this is real:** High / Very high — one sentence including the devil's-advocate verdict.

**Evidence (for reference):**
- file:line refs, brief quotes, pattern descriptions
```

### Quick-wins (bottom section, ~15 cap, split into Perf and Reliability subsections)

Each entry is tight — meant to be skimmed:

```markdown
#### {Short title}
- **Where:** `path/to/file.ext:42` (and N similar sites if it's a pattern)
- **Issue:** what's wrong, in one sentence
- **Fix:** one-line concrete change ("add `timeout: 5000` to the `fetch` options")
- **Why it matters:** one line — what hangs / loops / fails today
```

Sort within each subsection by impact (highest first), not by file. Reader should be able to read the top 5 and act on them without thinking.

In the report's `Stuff that didn't fit` section, list candidates that were considered and rejected (structural REJECTs and quick-wins DROPped in verification) — title plus one-line reason. Keeps the audit honest.

In `What I looked at`, include a one-paragraph "What was NOT audited" — anything you couldn't cover statically (e.g. runtime-only perf characteristics, infra in another repo, frontend perf needing browser profiling). Manages expectations.

## Notes on writing the report well

- **Write for impact, not mechanics.** The reader decides "is this worth my Tuesday?" Help them decide. Don't make them reconstruct your reasoning.
- **Be specific about size, risk, and verification.** "Medium" with no concrete claim is useless. "~3 engineer-days; the risky part is the queue-consumer idempotency change in `jobs/process.ts`; verify by injecting duplicate messages and confirming no double-charges" is useful.
- **Verification design is the structural section's most important field after Criticality.** A perf/reliability change without a measurement plan is one that may regress invisibly.
- **Quick wins should be paste-and-go.** If the reader can't act on a quick win in a minute, it isn't one — re-classify or drop.
- **File:line refs belong in Evidence**, not in the prose.

## Lens-specific hard rules

In addition to the parent `SKILL.md` hard rules:

- **Static analysis only.** No running profilers, no load tests, no executing the project. Every finding must be groundable in code/config the lens can read.
- **Max 3 structural findings.** Same dilution argument as tech-debt.
- **Independent verification mandatory for every structural finding.** No item enters the structural section without CONFIRM (or REFINE) from a devil's advocate. Quick wins use the lighter single-pass verification — never per-item devil's advocate.
- **Quick wins capped at ~15.** If more clear the bar, keep the highest-leverage 15 and note in the report that you trimmed.
- **Empty sections are honest.** "No structural findings cleared the bar" or "No quick wins worth flagging" are valid outcomes. Write them plainly.
- **No partial execution.** Don't fix the easy quick-wins yourself "while you're here." Different skill.
- **Perf claims need verification design.** Every structural perf finding must include the "How to verify it actually helps" section. A perf refactor with no measurement plan is one that may regress invisibly.
- **Subagents do not spawn further subagents.** Flat dispatch tree.

## When this lens is a bad fit

- The user wants the broader code-quality sweep → `tech-debt`.
- The user wants observability / ops review → out of scope for this lens.
- The user wants it fixed now → `devk-brainstorm`.
- The user wants a security audit → `tech-debt` touches security smells; this lens does not.
- The project has no I/O, no concurrency, no user-facing perf concerns (e.g. a pure-function library) → write a quick all-clear and stop.
