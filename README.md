# Claude Code — 本地可运行修复版

这是 Claude Code CLI 的源码。原始源码中缺少若干**构建期资源**(被 bundler 内联的文件、原生二进制)和若干**内部包**(以 no-op 桩替代),直接运行会在多处崩溃或卡死。本仓库对其做了**最小改动修复**,使其能在本地以 MVP 形态正常运行、正常使用。

> 修复完成于 2026-06。所有修复均以 `// LEAK-FIX:` 注释标记,且尽量提供环境变量以恢复原行为。

---

## 现在能用什么

经独立验证通过的能力(MVP):

- ✅ 启动交互式 REPL(完整 TUI 界面)
- ✅ 复用本机 `~/.claude` 的 OAuth 登录直接对话(无需重新认证)
- ✅ 端到端流式对话
- ✅ 核心工具:`Bash` / `Read` / `Write` / `Edit` / `Glob` / `Grep`
- ✅ 斜杠命令:`/help`、`/config`、`/clear` 等
- ✅ 非交互模式:`bun run start -p "你的问题"`

---

## 快速开始

**前提**:[Bun](https://bun.sh) ≥ 1.3(项目运行时;运行本身不需要 Node,npm 仅用于回退安装)。

```powershell
# 1. 安装 Bun (Windows PowerShell)
irm bun.sh/install.ps1 | iex
# 装好后 bun 在 C:\Users\<你>\.bun\bin,可能需要把它加入 PATH 或重开终端

# 2. 安装依赖(在项目目录)
bun install            # 若 peer deps 冲突可改用: npm install --legacy-peer-deps

# 3. 运行
bun run start                 # 交互式 REPL
bun run start -- --help       # 查看参数
bun run start -p "say hi"     # 非交互单轮
```

- **认证**:默认复用本机 `~/.claude` 的现有登录(OAuth)。它与官方安装的 `claude` 共用同一份配置/凭证目录。
- **与官方 `claude` 的区别**:本项目用 `bun run start` 从源码运行,是独立进程;启动后显示版本 `1.0.0-dev (Claude Code)` 即表示跑的是本仓库这一版,而非全局安装的官方版。
- **首次启动较慢/MCP 提示**:若你 `~/.claude` 配了 MCP 服务器(如 `claude-in-chrome` 需浏览器扩展),启动时连接可能拖慢。需要干净快速启动时可加 `--strict-mcp-config` 隔离个人 MCP 配置。

| 脚本 | 说明 |
|---|---|
| `bun run start` | 运行 CLI |
| `bun run dev` | 热重载运行(`--watch`) |
| `bun run build` | 生产打包 |
| `bun run typecheck` | 类型检查(注:存在大量非核心类型错误,不影响运行) |

---

## 修复了什么(根因)

四处关键修复,均源于泄露包"少文件 / 少方法"。`-p` 非交互模式不加载交互 REPL 模块,所以这些问题里有一部分只在交互模式暴露。

| 提交 | 问题 | 修复 |
|---|---|---|
| `9ea96d7` | 远程"最低版本"强制升级门会直接终止 `1.0.0-dev`;启动期插件/市场自动更新逐个 git 操作挂住启动 | 两者默认跳过(切断对外上报/自更)。恢复:`CLAUDE_CODE_ENFORCE_MIN_VERSION=1` / `CLAUDE_CODE_ENABLE_PLUGIN_AUTOUPDATE=1` |
| `fd84669` | `@anthropic-ai/sandbox-runtime` 是 no-op 桩,缺静态方法 → `isSandboxingEnabled()` 抛 `TypeError` → 非交互启动**静默挂死** | 桩缺方法时安全降级(平台视为不支持 / 依赖不可用),sandbox 跳过 |
| `501b7cd` | 缺 vendored `rg.exe` → Grep 报 ENOENT;BashTool 调用桩缺的 sandbox 方法 → Bash 崩 | Grep:缺 vendored 时回退系统 `rg`(并把 `rg.exe` 放回 vendored 路径,已 gitignore);Bash:守卫 `annotateStderrWithSandboxFailures` / `cleanupAfterCommand` |
| `2aa39a8` | **交互卡死总开关**:缺失 `src/utils/ultraplan/prompt.txt` → 顶层 `require()` 抛错 → 连锁致 `REPL.js` 模块加载崩在已进入 raw 模式的终端上 → 终端冻死、Ctrl+C 失效 | 该 `require` 加 try/catch 兜底(ultraplan 为非核心功能) |

---

## 范围与限制(MVP)

- **被桩的内部功能不工作**:computer-use(电脑操作)、Chrome MCP、sandbox(沙箱)、mcpb(桌面扩展打包)。这些依赖未随源码提供的内部包,保持"不可用/优雅降级"。
- **对外上报与自更已切断**:遥测(OpenTelemetry)、分析(GrowthBook/Datadog)、自动更新、远程托管设置在本地构建中默认失效(不回传、不自更)。
- **类型检查未追平**:Bun 直接执行 TypeScript(剥离类型,不做类型检查),`tsc` 的约 7000 个错误绝大多数在非核心路径,不影响运行;本次只追改动到的文件。
- **个人 MCP 配置**:`~/.claude` 中的 MCP 服务器在交互启动时会尝试连接,某些(如需浏览器扩展的)会拖慢启动;用 `--strict-mcp-config` 可隔离。

---

## 相关文档

- 设计与停止条件:`docs/superpowers/specs/2026-06-19-fix-leaked-claude-code-mvp-design.md`
- 实现计划:`docs/superpowers/plans/2026-06-19-fix-leaked-claude-code-mvp.md`
- 在本仓库工作的约定与踩坑记录:`CLAUDE.md`

---

## 技术栈

| 类别 | 技术 |
|---|---|
| 运行时 | [Bun](https://bun.sh) |
| 语言 | TypeScript |
| 终端 UI | React 19 + 自定义 [Ink](https://github.com/vadimdemedes/ink) 渲染器 |
| 校验 | [Zod](https://zod.dev) |
| 协议 | [MCP](https://modelcontextprotocol.io)、LSP |
| API | Anthropic SDK |

---

## 免责声明

本仓库源自一个公开仓库的 fork,仅用于本地个人学习与使用。原始源码版权归 [Anthropic](https://www.anthropic.com) 所有。
