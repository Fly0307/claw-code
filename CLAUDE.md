# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 仓库性质

这是 Claude Code（Anthropic 的 CLI）的**只读源码快照存档**，于 2026-03-31 通过 npm source map 泄露而公开。本仓库没有 `package.json`、构建系统、测试运行器或 lint 配置。代码为 TypeScript/TSX，面向 **Bun** 运行时，使用 **React + Ink** 构建终端 UI。

## 架构概览

### 入口与启动流程

`src/main.tsx` → Commander.js CLI 解析器 + React/Ink 渲染器。启动时在重型模块加载之前，以副作用方式并行触发预取（MDM 设置、Keychain 读取、GrowthBook）。主 REPL 屏幕为 `src/screens/REPL.tsx`（约 5K 行）。

### 核心引擎

- **`src/QueryEngine.ts`** — LLM 查询循环：流式响应、工具调用分发、thinking 模式、重试逻辑、token 计数
- **`src/query.ts`** — 高层查询编排：消息规范化、自动 compact 触发、上下文管理
- **`src/services/tools/toolOrchestration.ts`** — 将工具调用分为只读（并发）和变更（顺序）两类批次
- **`src/services/tools/toolExecution.ts`** — 单个工具执行，含权限检查和进度追踪

### 工具系统 (`src/tools/`)

每个工具是 `src/tools/<ToolName>/` 下的自包含模块，拥有自己的 prompt、权限逻辑和 UI 组件。工具通过 `src/tools.ts` 中的 `getAllBaseTools()` 注册，然后经过以下过滤：
1. Feature flags（`bun:bundle` 的 `feature()` 调用）
2. `USER_TYPE === 'ant'` 过滤仅内部使用的工具
3. 通过 `filterToolsByDenyRules()` 的权限拒绝规则
4. 模式过滤（简单模式仅有 Bash/Read/Edit；REPL 模式隐藏原语工具）

工具可用性常量（哪些工具允许 agent 使用）位于 `src/constants/tools.ts`。

### 命令系统 (`src/commands/`)

斜杠命令注册在 `src/commands.ts`。命令类型为 `Command`（定义于 `src/types/command.ts`），变体包括：`prompt`（向对话注入文本）、`local`（运行 JS）、`local-jsx`（渲染 React UI）。Feature-gated 命令使用条件 `require()`。

### 状态管理

- **`src/bootstrap/state.ts`** — 全局可变状态单例（会话 ID、费用计数器、已注册 hooks、频道列表），通过 `createSignal()` 发信号
- **`src/state/AppStateStore.ts`** — React 应用状态类型定义
- **`src/state/AppState.tsx`** — 包装 store 的 React context provider

### 权限系统

- **`src/hooks/toolPermission/`** — 三种处理策略：`interactiveHandler.ts`（用户交互提示）、`coordinatorHandler.ts`、`swarmWorkerHandler.ts`
- **`src/utils/permissions/`** — 规则解析、bash 命令分类（`bashClassifier.ts`、`yoloClassifier.ts`）、文件系统路径验证、auto-mode 状态
- **`src/tools/BashTool/bashSecurity.ts`** + `bashPermissions.ts` — Shell 命令安全分析和沙箱决策逻辑

### Bridge 系统 (`src/bridge/`)

IDE 扩展（VS Code、JetBrains）与 CLI 之间的双向 IPC。关键文件：
- `bridgeMain.ts` — Bridge 主循环
- `bridgeMessaging.ts` — 消息协议
- `replBridge.ts` — REPL 会话 bridge
- `jwtUtils.ts` — JWT 认证
- `sessionRunner.ts` — 会话执行

### MCP 集成 (`src/services/mcp/`)

Model Context Protocol：`MCPConnectionManager.tsx`（React context）、`client.ts`（服务器连接）、`config.ts`（服务器配置加载）、`auth.ts`（MCP 服务器 OAuth）。MCP 工具通过 `src/tools.ts` 中的 `assembleToolPool()` 与内置工具合并。

### Agent / Swarm 系统

- **`src/tools/AgentTool/`** — 子 agent 生成，内置 agent 类型在 `built-in/` 目录下（explore、plan、general-purpose、code-review、claude-code-guide、verification）
- **`src/utils/swarm/`** — Team/swarm 编排：`inProcessRunner.ts`（进程内 teammate 执行）、`backends/`（tmux、iTerm2、进程内面板管理）
- **`src/coordinator/coordinatorMode.ts`** — 多 agent 协调器模式
- **`src/tasks/`** — 任务类型：`LocalAgentTask`、`InProcessTeammateTask`、`RemoteAgentTask`、`LocalShellTask`、`DreamTask`

