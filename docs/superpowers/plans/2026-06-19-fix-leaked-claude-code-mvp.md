# 泄露 Claude Code 源码 MVP 修复实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把本仓库这份泄露的 Claude Code 源码修到本地能启动、能用复用的 `~/.claude` 凭证正常对话、核心工具与基本命令可用(MVP)。

**Architecture:** 运行路径驱动的迭代修复 —— 装好 Bun+依赖后,沿 `cli.tsx → init → REPL → 认证 → QueryEngine → 工具/命令` 真实运行路径深度优先排错,撞到因泄露而损坏处按"最小改动优先"修(核心路径认真修,非核心/缺失模块用最小桩或禁用)。**实现 agent 只改代码;每个任务的完成判定由独立验证 sub-agent 跑应用、回报真实证据。** 这不是单元测试 TDD:每步的"测试"是运行应用并观察行为。

**Tech Stack:** Bun(≥1.3,运行时)、TypeScript、React 19 + Ink(终端 UI)、Zod、Anthropic SDK、OAuth。

## Global Constraints

每个任务都隐含遵守以下全局约束(逐条摘自设计文档 `docs/superpowers/specs/2026-06-19-fix-leaked-claude-code-mvp-design.md`):

- **最小改动优先**:核心路径认真修;非核心/缺失模块处用最小桩或禁用跳过;不重实现缺失模块内部逻辑。
- **缺失模块策略**:已 stub 的保持 no-op;运行时新撞到的缺失内部模块,优先加最小桩(返回安全默认 / 抛"不支持")。
- **改动留痕**:每处因泄露而修的地方加注释 `// LEAK-FIX: <原因>`;按逻辑分组**小步 git 提交**。
- **切断对外上报/自更**:遥测(OpenTelemetry)、分析(GrowthBook/Datadog)、自动更新、远程托管设置/策略限制在本地构建中**失效且安全失效**(不回传、不自更、不致崩)。
- **不重构正常代码**:只修坏的。
- **被桩功能优雅降级**:提示"不支持"而非崩溃。
- **风格统一**:修复/新增代码贴合原 Claude Code 命名、注释密度、写法。
- **自主安装授权**:安装 Bun、`bun install`/`npm install --legacy-peer-deps`、修复所需开发工具/类型包,可自行执行无需询问;`git push`、删除非自建内容、向外部发数据仍需先确认。
- **提交信息**结尾附:`Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`。
- **typecheck 范围**:仅对改动到的文件做局部检查;非核心/未触碰文件的类型错误不追。

## 独立验证约定(贯穿全部任务)

- 每个任务末尾的"验证"步骤,**必须派一个独立 sub-agent** 执行(与做修复的 agent 分离)。
- 验证 sub-agent:**只运行与判定,不修改源码**;复用本机 `~/.claude` 配置/凭证;回报真实终端输出作为证据;给出该任务对应 DoD 判定项的 PASS/FAIL 结论。
- 任务判定以独立验证 sub-agent 的结论为准。FAIL 则回到实现继续修;命中第 6.2 节护栏(缺失模块需重实现、需超安装范围的系统级操作、同一错误修 ≥3 次仍不过、需联网拉缺失源码)则暂停转人工。

---

### Task 0: 环境就绪(安装 Bun + 依赖)

**对应 DoD:** #1 依赖可安装。

**Files:**
- 读:`package.json`、`bunfig.toml`、`bun.lock`、`README.md`(安装说明)
- 可能修:`package.json`(仅当依赖确实无法解析时,按最小改动调整;改动留痕)

**Interfaces:**
- Produces:可用的 `bun`(≥1.3)命令、已填充的 `node_modules`、可链接的 `stubs/` 桩包。后续所有任务依赖此环境。

- [ ] **Step 1: 确认 / 安装 Bun(已授权,无需询问)**

  先查:`bun --version`(PowerShell)。若缺失则安装(Windows):
  ```powershell
  powershell -c "irm bun.sh/install.ps1 | iex"
  ```
  安装后重开 shell 或刷新 PATH,再 `bun --version` 确认 ≥ 1.3。

- [ ] **Step 2: 安装依赖**

  优先 `bun install`;若因 React 19 canary / react-reconciler canary 的 peer deps 冲突失败,改用:
  ```bash
  npm install --legacy-peer-deps
  ```
  目标:命令成功结束、`node_modules` 生成、`stubs/@ant/*` 等本地桩包正确链接。

- [ ] **Step 3: 冒烟确认工具链**

  Run: `bun run typecheck` 仅用于观察基线错误规模(**不要求通过**,只记录数量级,便于后续判断哪些是核心路径错误)。

