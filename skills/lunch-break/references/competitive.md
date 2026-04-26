---
description: research what similar products do that we don't
---

# Lens: Competitive — what similar products do

## Scope

Use web research (WebFetch + WebSearch) to find what comparable products in this space do, and identify capabilities, patterns, or design decisions worth borrowing — or worth deliberately *not* doing.

This lens cares about:
- Identifying the actual competitive set (not just the obvious ones — also adjacent tools)
- Feature differences — what they have that this project doesn't (and vice versa)
- Pricing / packaging differences
- Onboarding differences (how do they introduce themselves to a new user?)
- Marketing positioning differences (how do they describe themselves?)
- Notable patterns / interaction designs / unique angles
- Recent moves (anything significant they've shipped recently)

This lens does NOT cover:
- In-depth UI redesigns of this project's own UI — `ui` lens
- Internal tech-debt — `tech-debt` lens

## Phase 1 — Scout

Dispatch 3–4 subagents. Note: scout 2 needs scout 1's output, so dispatch scout 1 (and scout 3) first, then scout 2 once scout 1 returns.

1. "Read the project's README, homepage, package.json, marketing copy, and main config to understand: what category is this product in, who is its user, what are its core capabilities. Output a 4-bullet summary of the product and its category."
2. (Web — runs after scout 1) "Given the product description '<plug in scout 1's summary>', search the web for the 5–10 closest competitors by category. For each, list name, URL, and one-line positioning. Use WebSearch."
3. "Look for any references in the repo (READMEs, blog posts, marketing copy, comments) where the team mentions other products by name — competitors, inspirations, alternatives. Quote them with file:line."

## Phase 2 — Deep dive

Take the 5–10 competitors identified by scout 2. Dispatch one subagent per competitor (so liberal ≈ 8–10 subagents, maximum ≈ 15+ if you also do meta-research).

**Per-competitor prompt template:**
> "Investigate <competitor name> at <URL>. Use WebFetch on their homepage, pricing page, features/product page, and changelog/blog if findable. Report:
> - **Positioning:** how *they* describe themselves in one line
> - **Key features:** especially ones that distinguish them from generic offerings in this category
> - **Pricing/packaging shape:** free tier? per-seat? usage-based? enterprise tier? gating?
> - **Notable interaction patterns or design decisions** visible from the marketing site (specific things, not 'modern design')
> - **Recent moves:** anything they've shipped or announced in the last 6 months (check blog/changelog/twitter if findable)
> - **One thing this competitor does that '<our project>' doesn't, that you think is genuinely interesting** (and one thing they do badly that we should avoid)"

For **maximum** mode, also dispatch:
- "Find recent (last 6 months) industry chatter about <category>. What are the trends, complaints, or wishlist items showing up in Hacker News / Reddit / Twitter / Producthunt discussions? Use WebSearch. Quote actual posts/comments where possible."
- "Find 2–3 *adjacent* products — not direct competitors but tools serving overlapping needs from a different angle (e.g. for a note-taking app: read-it-later tools, RSS readers, journaling apps). Profile each briefly. Sometimes the most interesting ideas come from sideways."
- "Look for the leader in the category if there is one. Then find the scrappy challenger. Compare how each positions and what each emphasizes — the divergence is usually instructive."

## Finding shape

Each finding answers: "What does <competitor> do that we don't, and should we care?"

- **Competitor:** name + link
- **Capability:** what they do (specific, not vague)
- **Why interesting:** what user need it serves; why it might matter for this project
- **Suggested response:** one of [borrow it, do it differently, deliberately don't, watch and wait], with one-line reasoning

Also include:
- **What we already do better** — sometimes the report is reassurance, not envy
- **Positioning observations** at the end — based on all this scanning, are there positioning angles in the market this project could occupy more clearly? Is the product underselling itself in some specific way relative to neighbors?

## Caveats to internalize

- Marketing pages are aspirational, not factual. A "feature" listed on a homepage may be half-shipped or paywalled into oblivion. Note when you're inferring from marketing vs. from a verified product surface.
- Don't treat "they have X, we don't" as automatic action. Half the time the right move is "they have X and that's their problem". Be opinionated.
- This lens uses WebFetch / WebSearch. If those aren't available in the current environment, say so in the report and fall back to whatever the in-repo material allows.
