# OCR CLI extras

Canonical help: `ocr --help` and `ocr <command> --help`. Installed binary is typically `ocr` or `$HOME/.local/bin/ocr` (`@alibaba-group/open-code-review`).

## Preflight inspect

Print **only** `provider=`, `model=`, and `key=SET|MISSING`. Never print secrets. Missing, unreadable, or malformed config → empty provider/model and `key=MISSING` (exit 0). Review/scan only — do not run this snippet in delegate mode.

```bash
python3 - <<'PY'
import json, os
prov = model = ""
key_set = False
try:
    p = os.path.expanduser("~/.opencodereview/config.json")
    if os.path.isfile(p):
        with open(p, encoding="utf-8") as f:
            d = json.load(f)
        if not isinstance(d, dict):
            d = {}
        prov = str(d.get("provider") or "")
        active = (d.get("providers") or {}).get(prov) if isinstance(d.get("providers"), dict) else {}
        if not isinstance(active, dict):
            active = {}
        llm = d.get("llm") if isinstance(d.get("llm"), dict) else {}
        model = str(d.get("model") or active.get("model") or llm.get("model") or "")
        token = (
            active.get("api_key")
            or active.get("auth_token")
            or llm.get("auth_token")
            or ""
        )
        key_set = bool(str(token).strip())
except Exception:
    prov = model = ""
    key_set = False
print(f"provider={prov}")
print(f"model={model}")
print("key=SET" if key_set else "key=MISSING")
PY
```

Numeric version compare (OCR ≥ 1.10, Git ≥ 2.41). Not string compare (`1.9` is older than `1.10`).

```bash
ocr_ver=$(ocr version)
git_ver=$(git --version)
python3 -c 'import re,sys
text, need = sys.argv[1], tuple(int(x) for x in sys.argv[2].split("."))
m = re.search(r"(\d+)\.(\d+)", text)
sys.exit(0 if m and (int(m.group(1)), int(m.group(2))) >= need[:2] else 1)
' "$ocr_ver" 1.10
python3 -c 'import re,sys
text, need = sys.argv[1], tuple(int(x) for x in sys.argv[2].split("."))
m = re.search(r"(\d+)\.(\d+)", text)
sys.exit(0 if m and (int(m.group(1)), int(m.group(2))) >= need[:2] else 1)
' "$git_ver" 2.41
```

`ocr config get` does not exist. `ocr config provider` / `ocr config model` are interactive — never run them from an agent.

`ocr llm test` is the live connectivity check (review/scan only, after provider+model+key pass).

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
| `ocr: command not found` | Hard fail. Show `npm install -g @alibaba-group/open-code-review`. Do not auto-install. |
| CLI `< 1.10` / `unknown flag: --output` | Ask before `npm i -g @alibaba-group/open-code-review@latest`. Do not fall back to truncated stdout. |
| `unknown flag: --format` on delegate | CLI < 1.9. Rerun without `--format`; do not invent JSON fields. |
| Provider / model empty | User runs `ocr config provider` then `ocr config model` in their terminal. |
| `key=MISSING` | User reruns provider setup or `ocr config set providers.<name>.api_key "$KEY"`. Never invent keys. |
| `ocr llm test` fails | Key/network/quota. Stop. Offer delegate. |
| Rate limits | `--concurrency 2` or `--effort low` |
| Interrupted range/commit **review** | `ocr review … --resume <id>` from `ocr session list` |
| Interrupted **scan** | `ocr scan --resume <id>` |
| Position `start_line`/`end_line` both 0 | Locate from comment text + file read |

Do not dump `~/.opencodereview/config.json`. Use the inspect snippet only.
