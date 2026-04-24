---
description: discoverability and search audit
---

# Lens: SEO / discoverability

## Scope

Audit the project for search engine and discoverability issues. Cover both technical SEO and content-level SEO opportunities.

This lens cares about:
- Meta tags (title, description, og, twitter)
- Structured data / JSON-LD
- Sitemap, robots.txt, canonical URLs
- URL structure and routing patterns
- Page titles and headings hierarchy (one h1, sensible nesting)
- Image alt text (from a discoverability angle, not strict a11y)
- Internal linking and crawlability
- Content gaps — pages or topics that don't exist but probably should
- Page speed signals visible in the code (blocking scripts, oversized images, render-blocking CSS)
- Open Graph / social preview cards (titles, descriptions, images)

This lens does NOT cover:
- Off-site SEO (backlinks, etc.)
- Paid search
- General copy quality on existing pages — `copy` lens

## Phase 1 — Scout

Dispatch 3–4 subagents:

1. "Identify the site framework (Next.js, Astro, Remix, SvelteKit, plain HTML, etc.). Report how meta tags, head management, and routing are handled in this codebase. Find the relevant config or layout files."
2. "List all public-facing routes / pages. For each, note whether it has unique meta tags or relies on a default."
3. "Check for robots.txt, sitemap.xml, canonical URL handling, and any structured data already present. Report what exists and what doesn't."
4. "Inventory the content surface (blog, docs, marketing pages, glossary, etc.). Roughly how much content is there, and is it indexable (server-rendered vs. client-rendered)?"

## Phase 2 — Deep dive

Dispatch 8–15 subagents (liberal) or 20+ across waves (maximum):

- "For each public page, check meta title and description. Flag missing, duplicated, or generic ones. Suggest unique titles and descriptions per page (write the actual strings)."
- "Check Open Graph and Twitter card tags on every public page. Are images set? Are titles set? Test what shared links would look like."
- "Look at the heading hierarchy on each marketing/content page. Is there exactly one h1? Are headings nested correctly? Quote any that aren't."
- "Audit URL structure. Are URLs human-readable, stable, topic-relevant? Flag any with query-string-only differentiation, opaque IDs, dates that lock content into a moment, or messy patterns."
- "Check for structured data opportunities — Article, Product, FAQ, BreadcrumbList, Organization, SoftwareApplication, etc. Note where JSON-LD would help and propose the schema."
- "Look at internal linking. Are key pages reachable in 1–2 clicks from the homepage? Are there orphan pages with no inbound links?"
- "Check image handling for SEO — are images sized appropriately, do they have descriptive alt text (not file names), are they served in modern formats (WebP/AVIF)?"
- "Look for missing 'about' / 'team' / 'contact' / 'pricing' / 'docs' / 'changelog' style pages that competitors typically have."
- "Identify content gaps — based on the product's domain, what topics or keywords *should* have a landing page or article but don't? Propose 5–10 specific page ideas."
- "Check for any rendering pitfalls — client-side-only content, JS-rendered text that crawlers might miss, lazy-loaded content above the fold."
- "Look at page-speed signals visible from code: render-blocking scripts in head, hero images not preloaded, web fonts loaded synchronously, no image dimensions specified (CLS risk)."
- "Check robots.txt and sitemap.xml content (if they exist). Anything wrong? Anything important being blocked?"

For **maximum** mode, second wave: pick the 1–2 highest-leverage findings (often: missing structured data on a content surface, or a content-gap pattern) and dispatch subagents to flesh out a concrete plan (e.g. "10-page series on <topic>, with titles, h1s, and 100-word outlines for each").

## Finding shape

Be concrete:
- **Cite the specific URL/route/file** the issue lives at.
- **Quote the missing or wrong tag** if applicable.
- **Propose the fix** with example markup, JSON-LD, or copy where useful.

For content gaps, name the missing page, propose its URL slug, and pitch its content in one line.
