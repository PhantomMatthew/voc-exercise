# AGENTS.md

## What this repo is

A standalone skill package for vocabulary exercise training (VOC Exercise). Not a buildable application — no build, test, or lint commands. The only runtime artifact is `SKILL.md`.

## Structure

- `SKILL.md` — Skill definition and instructions (the only file the agent loads at runtime)
- `_meta.json` — Package manifest (owner, slug, version)
- `vocabulary/units/Unit_<N>.md` — Vocabulary word lists, N = 1..5, Markdown table format
- `question sample.docx` — Reference document showing the two question formats
- `REF list .docx` — Source vocabulary reference lists

## Key constraints

- **Word lists are part of this repo** under `vocabulary/units/`. The skill reads them directly.
- **Runtime state lives outside the repo** at `~/.openclaw/state/voc-exercise/state.json`. Never commit state.
- The skill uses only **built-in Read / Write / Edit** tools — no MCP dependency.
- **Two question types** must be generated every round:
  - **Type 1**: Fill-in-the-blank sentences using target words
  - **Type 2**: Match sentence starters with correct endings
- **Daily word count**: 10 words per round. Day 1 = 10 new words; Day 2+ = 3 from wrong bank + 7 new words.
- A word is considered **mastered** only when it is answered correctly in BOTH question types.
- Distractors for Type 1 must use **target words from the current round** (shown in italics before the blank).
- Distractors for Type 2 must use **target words from the current round** (shown in italics as options).

## Editing SKILL.md

YAML-frontmatter + Markdown. The frontmatter fields (`name`, `description`, `trigger`, `keywords`) are consumed by the skill loader. Keep the instruction body concise — it is injected into the agent's context at runtime.

When changing the state schema, bump `_meta.json` `version` and document the migration in SKILL.md.