- [ ] **Step 4: 提交**

  ```bash
  git add -A
  git commit -m "chore: install toolchain and dependencies (Bun + node_modules)"
  ```
  注:`node_modules/` 已被 `.gitignore` 排除;本提交主要落地对 `package.json`/lock 的任何最小改动(若无改动则跳过提交)。

- [ ] **Step 5: 独立验证(sub-agent)**

  派独立 sub-agent 执行并回报:`bun --version`(≥1.3)、依赖安装命令的结尾输出(无致命错误)、`node_modules` 存在。PASS 条件 = DoD #1。

---

### Task 1: 启动到交互式 REPL

**对应 DoD:** #2 能启动。

**Files(运行路径锚点;具体改哪行由运行时报错确定):**
- `src/entrypoints/cli.tsx`(入口/参数分发)
- `src/entrypoints/init.ts`、`src/setup.ts`、`src/bootstrap/state.ts`(初始化序列)
- `src/replLauncher.tsx`、`src/screens/REPL.tsx`、`src/ink.ts` / `src/ink/`(REPL 与 Ink 渲染)

**Interfaces:**
- Consumes:Task 0 的 Bun + 依赖环境。
- Produces:`bun run start` 可进入交互式 REPL 的可运行入口。

- [ ] **Step 1: 建立失败观察**

  Run: `bun run start`(必要时在干净 cwd 下)。记录第一个崩溃/未捕获异常的完整堆栈与触发文件。

- [ ] **Step 2: 定位并按规则修复最靠前的阻断点**

  沿堆栈定位到具体文件。判定:
  - 若是核心启动路径(上面的锚点文件)→ 认真修(补缺失导入/类型/初始化)。
  - 若是缺失内部模块或非核心特性触发 → 用最小桩或禁用该路径(优雅降级)。
  每处修改加 `// LEAK-FIX: <原因>`。

- [ ] **Step 3: 对改动文件做局部 typecheck**

  Run: `bun x tsc --noEmit`(若耗时过长,改为只检查改动文件:`bun x tsc --noEmit <改动文件...>` 或临时缩小 `tsconfig` include)。确认本次改动文件无新增类型错误。

- [ ] **Step 4: 重跑,迭代**

  重复 Step 1–3,直到 `bun run start` 进入 REPL 不崩溃、界面渲染正常。同一错误连续修 ≥3 次仍不过 → 停,转人工(护栏)。

- [ ] **Step 5: 提交**

  ```bash
  git add -A
  git commit -m "fix(boot): start REPL without crashing"
  ```

- [ ] **Step 6: 独立验证(sub-agent)**

  派独立 sub-agent:`bun run start`,确认进入交互式 REPL、无崩溃/未捕获异常、界面渲染正常,回报终端输出(可截取首屏)。PASS 条件 = DoD #2。

---

### Task 2: 切断对外上报与自动更新(安全失效)

**对应约束:** 切断对外上报/自更;支撑 DoD #7 无回归。

**Files(锚点):**
- `src/utils/sinks.ts`(`initSinks`,遥测 sink 初始化)
- `src/services/analytics/{index.ts,growthbook.ts,datadog.ts,sink.ts,sinkKillswitch.ts}`(分析/GrowthBook/Datadog)
- `src/services/diagnosticTracking.ts`、`src/services/internalLogging.ts`
- `src/components/AutoUpdater*.tsx`、`src/cli/update.ts`、`src/commands/version.ts`(自动更新)
- `src/services/remoteManagedSettings/`、`src/services/policyLimits/`、`src/services/settingsSync/`(远程托管设置/策略限制)

**Interfaces:**
- Consumes:Task 1 可启动的 REPL。
- Produces:本地构建在启动与运行中**不向外回传、不自动更新**,且这些路径安全失效不致崩。

- [ ] **Step 1: 盘点对外通路的初始化点**

  Grep 上述锚点中实际发起网络/上报/更新检查的入口(如 `logEvent`、exporter flush、`checkForUpdate`、remote settings fetch)。列出在启动路径上会被触发的项。

- [ ] **Step 2: 按最小改动使其失效且安全失效**

  对每个触发点:让其成为 no-op / 提前 return / 走"禁用"分支(优先利用已有的 killswitch / 环境开关,如 `sinkKillswitch`、`metricsOptOut`、`DISABLE_*` env),而非删代码。确保失效路径不抛异常。每处加 `// LEAK-FIX: <原因>`。

- [ ] **Step 3: 局部 typecheck**

  Run: `bun x tsc --noEmit <改动文件...>`,确认无新增类型错误。

- [ ] **Step 4: 重跑确认不致崩**

  Run: `bun run start`,确认启动与基本交互不因这些改动崩溃。

