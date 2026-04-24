---
description: review user-facing text for quality and personality
---

# Lens: Copy / text quality / personality

## Scope

Look at every piece of user-facing text in the project — from button labels and tooltips to error messages, empty states, marketing pages, and long-form content. Flag where the writing is bland, generic, awkward, off-brand, inconsistent in voice, or just not as good as it could be.

This lens cares about:
- Microcopy: button labels, tooltips, placeholders, helper text, link text
- System messages: errors, confirmations, success states, empty states, loading messages
- Onboarding text and instructional copy
- Marketing/landing copy (hero sections, feature descriptions, CTAs, pricing)
- Long-form content (docs, articles, about pages, blog)
- Voice/tone consistency across the app
- Words that hide jargon, sound corporate, or are silently confusing

This lens does NOT cover:
- Visual design — `ui` lens
- Accessibility of text presentation (font size, line length, etc.) — `a11y` lens
- Whether features should exist — `what-next` lens

## Phase 1 — Scout

Dispatch 3–4 subagents:

1. "Find where user-facing strings live in this codebase. Are they in i18n files, hardcoded in JSX/templates, in markdown, in a CMS? Estimate volume per location and report how copy gets surfaced to users."
2. "Identify the marketing/landing surface (homepage, about, feature pages, pricing, blog). List all of them with brief content notes."
3. "Identify all error messages, empty states, and confirmation messages in the app. Quote a representative sample of each category — actual strings, with file:line."
4. "Look for any existing brand/voice guide, style guide, or tone document in the repo. If none, infer the current voice from the homepage and product UI in 1–2 sentences (e.g. 'professional but warm', 'no-nonsense technical', 'playful and conversational')."

## Phase 2 — Deep dive

Dispatch 8–15 subagents (liberal) or 20+ across waves (maximum):

- "Review every error message in the app. Flag any that are technical, blamey ('You did X wrong'), unhelpful ('Something went wrong'), or robotic. Quote the exact text and propose a rewrite in the project's voice."
- "Review every empty state. Are they helpful (teach the user what to do next) or just 'No items found'? Quote and rewrite the weak ones."
- "Audit button labels across the app. Are they action-oriented and specific? Are 'Submit' / 'OK' / 'Save' / 'Cancel' used where something more specific would be clearer ('Send invite', 'Discard draft')?"
- "Read the homepage hero. Does it pass the 'so what?' test? Would a stranger know what this product does and why they should care in 5 seconds? Propose a sharper version."
- "Review the feature/pricing/about pages for clichés ('seamless', 'powerful', 'best-in-class', 'unlock', 'leverage', 'revolutionize', 'next-generation'). Quote and rewrite to be concrete."
- "Look for personality opportunities — places where copy is currently generic but could carry brand voice: 404 pages, onboarding, empty states, success messages, micro-confirmations."
- "Audit form helper text and placeholders. Are they doing real work or are they noise? Are placeholders being used where labels should be?"
- "Read the onboarding flow end-to-end. Does the copy guide the user, or does it just describe what's already on screen?"
- "Look for inconsistent terminology — does the same concept have different names in different places (e.g. 'workspace' vs 'project' vs 'team', 'user' vs 'member' vs 'account')? List every drift you find."
- "Check confirmation / success messages. Do they confirm specifically what happened ('Invite sent to alex@…') or just 'Done' / 'Saved'?"
- "Find every tooltip and helper-text instance. Are any of them just restating what the button/field already says? Are any genuinely teaching something?"
- (If applicable) "Read the email templates / transactional emails. How does their voice compare to the in-app voice?"

For **maximum** mode, second wave: take the strongest 2–3 surface areas (e.g. error messages, hero copy) and dispatch subagents to do a fuller rewrite of the top 10 in each.

## Finding shape

For each finding:
- **Quote the original text exactly** (with file:line if you have it).
- **Name the issue** in a few words ("blames user", "buried the verb", "voiceless", "competing CTAs").
- **Propose a specific rewrite** — actual text the user could paste in, not a vague "make it warmer".
- Where useful, give 2 alternative rewrites at different tone levels.

Don't propose vague rewrites. The user should be able to ship your suggestion verbatim if they like it.
