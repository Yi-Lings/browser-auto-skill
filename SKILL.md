---
name: browser-auto-script
description: 总结 playwright-cli 浏览器自动化脚本编写的最佳实践、通用流程与 Windows 平台踩坑记录。遇到自动化需求时先读此 skill。
allowed-tools: Bash(playwright-cli:*) Bash(python:*) Bash(cmd.exe:*) Read(*) Write(*) Edit(*)
---

# 浏览器自动化脚本编写指南

基于 DeepSeek API 用量查询脚本的实际开发经验，提炼出的通用方法论和踩坑记录。

---

## 目录

1. [Playwright CLI 核心命令](#1-playwright-cli-核心命令)
2. [脚本架构与流程](#2-脚本架构与流程)
3. [Windows 平台兼容方案](#3-windows-平台兼容方案)
4. [已知坑与解决方案](#4-已知坑与解决方案)
5. [模板：脚本各部分要点](#5-模板脚本各部分要点)
6. [写脚本时的自检清单](#6-写脚本时的自检清单)

---

## 1. Playwright CLI 核心命令

| 命令 | 用途 | 说明 |
|------|------|------|
| `playwright-cli open [--headed] [--persistent] <url>` | 打开浏览器 | `--headed` 显示窗口；`--persistent` 保存登录状态 |
| `playwright-cli snapshot [--boxes]` | 获取页面 DOM 快照 | `--boxes` 含坐标和 ref，用于判断页面状态 |
| `playwright-cli requests` | 列出所有网络请求 | 关键：发现后端 API 端点及其请求 ID |
| `playwright-cli response-body <id>` | 获取指定请求的响应体 | 提取 API 返回的 JSON 数据 |
| `playwright-cli click <ref>` | 点击元素 | 通过 snapshot 中的 ref 定位 |
| `playwright-cli type <ref> <text>` | 输入文本 | 同上 |
| `playwright-cli close` | 关闭浏览器 | |

---

## 2. 脚本架构与流程

### 通用 6 步流程

```
[1/6] 打开浏览器
        │
[2/6] 检查登录状态 ←── 未登录 → 提示手动登录后继续（headed 模式）
        │ 已登录
[3/6] 发现 API 端点 ←── playwright-cli requests → 正则解析请求 ID
        │
[4/6] 提取数据     ←── playwright-cli response-body <id> → 保存为 JSON
        │
[5/6] 处理数据     ←── Python（内嵌或单独脚本）→ 生成报告
        │
[6/6] 关闭浏览器
```

### 关键设计决策

**1. 语言选择：永远用 bash 脚本，不要用 PowerShell**

Playwright CLI 的 npm 包同时提供了 `playwright-cli`（bash）和 `playwright-cli.ps1`（PowerShell）。实测 PowerShell wrapper 的 Node.js 错误流与 stdout 混叠，导致 `snapshot` 等命令报 `session.js:60` 错误且无法 catch。bash 版本无此问题。

→ **主逻辑写 bash（.sh），Windows 用户通过 .cmd 包装调用 Git Bash**

**2. Windows 入口：.cmd 批处理文件**

```cmd
@echo off
set "SCRIPT_DIR=%~dp0"
cd /d "%SCRIPT_DIR%"

REM 自动检测 Git Bash
set "BASH="
where bash >nul 2>&1 && set "BASH=bash"
if not defined BASH (if exist "D:\Git\usr\bin\bash.exe" set "BASH=D:\Git\usr\bin\bash.exe")
if not defined BASH (if exist "%ProgramFiles%\Git\usr\bin\bash.exe" set "BASH=%ProgramFiles%\Git\usr\bin\bash.exe")
if not defined BASH (if exist "%ProgramFiles(x86)%\Git\usr\bin\bash.exe" set "BASH=%ProgramFiles(x86)%\Git\usr\bin\bash.exe")

REM 关键：把 Git usr/bin 加入 PATH，否则 dirname/grep 找不到
for %%i in ("%BASH%") do set "BASH_DIR=%%~dpi"
set "PATH=%BASH_DIR%;%PATH%"

"%BASH%" "%SCRIPT_DIR%script.sh" %*
```

**3. 动态请求 ID 发现**

不要硬编码请求 ID。用正则从 `playwright-cli requests` 输出中提取：

```bash
REQUESTS=$(playwright-cli requests 2>/dev/null || true)
SUMMARY_ID=$(echo "$REQUESTS" | grep -oE '[0-9]+\.\s+\[GET\]\s+\S+get_user_summary' | grep -oE '^[0-9]+')
AMOUNT_ID=$(echo "$REQUESTS" | grep -oE '[0-9]+\.\s+\[GET\]\s+\S+usage/amount' | grep -oE '^[0-9]+')
```

**4. 持久化登录**

```bash
playwright-cli open --headed --persistent <url>
```

- `--persistent` 将登录状态保存到 `.playwright-cli/` 目录
- 首次手动登录，后续复用
- 无头模式也要加 `--persistent` 才能利用已有 cookie

**5. 自包含脚本：内嵌数据处理代码**

避免多文件依赖，将 Python（或其他处理代码）作为 heredoc 内嵌在 bash 脚本中：

```bash
cat > /tmp/gen_report.py << 'PYEOF'
# ... Python 代码 ...
PYEOF

python /tmp/gen_report.py "$REPORT_DIR"
rm -f /tmp/gen_report.py
```

---

## 3. Windows 平台兼容方案

### 坑：`/tmp` 在 Git Bash 和 Windows Python 中指向不同路径

| 环境 | `/tmp` 实际路径 |
|------|----------------|
| Git Bash | `/tmp` → Git Bash 内部 tmp |
| Windows Python (`python`) | `/tmp` → `C:\tmp\` |

使用 `python -c` 获取统一的临时目录：

```bash
WIN_TMP=$(python -c "import tempfile; print(tempfile.gettempdir())" 2>/dev/null || echo "/tmp")

# 然后用 $WIN_TMP 代替 /tmp
TMP_FILE="${WIN_TMP}/my_data.json"
playwright-cli response-body $ID > "$TMP_FILE"

# Python 脚本也通过命令行参数接收 WIN_TMP
python script.py "$REPORT_DIR" "$WIN_TMP"
```

### 坑：Git Bash `dirname` / `grep` 找不到

当 `.cmd` 用全路径 `D:\Git\usr\bin\bash.exe` 调用 bash 时，PATH 没设，`dirname`、`grep` 等命令找不到。

**解决**：在 `.cmd` 中将 Git 的 `usr/bin` 加到 PATH（见上方模板）。

### 坑：CMD 中文编码乱码

CMD 默认 GBK（中国区）或代码页 936，UTF-8 文件中的中文显示为乱码。

- `.cmd` 文件正文避免中文（用英文消息，中文全交给 bash 脚本输出）
- 或保存 `.cmd` 为 ANSI 编码

### 坑：Python 沙箱（Windows Store 版）

`python3` 命令解析到 Windows Store 版 Python，有文件 I/O 沙箱限制，退出码 49。

**解决**：永远用 `python`（`py` 启动器会找到 PATH 中第一个、通常是完整安装版）。

---

## 4. 已知坑与解决方案

### ⚠️ 坑 1: `page.evaluate(fetch())` 不携带认证

```js
// ❌ 浏览器内 fetch() 不带 Cookie/Authorization header
await page.evaluate(async () => {
  const r = await fetch('/api/v0/...');
  return await r.text();
});
```

**必须**用 `playwright-cli response-body <id>` 读取已存在的网络请求。

### ⚠️ 坑 2: 请求 ID 不固定

每次 session 的请求 ID 不同。必须先用 `playwright-cli requests` 动态发现。

### ⚠️ 坑 3: UI 数据显示不全

页面 DOM 只展示聚合数据（总 Tokens、总请求数），不展示缓存命中/未命中、各模型消费金额等。**必须走 API 层**。

### ⚠️ 坑 4: 首次需 headed 登录

- `--headless` 无法处理验证码/2FA
- 脚本应检测登录状态（`snapshot --boxes | grep -qiE "sign.?in|登录|手机号"`）
- 未登录时 headed 模式等待用户操作，headless 模式提示用户改用 headed

```bash
if echo "$SNAPSHOT" | grep -qiE "sign.?in|登录"; then
    if $HEADLESS; then
        echo "请先运行（去掉 --headless）登录一次"
        exit 1
    fi
    echo "请在浏览器中登录后按 Enter..."
    read -r
fi
```

### ⚠️ 坑 5: 等待/延迟

Playwright 操作需要足够延迟让页面/API 响应：

| 操作 | 建议延迟 |
|------|---------|
| 打开页面后 | 3s |
| 取 snapshot 前 | 3s |
| 请求 API 列表前 | 5s |
| 提取 response-body 前 | 1s |
| requests 重试间隔 | 2s，最多 3 次 |

### ⚠️ 坑 6: 文件有效性校验

不要假设 `response-body` 输出了一定是有效 JSON。写入后校验：

```bash
for f in "$TMP_FILE"; do
    sz=$(wc -c < "$f" 2>/dev/null || echo 0)
    if [ "$sz" -lt 10 ]; then
        echo "文件过小，放弃"
        exit 1
    fi
done
```

---

## 5. 模板：脚本各部分要点

### bash 主脚本 (`script.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

# 路径
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPORT_DIR="${SCRIPT_DIR}/output"
WIN_TMP=$(python -c "import tempfile; print(tempfile.gettempdir())" 2>/dev/null || echo "/tmp")

HEADLESS=false
[[ "${1:-}" == "--headless" ]] && HEADLESS=true

mkdir -p "$REPORT_DIR"

# 颜色输出函数
info()  { echo -e "✓ $1"; }
warn()  { echo -e "⚠ $1"; }

# Step 1-6 见上面的流程章节
```

### Python 数据处理脚本 (`gen_report.py`)

通过 `sys.argv` 接收路径参数：

```python
import json, sys, os
from openpyxl import Workbook

REPORT_DIR = sys.argv[1]    # 输出目录
WIN_TMP = sys.argv[2]       # 临时目录（Windows 兼容）

data = json.load(open(os.path.join(WIN_TMP, "my_data.json")))
# ... 处理和输出 ...
```

---

## 6. 写脚本时的自检清单

- [ ] 请求 ID 是否动态发现（不是硬编码）？
- [ ] 是否用了 `--persistent` 保存登录状态？
- [ ] 首次运行跳转到登录页时，是否给了用户清晰的 headed 模式提示？
- [ ] headless 模式下未登录是否优雅退出（不是卡在 `read`）？
- [ ] 临时文件路径是否做了 Windows 兼容（`$WIN_TMP` 而非直接 `/tmp`）？
- [ ] `.cmd` 入口是否把 Git `usr/bin` 加到 PATH？
- [ ] Python 命令用的是 `python`（不是 `python3`）？
- [ ] 脚本对 `playwright-cli` 的失败（stderr）有忽略处理（`2>/dev/null || true`）？
- [ ] 提取的数据有非空/字节数校验？
- [ ] 内嵌代码用了 heredoc（`<< 'PYEOF'` 而非 `<< PYEOF`，避免变量展开）？
- [ ] 临时文件在脚本退出时是否清理？

---

## 触发

用户说"自动化浏览器脚本"、"playwright 脚本"、"浏览器自动化"、"编写自动化脚本"、"做自动化流程"、"写个脚本自动打开网页"或任何涉及 playwright-cli 浏览器自动化脚本编写的需求。先读此 skill 再动手。