### 记忆系统 (`src/memdir/`)

持久化项目记忆：`MEMORY.md` 作为入口（最多 200 行 / 25KB），类型定义在 `memoryTypes.ts`。自动记忆提取通过 `src/services/extractMemories/` 实现。

### Compact / 上下文管理

`src/services/compact/` — 对话压缩：`compact.ts`（完整压缩）、`microCompact.ts`（微压缩）、`autoCompact.ts`（自动压缩触发）、`sessionMemoryCompact.ts`（会话记忆压缩）。

### Feature Flags

通过 `bun:bundle` 的 `feature()` 函数实现死代码消除。被 feature check 包裹的代码在构建时会被完全剥离。关键标志：

| 标志 | 用途 |
|---|---|
| `PROACTIVE` / `KAIROS` | 主动/睡眠模式、推送通知 |
| `BRIDGE_MODE` / `DAEMON` | IDE bridge、远程控制服务器 |
| `VOICE_MODE` | 语音输入 |
| `AGENT_TRIGGERS` | Cron/定时触发器 |
| `MONITOR_TOOL` | 后台进程监控 |
| `COORDINATOR_MODE` | 多 agent 协调器 |
| `WORKFLOW_SCRIPTS` | 工作流执行 |
| `CONTEXT_COLLAPSE` | 高级上下文压缩 |
| `TEAMMEM` | 团队记忆同步 |
| `BUDDY` | 伴游精灵 |

`USER_TYPE === 'ant'` 控制仅内部使用的工具（REPL、Tungsten、SuggestBackgroundPR）和命令。

### UI 层

- **`src/ink/`** — Fork/定制化的 Ink 终端渲染框架
- **`src/components/`**（约 144 个组件）— 终端 UI 的 React 组件
- **`src/components/design-system/`** — 共享 UI 原语（Dialog、FuzzyPicker、ProgressBar 等）
- **`src/components/permissions/`** — 各工具的权限请求 UI
- **`src/components/PromptInput/`** — 主输入组件，含模式切换、历史记录、语音

### 关键工具目录

- **`src/utils/bash/`** — Bash 命令解析、AST、shell 引号（支撑 shell 安全分析）
- **`src/utils/permissions/`** — 权限规则引擎、分类器、文件系统验证
- **`src/utils/plugins/`** — 插件生命周期：加载、市场、安装、MCP 集成
- **`src/utils/model/`** — 模型配置、provider 路由（直连 API、Bedrock、Vertex）、上下文窗口逻辑
- **`src/utils/settings/`** — 设置管理，含 MDM 支持、验证、变更检测
- **`src/utils/swarm/`** — Swarm/team 执行后端和权限桥接

### Skill 系统 (`src/skills/`)

打包的 skill 在 `src/skills/bundled/` 中（commit、review、debug、loop、simplify、schedule 等）。Skill 通过 `loadSkillsDir.ts` 从 `.claude/skills/` 目录和插件加载。

### 插件系统 (`src/plugins/`)

内置插件注册在 `builtinPlugins.ts`。插件加载由 `src/utils/plugins/pluginLoader.ts`（约 3.3K 行）编排，处理来自插件的命令、hooks、skills、agents 和 MCP 服务器。

### 服务集成

- **`src/services/api/`** — Anthropic API 客户端（`client.ts`）、流式传输（`claude.ts`）、bootstrap、文件 API、重试逻辑
- **`src/services/analytics/`** — GrowthBook feature flags + A/B 实验、OpenTelemetry/Datadog 遥测、1P 事件日志
- **`src/services/oauth/`** — Claude AI 订阅认证的 OAuth 2.0 流程
- **`src/services/lsp/`** — Language Server Protocol 客户端管理
- **`src/services/autoDream/`** — 主动模式的自动 dream 整合

## 关键设计模式

- **启动时并行预取**：`startMdmRawRead()` 和 `startKeychainPrefetch()` 在重型 import 之前触发
- **通过 `require()` 懒加载**：重型模块（OpenTelemetry、gRPC、feature-gated 子系统）通过 `require()` 延迟加载，以打破循环依赖和减少启动时间
- **工具并发分区**：`toolOrchestration.ts` 将工具调用分为只读批次（并行）和变更批次（顺序）
- **权限分类管线**：Bash 命令经过 AST 解析 → 命令分类 → yolo/classifier → 用户提示
- **Prompt cache 稳定性**：`assembleToolPool()` 中工具按名称排序，使内置工具形成连续前缀，以适配 Anthropic 服务端缓存断点
