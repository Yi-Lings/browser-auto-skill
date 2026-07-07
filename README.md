# Browser Auto Script Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

浏览器自动化脚本编写最佳实践与踩坑记录 — 基于 [Playwright CLI](https://www.npmjs.com/package/playwright-cli)，适用于各类 AI 编程助手。

> **⚠️ 此为 AI 助手指令集，非独立可执行工具。**  
> 由 AI 编程助手（Claude Code、VS Code Codex/Agent 模式 等）在用户提出浏览器自动化需求时读取，按其中的方法论现场编写脚本。

---

## 功能

- **6 步自动化工作流** — 打开浏览器 → 检查登录 → 发现 API 端点 → 提取数据 → 处理 → 关闭
- **动态请求 ID 发现** — 不硬编码 ID，从 `playwright-cli requests` 输出通过正则提取
- **Windows 兼容** — `.cmd` 包装器、Git Bash 自动检测、跨平台临时目录
- **已知坑点文档化** — `fetch()` 认证缺失、CRLF 行尾、Python 沙箱、编码问题等
- **自包含脚本** — 通过 heredoc 内嵌数据处理代码（Python/JS），无外部依赖
- **自检清单** — 编写任何自动化脚本前的 10 项检查点

---

## 使用方法

### 作为 AI 助手技能

向 AI 说出以下任一：

- "写个浏览器自动化脚本"
- "帮我用 playwright-cli 自动化这个操作"
- "Browser automation with playwright-cli"
- "编写自动化脚本抓取页面数据"

AI 将读取本指南并按照其方法论生成完整脚本。

### 手动参考

无需 AI 协助时，可直接阅读 [`SKILL.md`](SKILL.md) 中的模板章节自行组装：

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
WIN_TMP=$(python -c "import tempfile; print(tempfile.gettempdir())" 2>/dev/null || echo "/tmp")

# 步骤 1-6：遵循 SKILL.md 中的工作流
```

---

## 安装方式
1.对智能体直接说

```bash
帮我安装这个skill "https://github.com/Yi-Lings/browser-auto-skill/"
```

2.手动安装

### Claude Code

将本仓库克隆到项目的 `.claude/skills/` 目录下：

```bash
# 在项目根目录执行
mkdir -p .claude/skills
git clone https://github.com/Yi-Lings/browser-auto-skill.git .claude/skills/browser-auto-script
```

Claude Code 会自动加载 `SKILL.md` 作为可用技能。

### VS Code (Codex / Agent 模式)

将 `SKILL.md` 的内容复制到项目根目录的 `.github/copilot-instructions.md` 中，或直接将文件链接加入 Codex 上下文。

### 其他 AI 编程工具

大多数 AI 编程助手支持通过 Markdown 文件提供自定义指令。将 `SKILL.md` 作为上下文指令文件引入即可。

---

## 环境要求

| 依赖 | 版本 | 说明 |
|------|------|------|
| [Node.js](https://nodejs.org/) | ≥ 18 | 运行 `npx playwright-cli` |
| [Playwright CLI](https://www.npmjs.com/package/playwright-cli) | latest | `npx playwright-cli` |
| [Python](https://www.python.org/) | ≥ 3.8 | 可选，用于数据处理 |
| [Git Bash](https://git-scm.com/) | ≥ 2.x | Windows 必需 |

---

## 项目结构

```
browser-auto-skill/
├── SKILL.md      # 核心 — 工作流、模式、坑点
├── README.md     # 本文件
└── LICENSE       # MIT
```

本仓库仅分发指令文件 `SKILL.md` 加文档。脚本模板和代码片段以内嵌 heredoc 形式写在 `SKILL.md` 中，无需外部运行时文件。

---

## 已知问题 / 坑点

详见 `SKILL.md` §4。要点速览：

| 问题 | 解决方案 |
|------|----------|
| `page.evaluate(fetch())` 不带认证 | 改用 `playwright-cli response-body <id>` |
| 每次会话请求 ID 不同 | 从 `requests` 输出用 `grep -oE` 动态解析 |
| Git Bash 与 Windows Python 下 `/tmp` 指向不同目录 | 用 `python -c "import tempfile; ..."` 获取统一临时目录 |
| Git Bash 中 `dirname`/`grep` 找不到 | 在 `.cmd` 包装器中将 Git `usr/bin` 加入 `PATH` |
| CMD 中文编码乱码 | `.cmd` 文件避免中文，中文全部放在 bash 脚本中 |
| Windows Store 版 Python 沙箱（退出码 49） | 用 `python` 而非 `python3` |

---

## License

[MIT](LICENSE) © Yi-Lings
