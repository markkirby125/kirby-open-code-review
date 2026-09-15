---
name: kirby-open-code-review
description: "Use when the user asks for Kirby Open Code Review, /kirby-open-code-review, ocr, OpenCodeReview, Open Code Review, alibaba/open-code-review, ocr review, ocr scan, ocr delegate, or the installed `ocr` CLI. Do not use for a generic Grok /review or GitHub pending-review post unless the user named OCR."
category: technique
triggers: [kirby-open-code-review, ocr, open-code-review, opencodereview, ocr-review, ocr-scan, ocr-delegate]
---

# Kirby Open Code Review

Run Alibaba [open-code-review](https://github.com/alibaba/open-code-review) via the installed `ocr` CLI. OCR owns file selection, bundling, rule matching, and line positions. Do not substitute a hand-rolled Grok review.

Upstream portable skills: `open-code-review` (OCR-managed LLM) and `open-code-review-delegate` (host agent reviews; OCR only selects files + rules). This Kirby skill picks the mode and Grok invocation.

Git >= 2.41 required. Run `ocr <cmd> --help` for live flags.

- Docs: https://open-codereview.ai/docs
- GitHub: https://github.com/alibaba/open-code-review
- Quickstart: https://open-codereview.ai/docs/quickstart

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
  yes → Preflight A (binary, git, repo) — stop on first hard fail
        whole files / no meaningful diff? → scan
        "delegate" / "Grok reviews"? → delegate
        else → review (default)
        Preflight B (LLM) — review/scan only; skip for delegate
        Preflight C (preview + output dir)
```

## Preflight

Stop on the first hard fail. Do **not** auto-install. Do **not** run `ocr config provider` or `ocr config model` (interactive TUI — user must run them in a real terminal). Never print `api_key` / `auth_token` / the rest of `config.json`. Inspect snippet: **references/cli.md** (Preflight inspect).

Print **Preflight OK** on success. Delegate must **not** run inspect or `ocr llm test` just to fill the block.

Review/scan:
```
Preflight OK
  ocr: /path/to/ocr  (v1.12.2)
  git: 2.55.0  repo: /path/to/repo
  provider: deepseek  model: deepseek-flash  key: SET
  llm test: OK
  mode: review
```

Delegate:
```
Preflight OK
  ocr: /path/to/ocr  (v1.12.2)
  git: 2.55.0  repo: /path/to/repo
  provider/model/key/llm test: skipped
  mode: delegate
```

### A — binary, git, repo (all modes)

1. Resolve `OCR_BIN`: `command -v ocr` or executable `$HOME/.local/bin/ocr`. If only the home path works, use that absolute path for every later `ocr` call and mention PATH.

   **Fail** — print and stop:
   ```
   OCR is not installed on this machine.

   Install:
     npm install -g @alibaba-group/open-code-review

   Then confirm:
     ocr version
   ```

2. `ocr version` must succeed. Compare **numeric** major.minor (not string compare — `1.9` is older than `1.10`). Need **≥ 1.10** (`--output`; also covers delegate `--format json` ≥ 1.9). Older → stop; ask before `npm i -g @alibaba-group/open-code-review@latest`. Missing-binary fail (step 1) is not an install prompt for the agent.

3. `git --version` ≥ **2.41** (numeric major.minor). Missing or older — stop:
   ```
   OCR requires Git >= 2.41 (found: <version or missing>).
   ```

4. Resolve `REPO_ROOT`: `--repo <path>` if set, else cwd. Check `git -C "$REPO_ROOT" rev-parse --is-inside-work-tree`. Pass that same `--repo` to every later `ocr` call. Do not treat “cwd is a repo” as proof that `--repo` is.

### B — provider, model, key, llm test (review / scan only; skip for delegate)

5. Run the inspect snippet. Config missing or unreadable → treat provider and model as empty.

   **(a) provider empty** — stop:
   ```
   OCR provider is not configured.

   In your own terminal (interactive):
     ocr config provider
   ```

   **(b) provider set, model empty** (top-level `model`, else `providers.<provider>.model`, else `llm.model`) — stop:
   ```
   OCR provider is "<name>", but no model is selected.

   In your own terminal (interactive):
     ocr config model
   ```

6. Active provider key present? Boolean only (`SET` / `MISSING`). `MISSING` — stop:
   ```
   OCR provider is configured but no API key is set.

   In your own terminal:
     ocr config provider
   or:
     ocr config set providers.<name>.api_key "$KEY"
   ```
   Never invent a key.

7. `ocr llm test`. Fail → stop (key/network/quota). Offer delegate as an alternative. Do not retry blindly.

### C — target and output (all modes)

8. Preview (no LLM): `ocr review --preview` / `ocr scan --preview` / `ocr delegate preview --format json` with the same target flags. Zero reviewable files → stop: “nothing to review”.

9. `mkdir` the `--output` parent (review/scan). Not writable → stop.

Then collect one-line business context. Pass `-b "..."`. For **review** and **delegate** only, `-B <markdown>` wins over `-b`. Scan has no `-B` / `--effort`. Background-file limits: 1 MiB raw, 8000 sanitized chars — if exceeded, summarize; do not silently truncate. Do not interpolate untrusted summaries into double-quoted shell templates.

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
- `--preview` already ran in Preflight C; skip a second preview unless flags changed.

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

`--output` unknown should not happen after Preflight A (CLI ≥ 1.10). If it does, stop; ask before upgrade.

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
| Dump `config.json` / print API keys | Inspect snippet only (provider, model, key SET/MISSING) |
| `ocr config provider` / `model` from the agent | Stop; user runs those in a real terminal |
| `npm install -g` without being asked | Print the install error and stop |
| `ocr llm test` / inspect snippet in **delegate** | Skip; print `provider/model/key/llm test: skipped` |
| Resume workspace **review** | Not supported (scan resume is) |
| Truncate stdout | `--output` + full file read |
