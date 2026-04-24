---
description: brainstorm features (LLM-powered ok, no manual human work)
---

# Lens: What next — feature brainstorm

## Scope

Think like a product manager / CEO who's looked at the project and is asking: "What should we build next?" Generate concrete, opinionated feature ideas grounded in what the project already is and what users would plausibly want next.

This lens cares about:
- Features that build on what's already there (natural extensions)
- Features that close obvious gaps (things users probably ask for / hit walls on)
- LLM-powered features the project could add (this is fair game and often high-value)
- Features that compound (things that make existing features better, not just add new surfaces)
- Lightweight quality-of-life improvements that punch above their weight

**Hard constraint: no features that require manual human work.** Examples of disqualified ideas: "human-curated content collections", "manual moderation queue", "weekly editorial newsletter the user has to write". LLMs handling things autonomously *is* allowed and encouraged.

This lens does NOT cover:
- UI polish on existing features — `ui` lens
- Removing features (that's a tech debt audit)
- Changes to non-product copy — `copy` lens

## Phase 1 — Scout

Dispatch 3–5 subagents:

1. "Read the README, homepage, and main entry point of this project. Summarize what this product *is* — its core value prop and main use cases — in 4 bullets."
2. "Map the current feature surface area. List every distinct feature/capability the product currently offers."
3. "Read any roadmap, todo, ideas, issues, or planning file in the repo. Quote what the team is already considering."
4. "Identify the user data and content this product touches (what users create, store, see, interact with). This is the raw material for new features."
5. "Briefly identify 2–3 closest competitors / similar tools by category. Don't deep-dive, just name them and one-line each. (If the user is also running the `competitive` lens later, this is just an anchor.)"

## Phase 2 — Deep dive

Dispatch 8–15 subagents (liberal) or 20+ across waves (maximum). Mix angles:

**Natural extensions:**
- "Given the core feature is <X> (per scout summary), what 3–5 features would naturally extend it without changing the product's identity?"
- "What is currently a multi-step workflow in this product that could become a one-click feature?"

**LLM-powered features:**
- "Given the user data this product handles (<list from scout 4>), what LLM-powered features would create real value? Think: summarization, classification, transformation, generation, semantic search, recommendation, conversation, extraction, comparison. Be concrete about input/output for each idea."
- "Where in the current UX is the user typing or pasting structured-ish text/data that an LLM could parse, complete, or improve in-line?"
- "What 'AI assistant' surface would actually fit this product (vs. being a tacked-on chatbot)? Or is the answer 'no chatbot, do these embedded LLM features instead'?"
- "What background LLM work could happen automatically on user data (auto-tagging, auto-summarizing, auto-grouping, anomaly detection, draft suggestions) without the user having to ask?"

**Compounding features:**
- "What feature, if added, would make 3+ existing features more useful?"
- "What would make this product noticeably more useful for power users *without* harming the new-user experience?"
- "What integration (with what other tool) would unlock disproportionate value?"

**Closing obvious gaps:**
- "What's a feature that competitors in this space typically have but this product doesn't? Filter to ones that actually fit this product's identity."
- "What automation would remove a recurring annoyance for users (based on the workflows visible in the code)?"
- "What's a 'why doesn't this exist?' feature — something so natural that users probably wonder why it isn't there?"

**Wildcards:**
- "Propose 3 weird, ambitious features that might be wrong but would be very interesting if right. Take real risks here. Don't sandbag."

For **maximum** mode, run a second wave: take the strongest 3–4 ideas from wave 1 and dispatch subagents to flesh each into a mini-spec (one-paragraph pitch + suggested entry point in the existing UX + biggest risk + a fast-cheap experiment to validate it).

## Finding shape

Each idea should have:
- **What:** one-sentence pitch
- **Why now:** why this fits *this* product *this* moment, not a generic "AI is hot"
- **First version:** the smallest version that proves it works (concrete and shippable)
- **Honest concern:** the biggest reason it might be wrong

Order findings roughly by perceived value-to-effort, highest first. Group them by category (extensions / LLM-powered / compounding / gaps / wildcards) and clearly mark "weird wildcards" so the user knows which ones are spicy.

End with a **Themes I noticed** section — patterns across the ideas (e.g. "Many of the strongest ideas point at the same gap: this product doesn't yet help users *find* what they already created").
