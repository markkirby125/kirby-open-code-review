# kirby-open-code-review

[![Kirby Skills Collection](https://img.shields.io/badge/Kirby_Skills-Collection-blue?style=flat-square&logo=github)](https://github.com/markkirby125/kirby-skills-collection)

SOP for running Alibaba [Open Code Review](https://github.com/alibaba/open-code-review) (`ocr`) from an AI agent session.

OCR is a hybrid reviewer: deterministic file selection, bundling, and rule matching, plus an LLM agent that emits line-level comments. This skill tells the host agent to invoke the installed `ocr` CLI instead of hand-rolling a substitute review.

**Modes:**
- **Review** — git diffs (workspace, branch range, or a single commit)
- **Scan** — whole files, no meaningful diff required
- **Delegate** — OCR selects files and rules; the host agent performs the review (no OCR LLM key required)

Requires Git >= 2.41 and the `ocr` CLI (`npm install -g @alibaba-group/open-code-review`). OCR-managed review/scan also need a configured LLM (`ocr config provider`).

## Installation & Usage

This is a standard AI agent skill (compatible with Antigravity, Claude Code, Cursor, Windsurf, Grok).

> ### 🪄 The Magic Prompt
> Copy and paste this directly to your AI (Cursor, Windsurf, Claude Code):
>
> ```markdown
> @agent Please install the kirby-open-code-review skill into this workspace.
> 1. Read the `SKILL.md` file (and `references/` directory if applicable) from this repository: https://github.com/markkirby125/kirby-open-code-review
> 2. Identify the correct rules system for our current environment (e.g., `.cursor/rules/` for Cursor, `.windsurfrules` for Windsurf, `.clinerules` for Cline, `~/.agents/skills/` for Antigravity, or `~/.grok/skills/` for Grok).
> 3. Save the contents appropriately. If our environment supports multi-file dispatcher skills, clone the directory structure exactly.
> 4. Confirm when the installation is complete.
> ```

### Manual Installation

- **Cursor:** Copy `SKILL.md` to `.cursor/rules/kirby-open-code-review.mdc`
- **Windsurf:** Append the contents of `SKILL.md` to `.windsurfrules`
- **Antigravity / Claude-compatible agents:** Clone this repository to `~/.agents/skills/kirby-open-code-review`
- **Grok:** Clone or symlink this repository to `~/.grok/skills/kirby-open-code-review`

Trigger with `/kirby-open-code-review`, `ocr review`, `ocr scan`, or `ocr delegate`.

## Tech Stack

- **Format**: Markdown / YAML
- **CLI**: `ocr` from [@alibaba-group/open-code-review](https://www.npmjs.com/package/@alibaba-group/open-code-review)
- **Docs**: https://open-codereview.ai/docs
- **Compatibility**: Antigravity, Claude Code, Cursor, Windsurf, Cline, Grok