- [ ] **Step 5: 提交**

  ```bash
  git add -A
  git commit -m "fix(privacy): disable telemetry, analytics, auto-update and remote-managed settings (safe no-op)"
  ```

- [ ] **Step 6: 独立验证(sub-agent)**

  派独立 sub-agent:启动应用并(在可观察范围内)确认无对外上报/自更动作被触发、无相关崩溃;回报证据。

---

### Task 3: 认证(复用本机 ~/.claude 凭证)

**对应 DoD:** #3 已认证。

**Files(锚点):**
- `src/services/oauth/{index.ts,client.ts,getOauthProfile.ts,types.ts}`
- `src/utils/auth.ts`、`src/utils/authFileDescriptor.ts`、`src/utils/authPortable.ts`(凭证读取/描述符)
- 启动时凭证预取相关(`src/setup.ts` 中 keychain/api-key 预取路径)

**Interfaces:**
- Consumes:Task 1 可启动 REPL。
- Produces:启动后被识别为已登录、可用现有 token 调 API 的认证态。

- [ ] **Step 1: 定位本机凭证来源**

  在 PowerShell 下确认本机 Claude Code 凭证的实际存储:查看 `~/.claude`(`C:\Users\logic\.claude`)下是否有 `.credentials.json` 或等价文件,或是否走 Windows 凭据库。记录读取方式。

- [ ] **Step 2: 建立失败观察**

  Run: `bun run start`,观察认证态:是否提示未登录 / 读取 token 失败 / keychain 在 Windows 不可用导致报错。记录具体报错与文件。

- [ ] **Step 3: 按规则修复凭证读取路径**

  让凭证读取在 Windows 下命中"读 `~/.claude` 凭证文件"的正确分支(macOS keychain 路径若不适用则安全回退到文件读取)。最小改动,加 `// LEAK-FIX:`。
  - 若复用本机凭证不可行 → 回退:修通 `/login` OAuth 流程(`auth-code-listener.ts` 本地回调),此为该任务的备选路径。

- [ ] **Step 4: 局部 typecheck + 重跑**

  Run: `bun x tsc --noEmit <改动文件...>`;再 `bun run start` 确认被识别为已登录、无认证报错。

- [ ] **Step 5: 提交**

  ```bash
  git add -A
  git commit -m "fix(auth): reuse local ~/.claude credentials on Windows"
  ```

- [ ] **Step 6: 独立验证(sub-agent)**

  派独立 sub-agent:用本机 `~/.claude` 启动,确认无需手动重新登录即为已认证态(或 `/login` 能走通),回报证据。PASS 条件 = DoD #3。

---

### Task 4: 打通端到端对话

**对应 DoD:** #4 对话打通。

**Files(锚点):**
- `src/QueryEngine.ts`、`src/query.ts`、`src/query/`(查询管线/流式)
- `src/services/api/{client.ts,claude.ts,bootstrap.ts,withRetry.ts,errors.ts}`(API 客户端)

**Interfaces:**
- Consumes:Task 3 的认证态。
- Produces:在 REPL 发一条消息能收到完整流式回复。

- [ ] **Step 1: 建立失败观察**

  在 REPL 输入一条简单消息(如 "ping, reply with 'pong'")。记录:是否成功发起请求、流式是否中断、有无报错(401/模型 id/SSE 解析/缺失字段等)及其文件。

- [ ] **Step 2: 按规则修复对话路径**

  沿 QueryEngine → api/client 修复阻断点(认证头、模型 id、流式解析、retry/错误处理中因泄露损坏处)。最小改动,加 `// LEAK-FIX:`。

- [ ] **Step 3: 局部 typecheck + 重跑对话**

  Run: `bun x tsc --noEmit <改动文件...>`;再在 REPL 实测一条消息,确认收到完整流式回复。

- [ ] **Step 4: 提交**

  ```bash
  git add -A
  git commit -m "fix(query): complete one end-to-end streaming chat round-trip"
  ```

- [ ] **Step 5: 独立验证(sub-agent)**

  派独立 sub-agent:启动 → 发一条消息 → 确认收到 Claude 完整流式回复;回报对话证据。PASS 条件 = DoD #4。

---

### Task 5: 核心工具可用

**对应 DoD:** #5 核心工具。

**Files(锚点):**
- `src/tools.ts`(工具注册表)
- `src/tools/`(各工具实现:Bash、FileRead、FileWrite、FileEdit、Grep、Glob 对应子目录/文件)

**Interfaces:**
- Consumes:Task 4 可对话能力(模型需能实际触发工具调用)。
- Produces:Bash/Read/Write/Edit/Grep/Glob 六个核心工具各能成功执行一次。

