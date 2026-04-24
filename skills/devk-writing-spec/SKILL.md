---
name: devk-writing-spec
description: Pipeline step in the devk workflow — do NOT use as an entry point. The workflow is entered through devk-brainstorm. Use only after devk-implementing has finished clarifying requirements and the user has approved the summary (or when requirements already exist in .devk/requirements.md). This skill writes the actual spec (.devk/spec.md), runs it through parallel subagent review (architecture + code quality + optional docs research), incorporates fixes, and presents the final spec to the user for approval before plan-writing.
---

# devk-writing-spec — Write the spec and run it through review

You have approved requirements (in conversation memory and/or `.devk/requirements.md`). Now you write the technical spec, get it reviewed by independent subagents, incorporate the findings, and present it to the human for the second approval gate.

## Core principles (carry forward)

- **Quality over speed. No hacks. Project lives for years.** Bad spec = bad code downstream.
- **You own the HOW.** Package choices, patterns, architecture — you decide. Human sees them.
- **Announce material decisions with one-line rationale** directly in the spec (e.g., "Using `tanstack/query` — centralizes request caching; already in the project").
- **YAGNI.** If the requirement doesn't need it, don't design for it. Three similar lines is better than a premature abstraction.
- **Don't invent surface area.** Match existing project conventions unless there's a compelling reason to diverge (and if so, explain it in the spec).

## Step 1: Read requirements and survey

- Read `.devk/requirements.md` if it exists. If not, requirements are in conversation context.
- If you don't already have a project survey in context (from `devk-brainstorm`), briefly scan the relevant parts of the project yourself (Grep/Glob/Read the parts of the codebase the spec touches). Don't re-survey the whole project; focus on what this feature interacts with.

## Step 2: Draft the spec

Write the spec to `.devk/spec.md`. Use this structure:

```markdown
# Spec: <feature name>

## Goal
<1-2 sentence restatement from requirements>

## Approach
<3-6 sentences describing the overall shape: what gets built, how the pieces relate>

## Components
- **<Component A>** — <purpose, 1-2 lines>
- **<Component B>** — <purpose>

For each component, describe its responsibility at the right level of detail. Don't
list every function — list what the component owns.

## Data shapes / contracts
<Input and output shapes at boundaries. API contracts. DB schema changes. Event payloads.>

## Dependencies
- **<package>@<version>** — <why; when announced>
- **<package>** — existing, no change

For each NEW or UPGRADED dependency, give a one-line rationale. No new deps without justification.

## Error handling strategy
<How failures propagate. What user-visible errors look like. Retries, timeouts, fallbacks.>

## Testing strategy
<Test levels to cover (unit, integration, e2e). Key scenarios. What's NOT tested and why.>

## Out of scope
- <explicit deferral from requirements>

## Risks and open questions
<Things that might bite us. Unknowns. Edges we're deliberately choosing vs deferring.>

## Decisions and rationale
<Material decisions the LLM made, with 1-2 line "why". This is the announce list.>
```

Keep the spec *concrete enough* that a competent engineer could implement it, but *short enough* that reading it doesn't take longer than writing the code. A typical spec is 1-3 pages. If it's longer, ask whether you're overspecifying.

Do NOT:
- List every file path or function signature — that's for the plan.
- Design for hypothetical future needs.
- Add error handling for scenarios that can't happen (trust internal guarantees; only validate at boundaries).
- Add unnecessary abstractions.

## Step 3: Run parallel review

Spawn subagents in parallel (single message, multiple Agent tool calls):

**Reviewer 1 — Architecture** (always runs):
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: "Architecture review of spec"
- `prompt`: "Load and follow the `devk-reviewing-spec-architecture` skill via the Skill tool. The spec to review is at `.devk/spec.md`. Project is at the current working directory. Return findings in the structured format the skill specifies."

**Reviewer 2 — Quality** (always runs):
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: "Quality review of spec"
- `prompt`: "Load and follow the `devk-reviewing-spec-quality` skill via the Skill tool. The spec to review is at `.devk/spec.md`. Return findings in the structured format the skill specifies."

**Reviewer 3 — Docs research** (conditional):
Run ONLY if the spec introduces a new package/dep or depends on a specific external API whose current behavior matters. Skip if the spec only uses things already in the project.
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: "Verify docs for new deps"
- `prompt`: "Load and follow the `devk-researching-docs` skill via the Skill tool. Spec at `.devk/spec.md`. Focus on these new/upgraded dependencies: [list them with proposed versions]. Return the structured report the skill specifies."

