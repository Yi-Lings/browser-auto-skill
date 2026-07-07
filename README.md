# Browser Auto Script Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A **Claude Code skill** for browser automation script writing with [Playwright CLI](https://www.npmjs.com/package/playwright-cli). Summarizes best practices, general workflow patterns, and Windows platform troubleshooting.

> **⚠️ This is an AI assistant skill, not a standalone tool.**  
> It is designed to be loaded by [Claude Code](https://claude.ai/code) (or compatible AI coding agents) when the user requests browser automation tasks — the AI reads `SKILL.md` and follows its guidance to write scripts on the fly.

---

## Features

- **6-step automation workflow** — open browser → check login → discover API endpoints → extract data → process → close
- **Dynamic request ID discovery** — no hardcoded IDs; parse from `playwright-cli requests` output via regex
- **Windows compatibility** — `.cmd` wrapper, Git Bash detection, cross-platform temp directory
- **Known pitfalls documented** — authentication in `fetch()`, CRLF line endings, Python sandbox, encoding issues
- **Self-contained scripts** — embed data-processing code (Python, JS) via heredoc; no external dependencies
- **Self-check checklist** — 10-point checklist before writing any automation script

---

## Usage

This skill is consumed by Claude Code during a session. To trigger it, ask the AI:

- "写个浏览器自动化脚本"
- "帮我用 playwright-cli 自动化这个操作"
- "Browser automation with playwright-cli"
- "编写自动化脚本抓取页面数据"

The AI will read the skill and follow its methodology to produce a complete script.

### Manual script generation

If you want to generate a script without AI assistance, read the **template** sections in [`SKILL.md`](SKILL.md) and assemble your own:

```bash
# Minimal bash skeleton
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
WIN_TMP=$(python -c "import tempfile; print(tempfile.gettempdir())" 2>/dev/null || echo "/tmp")

# Step 1-6: follow the workflow in SKILL.md
```

---

## Requirements

| Dependency | Version | Notes |
|-----------|---------|-------|
| [Node.js](https://nodejs.org/) | ≥ 18 | For `npx playwright-cli` |
| [Playwright CLI](https://www.npmjs.com/package/playwright-cli) | latest | `npx playwright-cli` |
| [Python](https://www.python.org/) | ≥ 3.8 | Optional, for data processing |
| [Git Bash](https://git-scm.com/) | ≥ 2.x | Required on Windows |

---

## Project Structure

```
browser-auto-skill/
├── SKILL.md      # Core skill definition — workflow, patterns, pitfalls
├── README.md     # This file
└── LICENSE       # MIT
```

This skill intentionally ships only the instruction file (`SKILL.md`) plus documentation. Script templates and code snippets are embedded inside `SKILL.md` as heredoc-ready examples — no external runtime files needed.

---

## Known Issues / Pitfalls

All documented pitfalls are in `SKILL.md` §4 ("已知坑与解决方案"). Highlights:

| Issue | Solution |
|-------|----------|
| `page.evaluate(fetch())` lacks auth | Use `playwright-cli response-body <id>` instead |
| Request IDs change per session | Parse dynamically with `grep -oE` from `requests` output |
| `/tmp` differs between Git Bash and Windows Python | Use `python -c "import tempfile; ..."` |
| Git Bash `dirname`/`grep` not found | Add Git `usr/bin` to `PATH` in `.cmd` wrapper |
| CMD Chinese encoding garbled | Avoid Chinese in `.cmd` files; keep all UTF-8 in bash |
| Windows Store Python sandbox (exit code 49) | Use `python` not `python3` |

---

## License

[MIT](LICENSE) © Yi-Lings