- [ ] **Step 1: 用一段脚本化对话逐个触发工具**

  在 REPL 引导模型依次实际调用六个工具(例:让它用 Write 建一个临时文件、用 Read 读它、用 Edit 改它、用 Grep/Glob 搜它、用 Bash 执行 `echo`)。记录每个工具的成功/报错与文件。

- [ ] **Step 2: 按规则逐个修复失败工具**

  对失败工具沿 `src/tools/<工具>` 修复(权限校验、参数 schema、执行逻辑中因泄露损坏处)。最小改动,加 `// LEAK-FIX:`。被桩的非核心工具不在此列。

- [ ] **Step 3: 局部 typecheck + 重测**

  Run: `bun x tsc --noEmit <改动文件...>`;重跑触发对话,确认六个工具各成功返回预期结果。

- [ ] **Step 4: 提交**

  ```bash
  git add -A
  git commit -m "fix(tools): make Bash/Read/Write/Edit/Grep/Glob work"
  ```

- [ ] **Step 5: 独立验证(sub-agent)**

  派独立 sub-agent:启动 → 通过对话使六个核心工具各被调用一次 → 确认均成功、无崩溃;回报每个工具的输出证据。PASS 条件 = DoD #5。

---

### Task 6: 基本斜杠命令可用

**对应 DoD:** #6 基本命令。

**Files(锚点):**
- `src/commands.ts`(命令注册表)
- `src/commands/`(`/help`、`/clear`、`/config` 对应实现;如 `commands/config` 等)

**Interfaces:**
- Consumes:Task 1 的 REPL。
- Produces:`/help`、`/clear`、`/config` 能执行不崩溃。

- [ ] **Step 1: 建立失败观察**

  在 REPL 依次执行 `/help`、`/clear`、`/config`。记录崩溃/报错与文件。

- [ ] **Step 2: 按规则修复**

  沿对应命令实现修复阻断点;最小改动,加 `// LEAK-FIX:`。

- [ ] **Step 3: 局部 typecheck + 重测**

  Run: `bun x tsc --noEmit <改动文件...>`;重跑三个命令确认不崩溃、行为正常。

- [ ] **Step 4: 提交**

  ```bash
  git add -A
  git commit -m "fix(commands): make /help, /clear, /config work"
  ```

- [ ] **Step 5: 独立验证(sub-agent)**

  派独立 sub-agent:启动 → 执行三个命令 → 确认均能执行不崩溃;回报证据。PASS 条件 = DoD #6。

---

### Task 7: 终判与无回归(停止门)

**对应 DoD:** #7 无明显回归 + 全量 DoD 终判 → 目标完成停止。

**Files:**
- 读:本计划全部改动涉及的文件;`docs/superpowers/specs/2026-06-19-fix-leaked-claude-code-mvp-design.md` 第 6 节。

**Interfaces:**
- Consumes:Task 0–6 的全部成果。
- Produces:一份覆盖 DoD 全 7 项的独立验证报告;满足则目标完成、停止。

- [ ] **Step 1: 改动文件全量局部 typecheck**

  收集本次修复改动过的所有文件,Run: `bun x tsc --noEmit <全部改动文件...>`,确认无新增类型错误。

- [ ] **Step 2: 确认 phone-home/自更未致崩**

  Run: `bun run start` 一轮基本使用,确认 Task 2 的切断改动未引入崩溃。

- [ ] **Step 3: 提交(如有收尾改动)**

  ```bash
  git add -A
  git commit -m "chore: finalize MVP repair; no-regression typecheck on touched files"
  ```

- [ ] **Step 4: 独立验证(sub-agent)逐条核验 DoD 全 7 项**

  派独立 sub-agent 从干净状态走一遍,逐条核验设计文档 6.1 表的 #1–#7,每项回报 PASS/FAIL + 证据:
  1. `bun install` 成功
  2. `bun run start` 进入 REPL 不崩溃
  3. 复用 `~/.claude` 凭证为已登录态
  4. 一次端到端流式回复成功
  5. 六个核心工具各跑通一次
  6. `/help` `/clear` `/config` 不崩溃
  7. 改动文件 typecheck 通过、无新增崩溃

- [ ] **Step 5: 终判**

  七项全 PASS → **目标完成,停止并结束**,产出验证报告摘要。任一 FAIL → 回到对应 Task;命中 6.2 护栏 → 暂停转人工。

---

## 备注

- 推送策略:用户要求**暂不推送**,等修复有进展再统一推到 `origin`(`git push` 仍需先确认)。
- 本计划与设计文档(`docs/superpowers/specs/2026-06-19-fix-leaked-claude-code-mvp-design.md`)配套;约束以设计文档为准,本计划细化为可执行步骤。
