# devk-skills

Claude Code skills for people who want features, not code.

Think [superpowers](https://github.com/obra/superpowers), but if you never wanted to look at a diff in your life. You tell the LLM *what* you want. It owns the *how* — architecture, quality, tests, reviews, the boring engineering responsibilities you'd rather not carry.

The workflow: **brainstorm → (bounce ideas | implement → spec → plan → execute → review)**. Every step hands off to a subagent with a narrow job, so the main thread doesn't turn into mush. You stay at the product level. The LLM sweats the craft.

## Install

```sh
cp -r skills/devk-* ~/.claude/skills/
```

Then start any conversation with "let's build..." or "how should I..." and the `devk-brainstorm` skill takes it from there.

## Philosophy

- You drive intent. The LLM owns quality.
- Quality over speed. The project lives for years; a hack is worse than waiting.
- One question at a time. Always.
- If a senior engineer would mention it in a PR description, the LLM announces it. Otherwise silent.

Built on top of [superpowers](https://github.com/obra/superpowers) — same spirit, different vibe.
