# OCR CLI extras

Canonical help: `ocr --help` and `ocr <command> --help`. Installed binary on this machine is typically `ocr` or `$HOME/.local/bin/ocr` (`@alibaba-group/open-code-review`).

## Review / scan flags worth knowing

| Flag | Notes |
|---|---|
| `--effort low\|medium\|high` | Extra review rounds. Default medium = 2. Group wall time ≈ `--timeout` × rounds. |
| `--timeout <min>` | Per concurrent task, default 15. |
| `--concurrency <n>` | Default 8. Lower if the provider rate-limits. |
| `--exclude '<a,b>'` | gitignore-style; merged with `rule.json` excludes. |
| `--background` / `--background-file` | File wins. Abort if file > 1 MiB or sanitized text > 8000 chars. |
| `--provider` / `--model` | This-run override. `ocr llm providers` lists built-ins. |
| `--max-tokens` | Per-group prompt ceiling. |
| `--max-tokens-budget` | Stop dispatch when total tokens exceeded; skipped files `failed(budget)`; exit 0 unless **every** selected item failed. |
| `--no-filter` | Skip LLM post-filter of comments. |
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

Resume only range or commit reviews: `ocr review --from … --to … --resume <id>` (same target).

## Troubleshooting

| Symptom | Action |
|---|---|
| `ocr: command not found` | Consent, then `npm install -g @alibaba-group/open-code-review` |
| `unknown flag: --output` | CLI < 1.10. Ask before upgrade. Do not fall back to truncated stdout. |
| `unknown flag: --format` on delegate | CLI < 1.9. Rerun without `--format`; do not invent JSON fields. |
| LLM connection error | `ocr llm test`. User runs `ocr config provider` / `ocr config model`. Never invent keys. Or switch to delegate. |
| Rate limits | `--concurrency 2` or `--effort low` |
| Interrupted range/commit | `--resume <id>` from `ocr session list` |
| Position `start_line`/`end_line` both 0 | Locate from comment text + file read |

Do not open `~/.opencodereview/config.json` (secrets).
