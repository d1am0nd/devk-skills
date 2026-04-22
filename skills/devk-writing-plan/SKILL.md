---
name: devk-writing-plan
description: Use after devk-writing-spec has finished and the spec is user-approved. This skill converts .devk/spec.md into an executable plan at .devk/plan.md - work divided into self-contained sections, each with TDD steps pre-written, explicit dependencies, and parallel-safe groups marked. Output is ready to be consumed by devk-executing-plan which dispatches sections to subagents. Presents the plan for the user's final approval before execution begins.
---

# devk-writing-plan — Turn the approved spec into an executable plan

Spec is approved. Now produce a plan that `devk-executing-plan` can execute section-by-section, potentially with parallel subagents. This skill is about **structure and execution order**, not re-designing the spec.

## Core principles

- **Quality over speed. No shortcuts.** If a section can't be done properly in one bite, split it. Don't cram.
- **Each section must be self-contained.** A fresh subagent with just that section's text should be able to do it. That means: context the section needs is explicit in its description, not "see section 3."
- **TDD per section.** Each section lists what tests to write first. If a section can't have tests (e.g., pure config change), say so and justify.
- **Parallelize honestly.** Mark sections parallel-safe only when they truly don't touch overlapping files or state. False parallelism creates merge conflicts and worse bugs.
- **Sections should be right-sized.** A section is one focused chunk of work — not "implement the whole feature." If a section would take a subagent more than ~30 minutes of work or touch more than ~5-10 files, split it.

## Inputs

- `.devk/spec.md` (approved)
- `.devk/requirements.md` (context)
- The project itself — you may need to scan parts of it to size sections correctly.

## Plan structure

Write to `.devk/plan.md`:

```markdown
# Plan: <feature name>

## Overview
<3-5 sentences: the order of operations at a high level. Why this order?>

## Execution graph
<ASCII diagram or simple list showing dependencies. Example:>

  [S1: schema] → [S2: model] → [S3a: API] ∥ [S3b: UI] → [S4: wiring]

Sections in the same parallel group are separated by `∥`.

## Sections

### S1: <section title>

**Goal:** <one sentence>

**Depends on:** <prior sections by ID, or "none">
**Parallel-safe with:** <sibling section IDs, or "none">

**Files touched (anticipated):** <list or glob — approximate is fine, subagent may refine>

**Tests to write first (TDD):**
- <test name / scenario> — expects: <observable behavior>
- <test name / scenario> — expects: <...>

**Implementation outline:**
- <bullet 1 — what the implementation needs to do>
- <bullet 2>

**Acceptance criteria:**
- <observable thing the reviewer will check>

**Notes for the section agent:**
<anything non-obvious — e.g., "use the existing `X` helper, don't re-invent", or "this file has a linter rule about Y">

---

### S2: ...

<same structure>

---

## Material decisions (carried from spec)
<Repeat the "Decisions and rationale" from the spec so the execution agents see them without re-reading the spec. One line each.>

## Risks / open questions
<Anything from the spec that might bite during execution. Include mitigations.>
```

## How to size and split sections

Good sections are vertical slices, not horizontal layers:
- ✅ **Good:** "S3: User registration endpoint — DB insert, validation, response shape, tests for the full path"
- ❌ **Bad:** "S1: All DB changes, S2: All controllers, S3: All UI, S4: Wiring" (creates integration risk at the end; nothing works until S4)

Exceptions to the vertical-slice rule:
- **Shared foundation** (e.g., new DB schema, new shared type) that multiple sections depend on — make it S1, then downstream sections parallelize.
- **Cross-cutting concerns** (e.g., config change, migration) — standalone section.

## Parallel safety rubric

Mark two sections parallel-safe ONLY if ALL are true:
1. They don't modify the same files.
2. They don't modify config/schema/types that the other reads.
3. Their tests don't share state (e.g., both writing to the same DB table in integration tests).
4. Completing one doesn't change acceptance criteria of the other.

If uncertain → mark sequential. The cost of a false positive (merge conflicts, broken build) is much higher than the cost of sequential execution.

## TDD-first discipline

For each section, you must write the tests-first list. This forces clarity on what the section is actually *for*. If you can't describe the tests, the section is underspecified — clarify before moving on.

For sections that genuinely can't be unit-tested (e.g., "add a button to a page"), specify what observable behavior will be checked (e.g., "snapshot test, or manual check of click flow") and WHY no deeper test is feasible.

**Keep the "tests to write first" list lean.** Each entry should be one *behavior*, not one *assertion*. If five entries share identical setup and only differ in which field is checked, collapse them to one entry that covers the shape, or note "parameterize over X, Y, Z". The section agent will expand on it, but the plan sets the tone — sprawly test lists produce sprawly test files. Thorough ≠ fragmented.

## When the spec can't be planned

If you realize mid-plan that the spec has gaps — e.g., two spec sections contradict each other, or a required data shape is missing — STOP. Go back to the user:

> "While planning, I found <specific gap/conflict>. I can't plan around it without making decisions the spec didn't cover. Options:
> a) I make this call myself and document it — [my proposal]
> b) You want to decide this one — [option 1] / [option 2]
> c) Pause and loop back to spec to tighten this"

Don't guess on fundamentals. Small gaps you can fill with a one-line decision in the plan (announced); big gaps need the human.

## Approval gate (third and final gate before execution)

When the plan is written, present:

> ## Plan ready for approval
>
> Written to `.devk/plan.md`. Structure:
>
> - **Sections:** <N total, split by goal>
> - **Parallel groups:** <describe, e.g., "S3a ∥ S3b">
> - **Estimated scope:** <short description>
> - **Material decisions added during planning:** <any new ones not already in spec>
>
> Approve and I'll hand off to `devk-executing-plan` (which dispatches each section to a fresh subagent with TDD and per-section review). Or flag what you want to change.

**Wait for explicit approval.** On approval:

1. **Commit the plan** (if the project is a git repo). Stage only `.devk/plan.md`. Match project commit convention if obvious; default to `devk: plan for <feature title>`.
   ```
   git add .devk/plan.md
   git commit -m "devk: plan for <feature title>"
   ```
   Skip silently if not a git repo.

2. Invoke `devk-executing-plan` via the Skill tool.

## Reminders

- One question at a time IF you need to loop back to the user (only on genuine gaps).
- Sections are vertical slices by default; split shared foundations out when needed.
- Parallel-safe is a strong claim. Default to sequential when uncertain.
- TDD per section, no exceptions without justification.
- Approval gate is the LAST stop before execution. Make it easy to approve or redirect.
