# 设计:把泄露的 Claude Code 源码修到本地可用(MVP)

- 日期:2026-06-19
- 状态:已与用户确认设计,待复核;尚未进入实现
- 仓库:`E:/projects/myclaude`(`origin` = https://github.com/lgk-code/myclaude）

---

## 0. 背景与现状诊断

本仓库是 Claude Code CLI 的源码归档(README 自述为 2026-03-31 经 npm source map 泄露;用户说明系 fork 自他人公开仓库,用于个人 git 管理)。目标是把这份源码修到**本地能正常运行、正常使用**。

动手前已做的诊断结论:

| 维度 | 现状 |
|---|---|
| 运行时 | ❌ Bun **未安装**(本机仅 Node v24 + npm)。项目为 **Bun 专用**:入口靠 `bun run`,20 个文件使用真实 `Bun.*` API(`Bun.hash/semver/which/spawn/listen/stringWidth/YAML/JSONL` 等),Node 无法运行。**安装 Bun 是硬前提**。 |
| 依赖 | ❌ 未安装(`node_modules` 不存在),需 `bun install` 或 `npm install --legacy-peer-deps`。 |
| feature flag | ✅ 已被 `plugins/bunBundleDev.ts` shim(196 处 `bun:bundle` 导入),默认全 `false`,可用 `FEATURE_FLAGS` 开启。无需修改。 |
| 缺失模块 | ⚠️ 6 个内部包是 no-op 桩:`@ant/computer-use-input`、`@ant/computer-use-mcp`、`@ant/computer-use-swift`、`@ant/claude-for-chrome-mcp`、`@anthropic-ai/mcpb`、`@anthropic-ai/sandbox-runtime`。依赖它们的功能会"假装成功"但不工作。 |
| 真实运行错误 | ❓ 暂不可知。必须先装好 Bun + 依赖、真正运行,才能暴露泄露代码中"引用未包含模块 / 类型不匹配"的实际报错。 |

**结论**:这不是"读代码列清单"式修复,而是"**装环境 → 跑起来 → 看报错 → 逐个修**"的迭代过程。方案第一步必然是搭建可运行环境。

---

## 1. 目标(Scope)

档位:**核心可用(MVP)**。

**成功标准(全部满足即达标):**
1. `bun run start` 能启动交互式 REPL,不崩溃,能进入界面。
2. 复用本机 `~/.claude` 的 OAuth 凭证接入 Anthropic API,能正常多轮对话(流式回复正常)。
3. 核心工具可用:`Bash` / `Read` / `Write` / `Edit` / `Grep` / `Glob`。
4. 基本斜杠命令可用:如 `/help`、`/clear`、`/config` 等。

**非目标(本期不做):**
- 被桩的内部功能:computer-use(电脑操作)、Chrome MCP、沙箱(sandbox-runtime)、mcpb(桌面扩展打包)——不要求工作。
- 非核心特性:voice(语音)、remote(远程会话)、server(服务端模式)、bridge(IDE 集成)、coordinator(多 agent 编排)等——不在 MVP 范围。
- 不追求全量 `tsc --noEmit` 通过。

---

## 2. 执行方案(C — 混合:运行路径驱动 + 局部类型校验)

以"运行起来"作为驱动信号,沿真实运行路径深度优先修复;每改一个模块就对**改动到的文件**做局部 typecheck 作为安全网;**不追**未触碰 / 非核心文件的类型错误。

**阶段划分(后续实现计划在此基础上细化):**

- **阶段 0 — 环境就绪**
  - 安装 Bun(≥ 1.3)。
  - 安装依赖(`bun install`,必要时 `npm install --legacy-peer-deps`)。
  - 确认 `stubs/` 下的 stub 包正确链接。
- **阶段 1 — 能启动**
  - 沿 `src/entrypoints/cli.tsx → init → REPL → Ink 渲染` 修到能进入界面。
- **阶段 2 — 接通认证**
  - 让凭证读取路径在 Windows 下读到本机 `~/.claude` 的现有 OAuth token。
- **阶段 3 — 打通对话**
  - `QueryEngine` → Anthropic API 跑通一个流式来回,回复正常。
- **阶段 4 — 核心工具与基本命令**
  - 逐个验证核心工具(Bash/Read/Write/Edit/Grep/Glob)与基本命令。
- **每阶段**:仅对改动到的文件做 typecheck;非核心文件类型错误不追;阶段结束由**独立 sub-agent** 验证并给出真实运行证据(详见第 5 节)。

**未采用的方案及原因:**
- 方案 A(类型检查驱动,先扫平全量 `tsc` 错误):513k 行、错误大概率成千上万且多在 MVP 用不到的路径,违背"最小改动 / 不重构正常代码"。
- 方案 B(纯运行路径驱动,崩了再修):聚焦但缺类型安全网,易漏隐患。C 在 B 基础上加局部 typecheck,取两者之长。

---

## 3. 修复规则(Rules)

- **最小改动优先**:核心路径(启动 / 对话 / 核心工具)认真修;非核心、依赖缺失模块处用最小桩或禁用跳过。
- **缺失模块策略**:已 stub 的保持 no-op;运行时新撞到的缺失内部模块,优先加**最小桩**(返回安全默认值 / 抛出"不支持"),而非重新实现其内部逻辑。

---

## 4. 约束(Constraints)

1. **改动留痕 + 小步提交**:每处因泄露而修改的地方加统一标记注释 `// LEAK-FIX: <原因>`;按逻辑分组小步 git 提交,便于区分原始代码与修复、随时回溯。
2. **切断对外上报 / 自动更新**:遥测(OpenTelemetry)、分析(GrowthBook)、自动更新、远程托管设置 / 策略限制在本地构建中**失效**(不回传、不自更),且必须**安全失效**(失效不致崩溃)。
3. **不重构正常代码**:只修坏的,不顺手重构 / 优化能正常工作的代码,避免引入新问题。
4. **被桩功能优雅降级**:被桩的内部功能保持"提示不支持"而非直接崩溃。
5. **风格统一**:修复 / 新增代码贴合原 Claude Code 的命名、注释密度、写法与惯用法。

---

## 5. 验证(Verification)

- **独立验证(关键约束)**:每阶段的验证由**独立的 sub-agent** 执行,与负责修复的 agent 分离。修复 agent 只负责实现;验证 agent 独立运行应用、回报真实证据,避免"自己改自己验"导致的盲点。验证 agent 不得修改源码,只做运行与判定。
- **主验证**:验证 agent 复用本机 `~/.claude` 配置 / 凭证,`bun run start` 实测多轮对话,并逐个跑通每个核心工具,回报真实终端输出。
- **安全网**:对改动文件做局部 typecheck(可由修复 agent 自查,但阶段判定以独立验证 agent 的结论为准)。
- **证据原则**:每阶段结束由验证 agent 给出真实命令输出作为证据,不空口声称"修好了"(遵循 verification-before-completion)。

---

## 6. 完成判定与停止条件(Definition of Done & Stopping Conditions)

本节供自动推进式开发(如 Claude Code 的 `/goal` 目标功能)判断"何时算完成并停止"。
判定由**独立验证 sub-agent** 执行并出具证据;所有条件都写成可客观核验的形式。

### 6.1 完成判定(以下全部满足 → 目标完成,停止)

| # | 判定项 | 如何验证 | 通过标准 |
|---|---|---|---|
| 1 | 依赖可安装 | `bun install`(或 `npm install --legacy-peer-deps`) | 成功结束,无致命错误 |
| 2 | 能启动 | `bun run start` | 进入交互式 REPL,无崩溃/未捕获异常,界面渲染正常 |
| 3 | 已认证 | 复用本机 `~/.claude` 凭证启动 | 被识别为已登录、无需手动重新认证(或 `/login` 能走通) |
| 4 | 对话打通 | 在 REPL 发送一条消息 | 收到 Claude 完整流式回复(端到端一次成功来回) |
| 5 | 核心工具 | 引导模型实际调用 Bash/Read/Write/Edit/Grep/Glob 各一次 | 每个工具成功执行并返回预期结果,无崩溃 |
| 6 | 基本命令 | 执行 `/help`、`/clear`、`/config` | 均能执行,不崩溃 |
| 7 | 无明显回归 | 对改动文件做局部 typecheck;确认切断上报/自更未致崩 | 改动文件 typecheck 通过;无新增崩溃 |

七项全部 ✅ → 验证 sub-agent 出具通过报告 → **目标完成,停止并结束。**

### 6.2 失败 / 升级停止(护栏:命中任一条即暂停自动推进,转人工确认)

- 缺失内部模块无法用最小桩绕过,核心路径必须重实现(超出"最小改动")。
- 需要系统级操作(如安装 Bun)而尚未获授权。
- 凭证复用不可行,且修复 OAuth 链路超出 MVP 范围。
- 同一处错误连续修复 ≥ 3 次仍不通过(防止打转空转)。
- 需要联网拉取本仓库未包含的真实模块源码。

命中即:停止自动推进,记录现状与卡点,交用户决策。

### 6.3 范围护栏(防止目标蔓延导致永不停止)

- 不为"非目标"特性(computer-use / Chrome / 沙箱 / voice / remote / server / bridge 等)做任何修复以追求其工作。
- 不为追求全量 `tsc` 通过,去修改非核心、未被运行路径触碰的文件。

---

## 7. 风险与开放项

- **凭证读取**:Windows 下凭证存储位置 / 读取方式需实测(系统凭据库还是 credentials 文件);若复用本机凭证不可行,**回退**到走 `/login` OAuth 流程(需额外修复链路)。
- **依赖安装**:React 19 canary + react-reconciler canary 版本可能与锁文件 / peer deps 不一致,install 可能需 `--legacy-peer-deps`。
- **工作量不确定**:实际运行错误规模未知,阶段 1–3 的工作量存在不确定性。
- **法律/IP 提示**:内容自述为 Anthropic 专有源码;本修复仅用于用户本地个人使用,不涉及再分发。

---

## 8. 后续步骤

1. 用户复核本文档。
2. 复核通过后,进入 writing-plans,生成分阶段、可执行的实现计划(每阶段附 6.1 中对应的判定项与验证方式)。
3. 用户用 `/goal` 等方式实际启动修复;以第 6 节的完成判定/停止条件作为终止信号。
