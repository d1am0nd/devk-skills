# devk-skills

Claude Code skills for people who want features, not code.

Inspired by [superpowers](https://github.com/obra/superpowers), but for folks who never wanted to look at a diff in their life. You tell the LLM *what* you want. It owns the *how* — architecture, quality, tests, reviews, the boring engineering responsibilities you'd rather not carry.

Three sets of skills, three modes of work:

- **devk-brainstorm** — hands-on build: you build something together.
- **lunch-break** — hands-off explore: you walk away, it explores.
- **code-review** — hands-on review: you point at a changeset, it reviews carefully.

## Install

```sh
cp -r skills/devk-* skills/lunch-break skills/code-review ~/.claude/skills/
```

## devk-brainstorm — build things together

The workflow: **brainstorm → (bounce ideas | implement → spec → plan → execute → review)**. Every step hands off to a subagent with a narrow job, so the main thread doesn't turn into mush. You stay at the product level. The LLM sweats the craft.

Trigger it with `/devk-brainstorm` (or `/devk-brainstorm <your idea>` to skip the opening prompt). That's the reliable entry point — natural-language openings like "let's build..." will usually route there too, but the slash command is the sure thing.

## lunch-break — go explore while I'm away

You pick a lens (UI, copy, SEO, a11y, what-next, competitive, tech-debt, …), walk away, and come back to a written report of observations and ideas. The skill fans out subagents to cover ground in parallel, then writes a single report you can skim and cherry-pick from. No code changes — just ideas.

Trigger it with `/lunch-break <lens>`, or `/lunch-break` on its own to see the available lenses. Natural-language openings like "go explore the copy for me" or "do a UI review while I'm out" also work.

## code-review — review a local changeset together

You point at a local changeset (last N commits, branch vs main, staged, working tree, a specific commit). The skill is ruthless about context: lockfiles, generated code, and large test bodies stay out of the main thread. It surveys the change, forms an intent hypothesis from a smart-selected subset of diffs, then confirms that intent with you — one question at a time, a/b/c options — before dispatching subagents in parallel for the deep review. Section reviewers look one level deeper than the diff (callers, callees, neighbors) where sneaky bugs hide. A standards enforcer checks the change against CLAUDE.md / AGENTS.md / lint configs / neighbor patterns. Output is a tight report in chat — no files, no auto-fix. Findings are yours to act on.

Trigger it with `/code-review <scope>` or natural language like "review the last 4 commits", "review my branch vs main", or "code-review the staged changes".

## Philosophy

- You drive intent. The LLM owns quality.
- Quality over speed. The project lives for years; a hack is worse than waiting.
- One question at a time. Always.
- If a senior engineer would mention it in a PR description, the LLM announces it. Otherwise silent.
