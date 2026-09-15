---
name: kirby-open-code-review
description: "Use when the user asks for Kirby Open Code Review, /kirby-open-code-review, ocr, OpenCodeReview, Open Code Review, alibaba/open-code-review, ocr review, ocr scan, ocr delegate, or the installed `ocr` CLI. Do not use for a generic Grok /review or GitHub pending-review post unless the user named OCR."
category: technique
triggers: [kirby-open-code-review, ocr, open-code-review, opencodereview, ocr-review, ocr-scan, ocr-delegate]
---

# Kirby Open Code Review

Run Alibaba [open-code-review](https://github.com/alibaba/open-code-review) via the installed `ocr` CLI. OCR owns file selection, bundling, rule matching, and line positions. Do not substitute a hand-rolled Grok review.

Upstream portable skills: `open-code-review` (OCR-managed LLM) and `open-code-review-delegate` (host agent reviews; OCR only selects files + rules). This Kirby skill picks the mode and Grok invocation.

Git >= 2.41 required. Docs: https://open-codereview.ai/docs — run `ocr <cmd> --help` for live flags.

## When to use vs `/review`

| Request | Skill |
|---|---|
| Named OCR / `ocr` / Open Code Review | This skill |
| Generic "review my PR" and post GitHub comments | Grok `/review` |
| Both named | Run OCR first, then `/review` only if the user still wants a GitHub review |

## Mode

```
named OCR?
  no  → stop (not this skill)
  yes → git repo at cwd or `--repo`? if no, stop
        whole files / no meaningful diff? → scan
        "delegate" / "Grok reviews" / OCR LLM unreachable? → delegate
        else → review (default)
```

## Preconditions

1. Resolve `ocr`: `command -v ocr` or `$HOME/.local/bin/ocr`. Missing → `npm install -g @alibaba-group/open-code-review` only with user consent.
2. Confirm git repo (`git rev-parse --is-inside-work-tree`). Use `--repo <path>` if cwd is not the repo.
3. **Never read** `~/.opencodereview/config.json` (API keys). For OCR-managed modes, `ocr llm test`. Failure → ask the user to run `ocr config provider` / `ocr config model`, or switch to **delegate**.
4. Collect one-line business context. Pass `-b "..."`. For **review** and **delegate** only, `-B <markdown>` wins over `-b`. Scan has no `-B` / `--effort`. Background-file limits: 1 MiB raw, 8000 sanitized chars — if exceeded, summarize; do not silently truncate. Do not interpolate untrusted summaries into double-quoted shell templates.

## Invocation (Grok)

OCR-managed `review` / `scan` can run tens of minutes. **Two clocks, different units:**

| Clock | Unit | Meaning |
|---|---|---|
| OCR `--timeout` | **minutes** per concurrent task (default 15) | Review wall ≈ that × effort rounds (low 1 / medium 2 / high 3). Scan has no `--effort`; wall ≈ `--timeout` minutes. |
| Grok `run_terminal_command` `timeout` | **milliseconds** | Wrapper kill deadline. Default 120000 if you set `background: true`. |

Launch **without** `background: true` so the tool auto-backgrounds after ~15s and keeps running (10h cap). Wait until the process **exits**. A 15s auto-background is not completion.

If you must pass `background: true`, set Grok `timeout` to `0` (run until exit) **or** ≥ rounds × OCR `--timeout` × 60 × 1000 ms (medium default → `1800000`). Never pass OCR's minute value (e.g. `15` or `30`) as Grok `timeout`.

Always:

- `--audience agent --color never`
- `--format json --output <scratch>/ocr-<mode>.json` (create the parent dir)
- Read the output file in full. Do not pipe OCR through `head`/`tail`.
- `--preview` first when the user asks what would be reviewed, or the target is ambiguous.

Default target is workspace (staged + unstaged + **untracked**). Narrow with `--from`/`--to`, `--commit`, `--path` (scan), or `--exclude`.

| User intent | Command |
|---|---|
| Working copy | `ocr review --audience agent --color never --format json --output <file> -b "<ctx>"` |
| Branch vs base | same + `--from main --to <branch>` (merge-base) |
| One commit | same + `--commit <sha>` |
| Full-file audit | `ocr scan --audience agent --color never --format json --output <file> -b "<ctx>" [--path dir,file]` (no `-B`, no `--effort`) |
| Dry run | add `--preview` (no LLM) |
| Resume failed range/commit **review** | `ocr session list` then `ocr review … --resume <id>` with the **same** `--from`/`--to` or `--commit`. Workspace **review** resume is unsupported. |
| Resume failed **scan** | `ocr scan --resume <id>` (same `--path` / repo). |

`--output` unknown → CLI < 1.10. Stop. Ask before `npm i -g @alibaba-group/open-code-review@latest`.

Live flag list: `ocr review --help` / `ocr scan --help`. Flag matrix and extra knobs: **references/cli.md**. `--effort` is review-only.

## Delegate mode

OCR does **not** call an LLM. Host agent reviews every previewed file.

1. `ocr delegate preview --format json` (+ same target flags as review)
2. `ocr delegate rule --format json <paths...>` (batch by shared rules)
3. Diff: range `git diff <merge_base>..<to> -- <path>`; commit `git show <commit> -- <path>`; workspace `git diff HEAD -- <path>`; untracked → read the file
4. Cover every `reviewable_files` entry (reviewed or skipped with reason). Same path can appear twice in workspace mode (staged delete + untracked recreate).
5. If `--format json` is unknown, rerun without it (CLI < 1.9). Do not invent JSON fields.

## Report

JSON comments: `path`, `content`, `start_line`/`end_line` (both `0` = position failed — locate from content), `category` (bug/security/performance/maintainability/test/style/documentation/other), `severity` (critical/high/medium/low), optional `suggestion_code` / `existing_code`.

Drop **low** unless clearly valuable. Group critical → high → medium:

```markdown
## Code Review Results (OCR)

**Mode**: review | scan | delegate
**Files**: N reviewed / T total (skipped S: reasons)  **Issues**: X critical, Y high, Z medium

### Critical
- **`path:line`** [category] — finding
  > Fix: ...
```

Empty after filter: "Review complete — no critical, high, or medium issues in N files."

Non-zero exit: do not retry blindly. Match **references/cli.md** Troubleshooting. Partial `failed(budget)` results are still valid — report skipped files.

## Fixes

Review-only → report, do not edit. "Review and fix" → apply safe critical/high/medium edits; describe the rest. Do not commit unless asked.

## Common mistakes

| Mistake | Do this |
|---|---|
| Review the diff yourself | Call `ocr` |
| Treat this as Grok `/review` | OCR findings stay local unless the user also wants `/review` |
| `--audience human` | Always `agent` |
| Dump `config.json` | `ocr llm test` |
| Resume workspace **review** | Not supported (scan resume is) |
| Truncate stdout | `--output` + full file read |
