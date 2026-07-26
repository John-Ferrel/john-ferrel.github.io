---
title: AGENTS
draft: true
tags:
created: 2026-06-29 00:00
modified: 2026-07-26 18:48
---
# AGENTS.md

This is a Quartz personal technical blog. Content lives in this directory.

## Structure

- `coding/`: programming notes, Python, WSL, Git, CI/CD, neural networks.
- `projects/`: project notes and longer technical posts.
- `Tools/`: tool setup and operations notes.
- `specials/`: personal materials. Edit conservatively.
- `gaming/`: game notes.
- `templates/`: authoring templates.
- `0-assets/images/`: image assets.

## Frontmatter

Use YAML frontmatter:

```yaml
---
title: Page Title
tags:
  - tag-name
draft: false
created: YYYY-MM-DD HH:mm
modified: YYYY-MM-DD HH:mm
---
```

- Use `created` and `modified`; do not add `date`.
- Write tags as lowercase kebab-case, e.g. `machine-learning`, `ai-agent`, `saas-copilot`.
- Prefer frontmatter tags. Do not introduce inline hashtags unless requested.

## Links And Assets

- Use wiki links for internal notes: `[[Page Name]]` or `[[Page Name#Heading]]`.
- Use Obsidian embeds for images: `![[image.png]]`.
- Keep images under `0-assets/images/` unless there is a good local reason.
- When checking links, ignore wiki-link-like text inside code blocks or configuration examples.
- Clean external links by removing tracking parameters when they are not needed.

## Editing

- Inspect relevant files before editing.
- Preserve the author's mixed Chinese/English style and technical wording.
- Keep changes scoped to the request. Avoid broad rewrites or template forcing.
- Do not invent project facts, personal experience, metrics, results, or conclusions.
- Do not rename, move, delete, or reorganize files unless explicitly asked.
- Do not modify code blocks, commands, config snippets, or identifiers during prose polishing unless requested.

## Writing Style

- 禁止使用公式化的 AI 表达，包括先否定后肯定的“不是……而是……”及类似句式。直接陈述结论、证据和影响，减少套话对语料风格的干扰。

## Reporting

After changes, summarize files changed, checks performed, and any assumptions or uncertain points.
