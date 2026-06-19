# CLAUDE.md

本文件指导在此仓库工作的 Claude Code。架构细节见 `README.md`,当前修复计划见
`docs/superpowers/specs/2026-06-19-fix-leaked-claude-code-mvp-design.md`,此处只记录
"未来会话必须知道、但 README/代码里没有"的工作约定与关键事实。

## 项目是什么

Claude Code CLI 的源码归档(自述为 2026-03-31 泄露;由 fork 公开仓库而来,用于本地个人管理)。
当前目标:把这份源码修到**本地能正常运行、正常使用**。

## 运行前提(重要)

- **运行时是 Bun(≥1.3),不是 Node。** 入口靠 `bun run`,20 个文件使用真实 `Bun.*` API
  (`Bun.hash/semver/which/spawn/listen/stringWidth/YAML/JSONL` 等),Node 无法运行。
- **本机当前未安装 Bun**(仅有 Node v24 + npm),且 `node_modules` 未安装。实际修复前需:
  安装 Bun → `bun install`(必要时 `npm install --legacy-peer-deps`)。
- 入口:`src/entrypoints/cli.tsx`。常用脚本:`bun run start` / `bun run dev` / `bun run build` /
  `bun run typecheck`。
- feature flag:`bun:bundle` 已由 `plugins/bunBundleDev.ts` 在 dev 下 shim,默认全 `false`,
  可用 `FEATURE_FLAGS=A,B` 开启。**无需修改这套机制。**
- 缺失的内部包(`@ant/computer-use-*`、`@ant/claude-for-chrome-mcp`、`@anthropic-ai/mcpb`、
  `@anthropic-ai/sandbox-runtime`)是 `stubs/` 下的 no-op 桩;依赖它们的功能不工作。

## 当前任务:MVP 修复

目标档位 = **核心可用**:`bun run start` 启动 REPL 不崩溃 → 复用本机 `~/.claude` 的 OAuth 凭证
接入 API 正常多轮对话 → 核心工具(Bash/Read/Write/Edit/Grep/Glob)+ 基本命令(/help、/clear、
/config 等)可用。被桩的内部功能、voice/remote/server/bridge 等不在范围。

执行方案 = **运行路径驱动 + 局部类型校验**:沿 `cli.tsx → init → REPL → 认证 → QueryEngine → 工具`
深度优先修;每改一处只对改动文件做 typecheck;非核心/未触碰文件的类型错误不追。

**完成判定 / 停止条件**:见设计文档第 6 节。简言之——独立验证 sub-agent 逐条核验"依赖装好 /
能启动 / 已认证 / 对话打通 / 核心工具各跑通一次 / 基本命令不崩 / 改动文件 typecheck 通过",七项
全过即目标完成、停止;命中护栏(缺失模块需重实现、需系统级授权、同一错误修 ≥3 次仍不过、需联网拉
缺失源码等)则暂停转人工。用 `/goal` 等自动推进时以此为终止信号,防止空转或目标蔓延。

## 修复规则与约束(必须遵守)

1. **最小改动优先**:核心路径认真修;非核心/缺失模块处用最小桩或禁用跳过;不重实现缺失模块的内部逻辑。
2. **改动留痕**:每处因泄露而修的地方加注释 `// LEAK-FIX: <原因>`;按逻辑分组**小步 git 提交**。
3. **切断对外上报/自更**:遥测(OpenTelemetry)、分析(GrowthBook)、自动更新、远程托管设置/策略限制
   在本地构建中**失效且安全失效**(不回传、不自更、不致崩)。
4. **不重构正常代码**:只修坏的,不顺手重构/优化能正常工作的代码。
5. **被桩功能优雅降级**:提示"不支持"而非崩溃。
6. **风格统一**:修复/新增代码贴合原 Claude Code 的命名、注释密度、写法。

## 验证方式(必须遵守)

- **验证由独立 sub-agent 执行**,与负责修复的 agent 分离——修复 agent 只实现,验证 agent 独立运行
  应用、回报真实终端输出,避免"自己改自己验"。验证 agent 不改源码。
- 主验证:复用本机 `~/.claude` 配置/凭证,`bun run start` 实测对话 + 逐个跑通核心工具。
- 给证据,不空口声称"修好了"。

## Git

- remote `origin` = https://github.com/lgk-code/myclaude(个人管理用,公开)。
- 默认分支 `main`;换行符由 `.gitignore` + `.gitattributes`(统一 LF)管理。
- 提交信息结尾附:`Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。
- 推送属对外发布,**先确认再推**,不要自动推送。
