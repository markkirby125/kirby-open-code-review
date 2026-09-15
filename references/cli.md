# OCR CLI extras

Canonical help: `ocr --help` and `ocr <command> --help`. Installed binary on this machine is typically `ocr` or `$HOME/.local/bin/ocr` (`@alibaba-group/open-code-review`).

## Flag matrix (v1.12.x)

`Y` = flag exists. Confirm with `ocr <cmd> --help` after upgrades.

| Flag | review | scan | delegate preview/rule |
|---|---|---|---|
| `--audience` / `--format` / `--output` / `--color` | Y | Y | format only (no audience/output; no sarif) |
| `-b, --background` | Y | Y | Y |
| `-B, --background-file` | Y | **no** | Y |
| `--effort` | Y | **no** | **no** |
| `--timeout` (minutes) | Y | Y | **no** |
| `--resume` | range/commit only; **not** workspace | Y | **no** |
| `--from` / `--to` / `--commit` / `--repo` / `--exclude` / `--rule` | Y | repo/exclude/rule; scan uses `--path` not from/to/commit | Y |
| `--path` | **no** | Y | **no** |

`--background-file` wins over `-b`. Abort if file > 1 MiB or sanitized text > 8000 chars. Summarize rather than truncate. Do not interpolate untrusted summaries into double-quoted shell templates (`$()`, backticks, `$vars`).

## Shared knobs

| Flag | Notes |
|---|---|
| `--effort low\|medium\|high` | **Review only.** Default medium = 2 rounds. Group wall ≈ OCR `--timeout` minutes × rounds. |
| `--timeout <min>` | OCR clock: minutes per concurrent task, default 15. Not Grok `timeout` (ms). |
| `--concurrency <n>` | Default 8. Lower if the provider rate-limits. |
| `--exclude '<a,b>'` | gitignore-style; merged with `rule.json` excludes. |
| `--provider` / `--model` | This-run override. `ocr llm providers` lists built-ins. |
| `--max-tokens` | Per-group prompt ceiling. |
| `--max-tokens-budget` | Stop dispatch when total tokens exceeded; skipped files `failed(budget)`; exit 0 unless **every** selected item failed. |
| `--no-filter` | Skip LLM post-filter of comments (review). |
| `--format text\|json\|sarif` | Agents: `json`. Delegate: text or json only (no sarif). |
| `--repo <path>` | Git root when cwd is elsewhere. |
| `--rule <path>` | Highest-priority rule file. |

Scan-only: `--path` (comma-separated repo-relative files/dirs), `--no-plan`, `--no-dedup`, `--no-summary`, `--batch none\|by-language\|by-directory`.

## Rules

Priority: `--rule` > `<repo>/.opencodereview/rule.json` > `~/.opencodereview/rule.json` > built-in. First matching user rule replaces the system rule unless `merge_system_rule: true`.

```json
{
  "rules": [
    {
      "path": "**/*.ts",
      "rule": "Validate required parameters before use",
      "merge_system_rule": true
    }
  ]
}
```

Preview: `ocr rules check path/to/file.ts`

## Sessions / viewer

```bash
ocr session list --json
ocr session show <id>
ocr session comments --json [--severity critical,high] <id>
ocr session compare <id-a> <id-b>
ocr viewer --open=never   # URL only; default bind localhost:5483
```

Resume: `ocr review --from … --to … --resume <id>` or `ocr review --commit … --resume <id>` (same target). Workspace **review** resume is unsupported. Scan: `ocr scan --resume <id>`.

## Troubleshooting

| Symptom | Action |
|---|---|
| `ocr: command not found` | Consent, then `npm install -g @alibaba-group/open-code-review` |
| `unknown flag: --output` | CLI < 1.10. Ask before upgrade. Do not fall back to truncated stdout. |
| `unknown flag: --format` on delegate | CLI < 1.9. Rerun without `--format`; do not invent JSON fields. |
| LLM connection error | `ocr llm test`. User runs `ocr config provider` / `ocr config model`. Never invent keys. Or switch to delegate. |
| Rate limits | `--concurrency 2` or `--effort low` |
| Interrupted range/commit **review** | `ocr review … --resume <id>` from `ocr session list` |
| Interrupted **scan** | `ocr scan --resume <id>` |
| Position `start_line`/`end_line` both 0 | Locate from comment text + file read |

Do not open `~/.opencodereview/config.json` (secrets).