## Step 4: Read reviews, classify, act

Each reviewer returns structured findings: **blocker / concern / nit**.

**Blockers** — must be resolved before proceeding. Three possibilities:
- **Fixable in the spec** → edit the spec and note the fix. No need to re-review unless the fix is large.
- **Architectural conflict with project** (e.g., "this violates how auth is done everywhere else") → STOP. Do not fix silently. Present to the human: "The reviewer flagged this blocker. I think the right path is X, but it means we need to also do Y first. How do you want to proceed?" — this is a judgment call that deserves the human's eye.
- **Requirements mismatch** (reviewer believes the spec doesn't meet the stated requirements) → STOP. Check yourself: is the reviewer right? If yes, fix the spec. If no (reviewer misread), briefly note why and move on.

**Concerns** — address in the spec if the fix is small. If large, add to "Risks and open questions" and move on.

**Nits** — ignore unless they're about clarity and easy to fix.

## Step 5: Second pass (if needed)

If the spec changed meaningfully after review, consider one more quick review pass — but don't loop indefinitely. Use judgment: reviewers will always find more things to complain about. Stop when the blockers are gone and the spec honestly reflects the approach.

## Step 6: Present to human (approval gate)

Present the spec with a short summary. Keep the voice PM-friendly — the human is deciding whether this captures what they want, not code-reviewing the spec line by line. The spec file itself stays technical; your *presentation* is product-level.

Format:

> ## Design ready for your sign-off
>
> I've written up how I'd build this — saved at `.devk/spec.md` if you want the full version.
>
> **The gist:** <1-2 plain-language sentences: what gets built and how it fits together.>
>
> **Notable calls I made:** <material decisions translated to product terms where possible, e.g., "Using a background queue for email sends so signup stays fast" rather than "Using BullMQ on Redis".>
>
> **From the review pass:** <1-3 short bullets of what reviewers flagged and how I handled it — in plain language, not reviewer jargon.>
>
> **Open risks:** <anything unresolved worth knowing about.>
>
> Sound right? Approve and I'll move to planning. Or flag what's off.

**Wait for explicit approval.** On approval:

1. **Commit the spec** (if the project is a git repo). Stage only the devk artifacts — do NOT stage unrelated changes. Use a commit message that matches the project's convention if it's obvious (check `git log --oneline -10`); otherwise default to `devk: spec for <feature title>`. Example:
   ```
   git add .devk/spec.md .devk/requirements.md .gitignore
   git commit -m "devk: spec for <feature title>"
   ```
   If it's not a git repo, skip silently.

2. Invoke `devk-writing-plan` via the Skill tool.

## When to escalate rather than fix

The reviewers might reveal that the requirements themselves are flawed (e.g., "this would require N weeks of refactoring that wasn't in scope"). In that case, go back to the human with options — don't silently redefine the work. Your job is transparency, not heroism.

## If the user wants to stop here

If the human signals they want to pause or drop the feature after seeing the spec ("let's shelve this", "not worth it", "actually let's rethink"), acknowledge and offer to tidy up `.devk/`.

> Got it — pausing this.
>
> Working notes in `.devk/`:
> - `requirements.md` — what we agreed to build
> - `spec.md` — the design
>
> What should I do with them?
>
> **a) Leave as-is** — I'll pick up next time. *Default if you might come back to this.*
> **b) Archive** — move to `.devk/archive/<YYYY-MM-DD>-<slug>/`.
> **c) Delete** — clean slate. I'll confirm first.

Act on the answer:
- **a)** Do nothing.
- **b)** `mkdir -p .devk/archive/<date>-<slug>/`, `git mv` (or `mv`) existing `.devk/*.md` into it. Commit if git repo: `devk: archive in-flight work (<slug>)`.
- **c)** Show the list, confirm, then `rm`. Commit: `devk: discard in-flight work (<slug>)`.

## Reminders

- Quality over speed. No hacks in the spec. A spec that says "we'll just monkey-patch X" is a bad spec.
- Announce material decisions with rationale.
- The review is parallel (single message, multiple Agent calls).
- Don't rubber-stamp reviewer findings; read them critically. They can be wrong.
- Approval gate is explicit before moving to plan.
