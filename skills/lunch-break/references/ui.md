---
description: audit UI/UX rough edges across the app
---

# Lens: UI / UX rough edges

## Scope

Look at the visual and interactive design of the application. Find rough edges, inconsistencies, and missed opportunities for polish. This lens cares about:

- Visual hierarchy and layout
- Spacing, alignment, sizing inconsistencies
- Color, contrast, and visual balance (visual quality, not strict a11y compliance — that's the `a11y` lens)
- Component states (hover, focus, disabled, loading, empty, error)
- Form design (labels, inline errors, affordances)
- Interaction feedback (does the user know something happened?)
- Responsive behavior (does it hold up on different sizes?)
- Visual consistency across the app (drift between similar components)

This lens does NOT cover:
- Accessibility compliance — `a11y` lens
- Copy / microcopy — `copy` lens
- Performance / load times
- Adding new features — `what-next` lens

## Phase 1 — Scout

Dispatch 3–4 subagents in parallel. Each scout's job: map a slice of the UI surface and report back what they found. Don't review quality in this phase — just inventory.

Scout prompts (adapt routes/files to the actual project):

1. "Map all top-level routes/pages in the app. For each, list the main UI components and a one-sentence purpose. Don't review quality — just inventory."
2. "Find all reusable UI components (Button, Card, Modal, Input, etc.). Note where they live and roughly how many places use each."
3. "Identify all form-driven flows in the app (login, signup, settings, checkout, contact, etc.). List each form, its fields, and where it lives."
4. (If applicable) "Look at the design system / theme files (CSS variables, Tailwind config, theme tokens, design tokens). Summarize the color palette, spacing scale, typography choices."

## Phase 2 — Deep dive

Based on what scouts found, dispatch 8–15 subagents (liberal) or 20+ across waves (maximum). Each gets a focused, hypothesis-driven prompt. Mix and match these patterns based on actual scout output:

- "Review <route X> for visual hierarchy issues. Are the most important actions visually dominant? Is anything fighting for attention?"
- "Check <component Y> across all its usages. Is it visually consistent, or are props being misused to make it look different in different places?"
- "Audit the empty states for <list/grid components>. Are they helpful (teach the user what to do) or just blank?"
- "Audit loading states across the app. Are they consistent? Do they tell the user what's loading? Are there layout shifts?"
- "Review all forms for: clear labels, visible focus states, inline error placement, sensible field order, sensible default focus."
- "Check responsive behavior of the 3 most complex pages at narrow widths (~375px). What breaks?"
- "Look for inconsistent spacing between sections of <page Z>. Is the rhythm off?"
- "Look at hover/focus/active states on interactive elements. Are they consistent? Are any missing entirely?"
- "Audit modals/dialogs/popovers. Do they all have consistent close affordances? Do they handle long content gracefully?"
- "Examine the navigation. Is the active state always clear? Does it survive deep nav?"
- "Look at all icon usage. Are icons consistent in style/weight/size? Are any used inconsistently for different meanings?"
- "Check error UI across the app — toast errors, inline form errors, full-page errors. Are they coherent?"

For **maximum** mode, run a second wave: take the most interesting findings from wave 1 and dispatch subagents to dig deeper into those specific areas (e.g. "Wave 1 found checkout flow has weak hierarchy — explore it in depth and propose a re-structure with rough mockup-in-words").

## Finding shape

Each UI finding should reference specific screens, components, and files. Quote the visual or interactive issue precisely (not vague — what is wrong, where). Suggested changes should be design-level (not full CSS), e.g. "increase visual weight of primary CTA on /checkout step 2 — currently it competes with the secondary 'continue shopping' link". Where useful, describe the proposed end-state visually in 1–2 sentences.
