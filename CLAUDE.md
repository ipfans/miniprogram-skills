# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A plugin marketplace providing WeChat Mini Program development skills for Claude Code, Codex, and other AI coding agents. The single plugin currently shipped is `wechatide-skill` (WeChat DevTools workflow) under `plugins/wechatide-skill/`.

## Dual marketplace format

Every plugin must be registered in **two** marketplace manifests with different schemas:

- **Claude Code** (`.claude-plugin/marketplace.json`): `"source": "./plugins/<name>"` — plain string path.
- **Codex** (`.agents/plugins/marketplace.json`): `"source": { "source": "local", "path": "./plugins/<name>" }` — object with `policy` and `category` fields.

Each plugin also has **two** internal manifests: `.claude-plugin/plugin.json` (Claude Code) and `.codex-plugin/plugin.json` (Codex, uses `interface` object for display metadata).

Always update both marketplace files and both plugin manifests when adding or modifying a plugin.

## Version management

The **SKILL.md frontmatter** (`version:` in `plugins/<name>/skills/wechatide-tools/SKILL.md`) is the source of truth. When bumping a version:

1. Update the `version` in SKILL.md frontmatter
2. Sync to `plugins/<name>/skill.yaml`
3. Sync to `plugins/<name>/.claude-plugin/plugin.json`
4. Sync to `plugins/<name>/.codex-plugin/plugin.json`
5. Update the version in `README.md` plugin table

## Plugin structure

```
plugins/<name>/
├── .claude-plugin/plugin.json
├── .codex-plugin/plugin.json
├── skills/
│   ├── <skill>/SKILL.md
│   └── wechatide-tools/        # root skill entry + shared references
│       ├── SKILL.md
│       └── references/
├── SKILL.md                    # root skill entry (optional)
└── skill.yaml                  # version metadata
```

Relative links in skill SKILL.md files use `../wechatide-tools/references/<file>` to reach shared reference docs. Files inside `wechatide-tools/references/` use bare filenames for sibling references.

## Language conventions

- Skill content (SKILL.md, references) is written in **Chinese (Simplified)**.
- Repo-level files (README.md, CLAUDE.md, marketplace manifests) are in **English**, except where the content is inherently Chinese (plugin descriptions aimed at Chinese-speaking users).

## Scripts

The installer scripts under `plugins/wechatide-skill/skills/installer/scripts/` are Node.js ESM (`.mjs`) using only built-in modules — no `npm install` required. Run with `node <script>`.

## No build system

This repo has no package manager, build step, test runner, or linter. All content is Markdown, JSON, YAML, and standalone `.mjs` scripts.
