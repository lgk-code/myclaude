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
- **Bun 已安装**:`C:\Users\logic\.bun\bin\bun.exe`(v1.3.14),**不在默认 PATH**。在 PowerShell 里先
  `$env:Path = "$env:USERPROFILE\.bun\bin;$env:Path"` 再用 `bun`。依赖已 `bun install`(367 包,直接成功,无需 npm 回退)。
- 入口:`src/entrypoints/cli.tsx`。常用脚本:`bun run start` / `bun run dev` / `bun run build` /
  `bun run typecheck`。
- feature flag:`bun:bundle` 已由 `plugins/bunBundleDev.ts` 在 dev 下 shim,默认全 `false`,
  可用 `FEATURE_FLAGS=A,B` 开启。**无需修改这套机制。**
- 缺失的内部包(`@ant/computer-use-*`、`@ant/claude-for-chrome-mcp`、`@anthropic-ai/mcpb`、
  `@anthropic-ai/sandbox-runtime`)是 `stubs/` 下的 no-op 桩;依赖它们的功能不工作。

## 运行与调试经验(踩坑记录 —— 避免重复犯)

> 这些是实修中真实踩到的坑,务必先读再动手运行。

1. **绝不在前台直接跑 `bun run start` / `-p` 并干等。** 它是交互式 TUI:
   - 无 TTY/管道环境下会**无限挂起**;且管道下 **stdout 是块缓冲的——进程不退出就完全看不到任何输出**,
     极易误判为"卡死/没反应"(本会话曾因此让进程空挂约 40 分钟)。
   - **正确姿势**:用 `--debug-file <绝对路径>` 把日志写文件(绕过缓冲),`tail`/轮询该文件看真实进度;
     给 stdin 喂 EOF —— `cmd /c "bun run start -p ""...""  < NUL 2>&1"`;用**后台运行 + 监测脚本**而非前台等待。
   - 结束后务必 `Get-Process bun | Stop-Process -Force` 清理,否则挂起的 bun 进程会堆积(每个 ~100-280MB)。
2. **跑本机真实 `~/.claude` 配置会拖入大量启动期网络/git 同步**(插件自动更新、市场刷新、MCP 服务器连接),
   逐个慢/挂,是"启动半天没动静"的主因。验证**核心 boot/auth/chat** 时用 **`--strict-mcp-config`** 隔离个人
   MCP 配置(尤其 `claude-in-chrome` 需浏览器扩展、headless 连不上会挂)。
3. **Bun 直接执行 TS、不做类型检查**:`tsc` 有 7263 个错误,但绝大多数**不阻断运行**。判断"能否跑"看
   运行时报错,**不要**去追 `tsc` 的非核心错误(方案 C 明确不追)。
4. **已知运行时缺失(多为非致命,但记牢)**:
   - `src/utils/vendor/ripgrep/x64-win32/rg.exe` 缺失(vendored ripgrep 未随泄露包含)→ 影响 Grep(Task 5)。
   - `fflate` 包缺失(`src/utils/dxt/zip.ts`)→ 市场 GCS 拉取失败,已回退 git(非致命)。
5. **本工具的 PowerShell 不允许前台长 `Start-Sleep`**;需要等待时用后台命令(`run_in_background`)+ 监测脚本,
   或 Monitor 的 until-loop,不要用前台 sleep 干等。
6. **已在代码里默认切断的启动期阻塞/上报**(均 `// LEAK-FIX`,可用对应 env 恢复):远程强制升级门
   (`src/utils/autoUpdater.ts` `assertMinVersion`)、启动期插件/市场自动更新
   (`src/utils/plugins/pluginAutoupdate.ts`)。

## 自主操作授权(用户已授权)

- **为完成本项目所需的下载/安装,可自行执行,无需逐次询问。** 包括:安装 Bun 运行时、
  `bun install` / `npm install --legacy-peer-deps`、修复所需的开发工具与类型包等。完成后简要
  说明装了什么即可。
- 边界:本授权仅覆盖"为完成本项目修复所需的下载/安装(入站)"。**仍需先确认**的高风险/对外动作
  不在此列——尤其 `git push`(见 Git 一节)、删除非自己创建的内容、向外部服务发送数据等。

## 当前任务:MVP 修复

目标档位 = **核心可用**:`bun run start` 启动 REPL 不崩溃 → 复用本机 `~/.claude` 的 OAuth 凭证
接入 API 正常多轮对话 → 核心工具(Bash/Read/Write/Edit/Grep/Glob)+ 基本命令(/help、/clear、
/config 等)可用。被桩的内部功能、voice/remote/server/bridge 等不在范围。

执行方案 = **运行路径驱动 + 局部类型校验**:沿 `cli.tsx → init → REPL → 认证 → QueryEngine → 工具`
深度优先修;每改一处只对改动文件做 typecheck;非核心/未触碰文件的类型错误不追。

**完成判定 / 停止条件**:见设计文档第 6 节。简言之——独立验证 sub-agent 逐条核验"依赖装好 /
能启动 / 已认证 / 对话打通 / 核心工具各跑通一次 / 基本命令不崩 / 改动文件 typecheck 通过",七项
全过即目标完成、停止;命中护栏(缺失模块需重实现、需超出"安装依赖/工具"范围的系统级操作、同一错误
修 ≥3 次仍不过、需联网拉缺失源码等)则暂停转人工。用 `/goal` 等自动推进时以此为终止信号,防止空转
或目标蔓延。

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
