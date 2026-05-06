# Claude Code 垂直能力解构报告

> 项目: Anthropic Claude Code (源码级分析)
> 分析源: 泄露 TypeScript 源码 (直读 src/ 目录)
> 分析日期: 2026-05-06
> 分析方法: 特征逆向挖掘法 (Feature Reverse Mining)
> 数据来源权威性层级: 泄露源码直读 > 推断 > 社区二手分析
> 特殊性: 基于 npm 包 source map 泄露的 TypeScript 源码，非官方开源；源码语言 TypeScript (1884 文件)
> 仓库: /Users/kongpei/workspace/project/harness/claude-code/src/

---

## 产品画像

| 属性 | 值 | 来源 |
|------|-----|------|
| 产品定位 | "本地代码 Agent 平台" — 面向代码工作流的本地 agent 平台，非命令行聊天工具 | 源码体量 + 架构复杂度 |
| 开发者 | Anthropic | 官方产品 |
| 分析源 | 泄露 TypeScript 源码 (1884 文件) | 仓库统计 |
| 源码语言 | TypeScript (.ts/.tsx) | src/ 目录 |
| 许可证 | 非开源 (source map 泄露逆向) | 事实陈述 |
| 核心入口 | `entrypoints/cli.tsx` → `init.ts` → `launchRepl`/`main.tsx` | `entrypoints/cli.tsx:33` |
| 执行内核 | `query.ts` (async generator loop) + `QueryEngine.ts` (class wrapper) | `query.ts:219`, `QueryEngine.ts:184` |
| CLI 架构 | React/Ink 终端 UI (JSX 组件, React Compiler) | `components/` 目录, 所有 .tsx 含 `react/compiler-runtime` |
| Tool 系统 | `Tool.ts` (type system) → `tools.ts` (registry) → `toolOrchestration.ts` (execution) | `Tool.ts:1-792`, `tools.ts:1-389`, `services/tools/toolOrchestration.ts` |
| Streaming Tool | `StreamingToolExecutor` — 工具执行实时流式输出 | `services/tools/StreamingToolExecutor.ts`, `query.ts:96` |
| 权限系统 | `types/permissions.ts` (PermissionMode/PermissionResult) + `utils/permissions/` | `Tool.ts:43-47`, `utils/permissions/` |
| 配置系统 | `init.ts` / `setup.ts` → 初始化引导, `utils/config.ts` | `entrypoints/cli.tsx:33` |
| 上下文管理 | `services/compact/compact.ts` (primary) + `apiMicrocompact.ts` + `reactiveCompact` + `snipCompact` + `contextCollapse` | `services/compact/` (11 files) |
| Memory | 5 层: User/Project/Local/Managed/AutoMem + TeamMem (feature gate) | `utils/memory/types.ts:3-10`, `memdir/memdir.ts:34-38` |
| 扩展机制 | MCP / Plugin / Skills / Remote / Bridge / Swarm / Tasks / Hooks | `tools/MCPTool/`, `utils/mcp/`, `utils/plugins/`, `remote/`, `utils/swarm/` |
| Sandbox | `@anthropic-ai/sandbox-runtime` 封装, SandboxManager + SandboxRuntimeConfig + SandboxViolationStore + SandboxDoctorSection + SandboxPermissionRequest | `utils/sandbox/sandbox-adapter.ts:7-22` |
| 分析系统 | GrowthBook feature flags (100+ gates) + Statsig + Datadog | `services/analytics/` (8 files) |
| Multi-Agent | 6 个 Built-in Agents + Fork Subagent + Coordinator→Workers + Swarm/Teammate | `tools/AgentTool/builtInAgents.ts` |

> 推断: Claude Code 的定位是 Anthropic 对 Cursor/Copilot 生态的降维打击 — 不做 IDE 插件，而是做一个本地 agent 平台，通过 CLI + REPL + SDK + MCP + Remote 多入口覆盖从个人开发者到团队的代码工作流。其架构复杂度 (1884 文件) 远超一般 CLI 工具，反映了一个完整的 agent 操作系统级设计。

---

## 一、架构图提取

### 1.1 六层系统架构

Source: 源码直读 `entrypoints/cli.tsx` → `init.ts` → `QueryEngine.ts`/`query.ts` → `Tool.ts`/`tools.ts` → `utils/`

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Claude Code 六层架构                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Layer 1: Entrypoints                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  CLI (entrypoints/cli.tsx) │ SDK (entrypoints/agentSdkTypes.js)     │ │
│  │  print.ts (headless) │ Remote (remote/SessionsWebSocket.ts)        │ │
│  │  MCP Server (utils/claudeInChrome/mcpServer.js)                   │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                            │
│  Layer 2: Bootstrap / Init                                              │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  entrypoints/cli.tsx:33 main() → dynamic import init → config     │ │
│  │  bootstrap/state.js: sessionId, cwd, model, feature flags, usage   │ │
│  │  utils/config.js → settings resolution (user/project/local/managed)│ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                            │
│  Layer 3: TUI / REPL (React/Ink)                                       │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Ink.js components: Box, Text, Select, Spinner                      │ │
│  │  commands/ directory: 30+ slash commands (session, agent, model…)  │ │
│  │  components/permissions/: PermissionDialog, SandboxPermissionRequest│ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                            │
│  Layer 4: Execution Kernel                                              │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  QueryEngine.ts (class, stateful) / query.ts (async generator)      │ │
│  │    while loop: context assembly → LLM request → stream → tool exec │ │
│  │    transitions: Continue (more tools) → Terminal (finish)           │ │
│  │    state: messages, toolUseContext, autoCompactTracking, turnCount  │ │
│  │    compact: reactiveCompact / autoCompact / snipCompact / collapse  │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                            │
│  Layer 5: Tool / Permission / Memory / Sandbox                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  tools.ts: 40+ tools registered (Bash, Read, Write, Edit, Grep,    │ │
│  │    Glob, Agent, Skill, MCP, WebFetch, WebSearch, Task*, TodoWrite, │ │
│  │    PlanMode, Worktree, LSP, AskUser, Config, REPL, Cron, …)        │ │
│  │  Tool.ts: type system (ToolDef, ToolUseContext, PermissionResult)   │ │
│  │  toolOrchestration.ts + StreamingToolExecutor.ts                    │ │
│  │  sandbox-adapter.ts: SandboxManager + SandboxRuntimeConfig (985 LoC)│ │
│  │  memory/: types.ts (5 types + TeamMem gate) + memdir/memdir.ts     │ │
│  │  sessionStorage.ts: 5105 LoC transcript logging + JSONL persistence │ │
│  └────────────────────────────┬───────────────────────────────────────┘ │
│                               │                                            │
│  Layer 6: Extensions / Remote / Swarm                                    │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  MCP: tools/MCPTool/MCPTool.ts + utils/mcpWebSocketTransport.ts    │ │
│  │       + services/mcp/MCPConnectionManager.tsx                       │ │
│  │  Plugin: utils/plugins/pluginLoader.js + commands/plugin/           │ │
│  │  Skills: skills/loadSkillsDir.js + utils/skills/skillChangeDetector │ │
│  │  Remote: remote/RemoteSessionManager.ts + remotePermissionBridge.ts │ │
│  │  Swarm: utils/swarm/ (constants, inProcessRunner, spawn*, backends)│ │
│  │  Bridge: utils/teammateMailbox.ts + teammate.ts (292 LoC)           │ │
│  │  Hooks: utils/hooks.js + hooks/postSamplingHooks.js                │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Sandbox 四层结构

Source: `utils/sandbox/sandbox-adapter.ts:985` 完整实现

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sandbox 四层结构 (sandbox-adapter.ts)          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: 决策层 (Should Use?)                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ SandboxManager.isSupportedPlatform()                       │ │
│  │ SandboxManager.isSandboxEnabledInSettings()                │ │
│  │ shouldAllowManagedSandboxDomainsOnly()                     │ │
│  │ shouldAllowManagedReadPathsOnly()                          │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                   │
│  Layer 2: 配置层 (convertToSandboxRuntimeConfig)                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ SettingsJson → SandboxRuntimeConfig                         │ │
│  │   network: allowedDomains / deniedDomains                   │ │
│  │   filesystem: FsReadRestrictionConfig / FsWriteRestriction  │ │
│  │   resolvePathPatternForSandbox(): // + / + ~/ prefix resolve│ │
│  │   resolveSandboxFilesystemPath(): absolute path expansion   │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                   │
│  Layer 3: 权限层 (Bash Permissions)                               │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ PermissionRuleValue: toolName + ruleContent matching        │ │
│  │ Bash tool → sandbox-aware command filtering                 │ │
│  │ WebFetch → domain allow/deny via network config             │ │
│  │ SandboxPermissionRequest.tsx: host-level user approval UI   │ │
│  └────────────────────────────┬───────────────────────────────┘ │
│                               │                                   │
│  Layer 4: 执行+清理层                                             │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ @anthropic-ai/sandbox-runtime: actual sandbox process       │ │
│  │ SandboxViolationStore: violation tracking + logging         │ │
│  │ SandboxDoctorSection: dependency check UI (/sandbox cmd)    │ │
│  │ cleanup: rmSync shell transient dirs                        │ │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Multi-Agent 三种模式

Source: `tools/AgentTool/builtInAgents.ts`, `utils/teammate.ts`, `utils/swarm/`, `coordinator/coordinatorMode.ts`

```
┌───────────────────────────────────────────────────────────────────────┐
│                  Multi-Agent 三种协作模式 (并存)                        │
├───────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Mode 1: Fork Subagent (父子上下文继承)                                │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ forkSubagent.ts:210                                               │ │
│  │   Parent → AgentTool without subagent_type → FORK_AGENT           │ │
│  │   child inherits FULL parent conversation + system prompt         │ │
│  │   byte-identical prefix → prompt cache sharing across siblings    │ │
│  │   FORK_BOILERPLATE_TAG guard prevents recursive forking           │ │
│  │   maxTurns: 200, model: 'inherit', permissionMode: 'bubble'      │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  Mode 2: Coordinator → Workers (协调者-工作者)                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ coordinatorMode.ts:369 - feature('COORDINATOR_MODE') gate         │ │
│  │   workerAgent.ts (feature-gated, not in this build)               │ │
│  │   isCoordinatorMode() checks CLAUDE_CODE_COORDINATOR_MODE env     │ │
│  │   workers get ASYNC_AGENT_ALLOWED_TOOLS subset (constants/tools.ts)│ │
│  │   INTERNAL_WORKER_TOOLS: team create/delete/sendMsg/synthetic     │ │
│  │   matchSessionMode(): auto-restore coordinator state on resume    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  Mode 3: Swarm / Teammate (对等群体, 3 backends)                       │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ utils/swarm/constants.ts: tmux-session based team management      │ │
│  │ utils/teammate.ts:292 - identity resolution (2 paths)             │ │
│  │   Path A: AsyncLocalStorage (in-process teammates)                │ │
│  │   Path B: dynamicTeamContext (tmux teammates via CLI args)         │ │
│  │ Backends (utils/swarm/backends/):                                 │ │
│  │   1. inProcessRunner.ts - same process, isolated context           │ │
│  │   2. Tmux panes - separate tmux panes per teammate                 │ │
│  │   3. External process - spawn via execPath or TEAMMATE_COMMAND     │ │
│  │ Mailbox: utils/teammateMailbox.ts for inter-agent messaging       │ │
│  │ Spawn: tools/shared/spawnMultiAgent.ts (1093 LoC)                 │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 二、原子化特征提取表

### 维度一: 通信与适配 (D1)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 源码出处 |
|----------|-------------|---------|------------|---------|
| 多入口架构 | CLI/print/SDK/MCP-Server/Remote/Chrome-Native 六种入口 | 覆盖交互式/脚本化/IDE/远程/MCP 全场景 | 非单一 CLI，多形态 agent 平台 | `entrypoints/cli.tsx:33-302` |
| Ink/React TUI | `entrypoints/cli.tsx` → React/Ink JSX 组件 + React Compiler (`react/compiler-runtime`) | JSX 组件化终端，30+ slash commands | 与 hermes-agent 同源技术栈，但已启用 React Compiler | `entrypoints/cli.tsx`, `commands/` (30+ .tsx files) |
| Slash Command 系统 | `commands/` 目录 30+ 命令: model, session, plugin, skills, agents, review, chrome, mcp… | 用户通过 `/` 控制 agent 行为全生命周期 | 命令即 JSX 组件 (React/Ink rendering) | `commands/plugin/plugin.tsx`, `commands/agents/agents.tsx` |
| MCP WebSocket Transport | `utils/mcpWebSocketTransport.ts:200` — Bun/Node dual runtime | 通过 WebSocket 双向 MCP 通信 | 双运行时适配 (Bun native vs ws package) | `utils/mcpWebSocketTransport.ts:22-70` |
| Remote Sessions | `remote/SessionsWebSocket.ts` + `remote/RemoteSessionManager.ts` | 远程 session 管理，permission bridge | 推断: 支持分布式/远程 agent 部署 | `remote/RemoteSessionManager.ts` |
| SDK 入口 | `entrypoints/agentSdkTypes.js` — typed SDK message protocol | 编程式调用，支持 SDK status/compat/encoding | 独立的 SDK 消息类型系统 | `QueryEngine.ts:9-16` (SDK message type imports) |

### 维度二: 执行深度 (D2)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 源码出处 |
|----------|-------------|---------|------------|---------|
| 40+ Built-in Tools | `tools.ts:1-389` — Bash, Read, Write, Edit, Grep, Glob, Agent, Skill, MCP, WebFetch, WebSearch, Task*, TodoWrite, PlanMode, Worktree, LSP, Config, REPL, Cron… | 完整代码工作流工具链 | 工具数量远超一般 CLI agent | `tools.ts:3-97` (tool imports), `constants/tools.ts:1-112` |
| streaming query loop | `query.ts` async generator yielding StreamEvent/RequestStartEvent/Message | LLM stream → yield → tool execution → yield → continue | async generator 模式支持优雅中断和恢复 | `query.ts:219-251` |
| StreamingToolExecutor | `services/tools/StreamingToolExecutor.ts` | 工具执行实时流式输出，不等工具完成即可推理下一步 | 在已知分析产品中未见同类机制 | `query.ts:96`, `services/tools/StreamingToolExecutor.ts` |
| 并行工具调用 | `services/tools/toolOrchestration.ts` — runTools() | 多工具并行执行再聚合结果 | `query.ts:98` 调用 runTools | `services/tools/toolOrchestration.ts` |
| QueryEngine 类封装 | `QueryEngine.ts:1295` — stateful conversation manager | submitMessage() per turn, 持久化 messages/fileState/usage | class-based state management across turns | `QueryEngine.ts:184-200` |
| 代码编辑 | FileEditTool + FileWriteTool + NotebookEditTool + Patch 策略 | 手术级代码修改 (surgical edits) | FileEditTool 支持字符串级精确替换 | `tools/FileEditTool/constants.js`, `tools/FileWriteTool/prompt.js` |
| Task system | TaskCreate/Get/Update/List/Output/Stop + TaskStateBase | 结构化任务生命周期管理 | 完整的 task 状态机 | `tools/TaskCreateTool/`, `Task.js` |
| LSP integration | `tools/LSPTool/LSPTool.js` | 语言服务器协议集成，智能代码分析与补全 | LSP 原生工具化 | `tools/LSPTool/LSPTool.js` |

### 维度三: 任务编排 (D3)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 源码出处 |
|----------|-------------|---------|------------|---------|
| 6 个 Built-in Agents | GENERAL_PURPOSE, PLAN, EXPLORE, VERIFICATION, CLAUDE_CODE_GUIDE, STATUSLINE_SETUP | 各有独立 system prompt, tool set, model 配置 | 内置 agent 角色专业化 | `tools/AgentTool/builtInAgents.ts:22-72` |
| GENERAL_PURPOSE_AGENT | tools: ['*'], shared prefix + guidelines | 默认全能力子 agent，用于复杂搜索/多步任务 | explicit guidelines: "NEVER create docs", "prefer editing" | `tools/AgentTool/built-in/generalPurposeAgent.ts:25-34` |
| PLAN_AGENT | 只读, 禁止文件写/编辑/Agent, omitClaudeMd: true, model: 'inherit' | 软件架构师，输出分步实现计划 + 关键文件列表 | "READ-ONLY MODE" 严格声明，disallowedTools explicit | `tools/AgentTool/built-in/planAgent.ts:73-92` |
| EXPLORE_AGENT | 只读, ant→inherit, external→haiku, omitClaudeMd: true | 快速代码搜索 agent, 三种 thoroughness level | 外部用 haiku 降本, 内部用同模型 | `tools/AgentTool/built-in/exploreAgent.ts:64-83` |
| VERIFICATION_AGENT | 只读 + 临时写 /tmp, 试图破坏而非确认 | 对抗性验证, 5 种检查策略, 必须实际运行命令 | "not confirm it works — try to break it" | `tools/AgentTool/built-in/verificationAgent.ts:10-152` |
| Fork Subagent | `forkSubagent.ts:210` — feature('FORK_SUBAGENT'), maxTurns:200 | 父子上下文继承, prompt cache sharing | byte-identical prefix for cache optimization | `tools/AgentTool/forkSubagent.ts:60-71, 107-169` |
| Coordinator Mode | `coordinator/coordinatorMode.ts:369` — feature('COORDINATOR_MODE') gate | 协调者-工作者模式, session mode auto-sync | 动态 require workerAgent module (dead code elimination) | `coordinator/coordinatorMode.ts:36-41` |
| TodoWrite tool | `tools/TodoWriteTool/` — structured task tracking | Agent 结构化追踪子任务进度 | `tools.ts:56` explicitly imported | `tools/TodoWriteTool/` |
| AutoCompact | `services/compact/autoCompact.ts` | Token 超限时自动压缩历史消息 | 与 reactiveCompact/snipCompact/collapse 并列 4 种 compaction | `services/compact/` (11 files) |

### 维度四: 安全隔离 (D4)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 源码出处 |
|----------|-------------|---------|------------|---------|
| Sandbox 四层结构 | shouldUseSandbox → convertToSandboxRuntimeConfig → PermissionRule → execute+cleanup | 决策-配置-权限-执行解耦 | 985 行 adapter, 每层独立可替换 | `utils/sandbox/sandbox-adapter.ts:172-985` |
| @anthropic-ai/sandbox-runtime | 外部专业沙箱运行时封装 | 进程级隔离执行 | 自有自研沙箱 runtime, 非复用 Docker/npm | `utils/sandbox/sandbox-adapter.ts:7-22` |
| SandboxViolationStore | 从 sandbox-runtime 引入 | 违规事件追踪与存储 | 违规可审计 | `utils/sandbox/sandbox-adapter.ts:21` |
| Permission 三层 (allow/deny/ask) | `Tool.ts:123-138` — alwaysAllowRules/alwaysDenyRules/alwaysAskRules | 细粒度工具级权限 | 权限按 source (user/project/local/managed) 分层 | `Tool.ts:123-138` |
| PermissionMode | 'default' / 'acceptEdits' / 'bypassPermissions' / 'plan' 等 | 不同安全级别对应不同审批策略 | plan mode 可降权 | `Tool.ts:123-124` |
| File system restrictions | `sandbox-adapter.ts` — FsReadRestrictionConfig + FsWriteRestrictionConfig | 只读/只写路径精确控制 | 支持 // 前缀 (绝对) 和 / 前缀 (settings-relative) | `sandbox-adapter.ts:99-146` |
| Policy settings | 'policySettings' source — allowManagedDomainsOnly / allowManagedReadPathsOnly | 企业级不可覆盖策略 | 管理源最高优先级 | `sandbox-adapter.ts:152-164` |
| Denial tracking | `utils/permissions/denialTracking.js` | 拒绝记录与统计 | 安全审计支持 | `Tool.ts:60` (DenialTrackingState import) |

### 维度五: 记忆系统 (D5)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 源码出处 |
|----------|-------------|---------|------------|---------|
| 5 层 Memory 类型 | User / Project / Local / Managed / AutoMem + TeamMem (feature gate) | 从个人到团队的分层记忆 | 类型化 memory, 不同生命周期 | `utils/memory/types.ts:3-10` |
| MEMORY.md Entrypoint | `memdir/memdir.ts:34-103` — MAX_ENTRYPOINT_LINES=200, MAX_ENTRYPOINT_BYTES=25_000 | 结构化索引文件，支持 topic 链接 | 双层 cap (line+byte) 防止超大索引 | `memdir/memdir.ts:38, 57-103` |
| loadMemoryPrompt | `memdir/memdir.ts` export, 从文件系统加载 memory | MEMORY.md 索引注入 system prompt | 文件系统级持久化, 版本可控 | `memdir/memdir.ts` |
| SessionStorage | `utils/sessionStorage.ts:5105` — JSONL transcript logging | 会话级完整记录，支持 replay/resume | 5105 行复杂存储层, 包含 compaction/session 切换 | `utils/sessionStorage.ts` |
| Auto Memory (AutoMem) | `memdir/paths.js` — getAutoMemPath + isAutoMemoryEnabled | Agent 自动创建和维护 memory | 自动化记忆管理，区别于手动 MEMORY.md | `memdir/paths.js` |
| Team Memory (TEAMMEM) | `services/teamMemorySync/` — feature-gated team memory sync | 团队共享记忆，含 secret scanner | secretScanner 防止 secrets 写入共享 memory | `services/teamMemorySync/secretScanner.ts` |
| Content Replacement | `utils/toolResultStorage.ts` — ContentReplacementState | 工具输出替换为引用, 减少 token 消耗 | 记录 → sessionStorage → contentReplacement 回放 | `query.ts:99`, `utils/sessionStorage.ts:46` |
| Snip Compact | `services/compact/snipCompact.js` — feature('HISTORY_SNIP') | 对话历史裁剪, SDK 专用 | SDK 无 UI scrollback, 需强力压缩 | `QueryEngine.ts:169-173` |

### 维度六: 扩展生态 (D6)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 源码出处 |
|----------|-------------|---------|------------|---------|
| MCP 深度集成 | `tools/MCPTool/MCPTool.ts` — lazySchema, isOpenWorld=false, maxResult=100K | 外部 MCP Server 工具动态注入 agent tool pool | MCP tool 表现为原生 tool, 统一权限/渲染 | `tools/MCPTool/MCPTool.ts:27-77` |
| MCP Connection Manager | `services/mcp/MCPConnectionManager.tsx` | MCP 服务器连接生命周期管理 | React 组件化 MCP 管理 | `services/mcp/MCPConnectionManager.tsx` |
| MCP WebSocket Transport | `utils/mcpWebSocketTransport.ts:200` | WebSocket 双向 MCP 传输 | JSONRPCMessageSchema 验证 | `utils/mcpWebSocketTransport.ts:82` |
| Plugin 系统 | `utils/plugins/pluginLoader.js` + `commands/plugin/` (10+ .tsx files) | 可插拔功能扩展 + Marketplace + Trust | 完整 Plugin 生态: 安装/信任/验证/管理 | `commands/plugin/plugin.tsx`, `commands/plugin/BrowseMarketplace.tsx` |
| Skills 系统 | `skills/loadSkillsDir.js` — clearSkillCaches + onDynamicSkillsLoaded | 渐进式信息披露的 skill 加载 | 支持 dynamic skills (运行时加载) | `skills/loadSkillsDir.js` |
| Skill 变更检测 | `utils/skills/skillChangeDetector.ts:311` — chokidar file watcher | 文件变更 1000ms 稳定阈值后自动 reload | 含 Bun 死锁 workaround (usePolling) | `utils/skills/skillChangeDetector.ts:27-62` |
| Hooks 生命周期 | `utils/hooks.js` + `hooks/postSamplingHooks.js` + `hooks/useCanUseTool.js` | pre/post tool execution hooks | pre-compact/post-compact/session_start 全生命周期 | `services/compact/compact.ts:55-56` |
| Remote 远程 | `remote/RemoteSessionManager.ts` + `remote/remotePermissionBridge.ts` | 远程 agent 会话管理 | remote env var (CLAUDE_CODE_REMOTE) 控制 | `entrypoints/cli.tsx:9` |
| Worktree 隔离 | `tools/EnterWorktreeTool/` + `tools/ExitWorktreeTool/` | git worktree 级别的代码隔离 | 利用 git worktree 实现多分支并行工作 | `tools/EnterWorktreeTool/`, `tools/ExitWorktreeTool/` |
| Task System | `utils/task/framework.js` + 5 Task* Tools | 异步后台任务创建/管理/追踪 | registerTask, task disk output | `utils/task/framework.js`, `tools/TaskCreateTool/` |

---

## 三、技术链路验证

### 3.1 正常路径: CLI 对话 → 多轮工具调用

基于源码链路还原：

```
User types: "Refactor the auth module to use JWT instead of sessions"
  │
  ├─ Step 1: entrypoints/cli.tsx:33 main()
  │    ├─ parse args, fast-paths checked (--version, --dump-system-prompt)
  │    └─ dynamic import init.ts → config load → launchRepl
  │
  ├─ Step 2: QueryEngine.submitMessage(message)    [QueryEngine.ts:184]
  │    ├─ processUserInput() → command resolution    [utils/processUserInput/]
  │    ├─ Context assembly:
  │    │    ├─ loadMemoryPrompt() + getAutoMemPath() → MEMORY.md injection
  │    │    ├─ fetchSystemPromptParts() → CLAUDE.md, system prompt [QueryEngine.ts:72]
  │    │    ├─ agentDefinitions → 6 built-in agents injected [QueryEngine.ts:135]
  │    │    ├─ mcpClients → MCP tools discovered (懒加载 schema)
  │    │    ├─ skills → matched skills injection (name + description only)
  │    │    └─ plugins → plugin-loaded tools via loadAllPluginsCacheOnly()
  │    │
  │    └─ Compact check: autoCompact tracking → if needed → compact() [services/compact/autoCompact.ts]
  │
  ├─ Step 3: query() async generator loop    [query.ts:219]
  │    ├─ buildQueryConfig() → createDumpPromptsFetch()    [query/config.js]
  │    │
  │    ├─ LLM Request: queryModelWithStreaming()    [services/api/claude.ts]
  │    │    ├─ Anthropic API call with streaming
  │    │    ├─ yield each stream event (thinking, text, tool_use)
  │    │    └─ doesMostRecentAssistantMessageExceed200k() check [utils/tokens.ts:85]
  │    │
  │    ├─ Tool dispatch: runTools()    [services/tools/toolOrchestration.ts]
  │    │    ├─ findToolByName() → match from tools.ts registry [Tool.ts]
  │    │    ├─ checkPermissions() → permission mode resolution
  │    │    ├─ StreamingToolExecutor → real-time output streaming
  │    │    └─ mapToolResultToToolResultBlockParam() → normalized result
  │    │
  │    └─ Continue → next iteration or Terminal → stop    [query/transitions.js]
  │
  ├─ Step 4: Multi-turn execution
  │    ├─ Turn 1: search_files("auth"), read_file("src/auth/index.ts")
  │    ├─ Turn 2: glob("src/auth/**/*.ts"), read_file("src/auth/session.ts")
  │    ├─ Turn 3: FileWriteTool("src/auth/jwt.ts"), FileEditTool("src/auth/index.ts")
  │    └─ Turn 4: BashTool("npm run test -- --auth") → sandbox if needed
  │
  └─ Step 5: Post-processing
       ├─ compact() if token budget exceeded [services/compact/compact.ts]
       ├─ SessionStorage.flush() → JSONL transcript [utils/sessionStorage.ts]
       ├─ postSamplingHooks → execute [hooks/postSamplingHooks.js]
       └─ ContentReplacement → tool output replaced with refs [utils/toolResultStorage.ts]
```

### 3.2 异常路径: Sandbox 隔离执行

```
Agent 决定执行用户提供的脚本
  │
  ├─ Step 1: Decision Layer    [sandbox-adapter.ts]
  │    ├─ SandboxManager.isSupportedPlatform()    [line 985]
  │    └─ SandboxManager.isSandboxEnabledInSettings()
  │
  ├─ Step 2: Configuration Layer    [sandbox-adapter.ts:172]
  │    ├─ convertToSandboxRuntimeConfig(settings)
  │    │    ├─ WebFetch domain extraction → allowedDomains/deniedDomains
  │    │    ├─ managedDomainsOnly check → policy override
  │    │    └─ resolvePathPatternForSandbox() → // → /, / → settings-relative
  │    │
  │    └─ SandboxRuntimeConfig: filesystem restrictions + network policy
  │
  ├─ Step 3: Permission Layer
  │    ├─ PermissionRule matching: toolName + ruleContent
  │    ├─ Bash command → allowlist check
  │    ├─ WebFetch → domain allow/deny via network config
  │    └─ SandboxPermissionRequest.tsx: host-level user approval UI
  │
  └─ Step 4: Execution + Cleanup Layer
       ├─ @anthropic-ai/sandbox-runtime executes in isolated process
       ├─ SandboxViolationStore logs any violations
       ├─ Timeout: implicit (via sandbox-runtime)
       └─ cleanup: rmSync transient dirs [sandbox-adapter.ts:23]
```

### 3.3 Multi-Agent 协作: Fork Subagent 路径

```
Agent call with no subagent_type specified
  │
  ├─ Step 1: isForkSubagentEnabled()    [forkSubagent.ts:32]
  │    ├─ feature('FORK_SUBAGENT') gate
  │    ├─ Not isCoordinatorMode() (mutually exclusive)
  │    └─ Not nonInteractiveSession
  │
  ├─ Step 2: isInForkChild() guard    [forkSubagent.ts:78]
  │    ├─ Scan messages for FORK_BOILERPLATE_TAG
  │    └─ Prevent recursive forking
  │
  ├─ Step 3: buildForkedMessages()    [forkSubagent.ts:107]
  │    ├─ Clone parent assistant message (all tool_use blocks)
  │    ├─ Build placeholder tool_results (identical for cache sharing)
  │    └─ Append per-child directive (FORK_DIRECTIVE_PREFIX)
  │
  ├─ Step 4: execute async fork with FORK_AGENT definition    [forkSubagent.ts:60]
  │    ├─ tools: ['*'] (exact parent tool pool)
  │    ├─ maxTurns: 200
  │    ├─ model: 'inherit'
  │    └─ permissionMode: 'bubble' → prompts surface to parent terminal
  │
  └─ Step 5: Child agent executes → scoped result → parent continues
```

### 3.4 资源表现 (源码直接提取)

| 参数 | 精确值 | 源码出处 |
|------|--------|---------|
| FORK_AGENT maxTurns | 200 | `forkSubagent.ts:65` |
| MEMORY.md max lines | 200 | `memdir/memdir.ts:35` |
| MEMORY.md max bytes | 25,000 | `memdir/memdir.ts:38` |
| MCPTool maxResultSizeChars | 100,000 | `tools/MCPTool/MCPTool.ts:35` |
| Skill change stability threshold | 1000ms | `utils/skills/skillChangeDetector.ts:27` |
| Skill reload debounce | 300ms | `utils/skills/skillChangeDetector.ts:42` |
| Skill polling interval (Bun) | 2000ms | `utils/skills/skillChangeDetector.ts:49` |
| Compact post-compact token budget | 50,000 | `services/compact/compact.ts:123` |
| Compact post-compact max files | 5 | `services/compact/compact.ts:122` |
| Compact post-compact skills budget | 25,000 | `services/compact/compact.ts:130` |
| MAX_OUTPUT_TOKENS recovery limit | 3 attempts | `query.ts:164` |
| maxOutputTokensRecovery | 3 | `query.ts:164` |
| Iterator yield types | StreamEvent / RequestStartEvent / Message / TombstoneMessage / ToolUseSummaryMessage | `query.ts:222-228` |

---

## 四、Gap Analysis (差异分析)

### 4.1 核心风险与缺失项

| 缺失/风险项 | 检测来源 | 影响等级 | 详情 |
|--------|---------|---------|------|
| 闭源风险 — 非官方开源 | 事实 | **致命** | 基于 npm source map 泄露逆向。Anthropic 随时可能修改/关闭/收费。无开源社区贡献路径 |
| 无公开 Benchmark | 源码无评测框架 | 高 | 源码中未见 SWE-bench/GAIA/HumanEval 评测集成 |
| 依赖 Anthropic API | `services/api/claude.ts` 单向调用 | 高 | 深度绑定 Claude 模型。无 OpenAI/Llama/其他 provider 切换 |
| 单模型 architecture | `utils/model/model.js` — getMainLoopModel() 仅返回 Claude variants | 高 | Coordinator 模式可能用不同 worker 模型，但同 ecosystem |
| 无 Gateway 多平台 | 源码中无消息平台集成 | 中 | 纯 CLI/SDK/Remote, 无 Discord/Slack/Telegram 等集成 |
| Browser automation | 仅通过 MCP (claude-in-chrome MCP server) | 中 | 无原生 browser tools, 依赖外部 MCP server |
| workerAgent 未在源码中 | feature-gated + dead code elimination | 低 | workerAgent.ts 通过动态 require, 不在当前 build 中 |
| 无多租户隔离 | 源码中未见 tenant/user 隔离 | 低 | session-level 隔离存在, 但无 server 多租户 |

### 4.1.1 补充核查：与已知开源产品的 Gap 对比

| 能力维度 | Claude Code (源码) | Hermes Agent | DeepAgents | Gap 判定 |
|---------|-------------------|-------------|------------|---------|
| 多 Provider 支持 | ❌ 单 Anthropic API | ✅ 200+ via OpenRouter | ✅ 20+ providers | Claude Code 锁定单生态 |
| 开源许可 | ❌ 闭源 | ✅ MIT | ✅ MIT | Claude Code 不可自托管 |
| Gateway 多平台 | ❌ 无 | ✅ 18+ 消息平台 | ❌ 无 | — |
| Skills 自我创建 | ❓ 有 AutoMem 但未确认 self-create | ✅ 自动从经验创建 | ❌ 手动声明 | — |
| Sandbox 后端 | ✅ `@anthropic-ai/sandbox-runtime` | ✅ 6 Backend | ✅ 5 Provider | 三者均强 |
| MCP 集成 | ✅ 深度集成 (原生 tool wrapper) | ✅ 双向 MCP | ✅ MCP Client | Claude Code MCP 作为顶层 tool |
| Multi-Agent | ✅ 6 built-in + fork + coordinator + swarm | ✅ subagent + kanban | ✅ 3 形态 subagent | Claude Code 最复杂 |
| 公开 Benchmark | ❌ 无 | ✅ batch_runner + mini_swe | ✅ 108 evals × 7 cats | DeepAgents 评测最成熟 |
| Browser 自动化 | ⚠️ 仅 via MCP server | ✅ 12 browser tools | ❌ 无 | — |
| Plugin Marketplace | ✅ commands/plugin/ | ❌ 无 | ❌ 无 | Claude Code 独有 |
| 分析系统强度 | ✅ GrowthBook (100+ gates) + Statsig + Datadog | ❌ 无 | ❌ 无 | Claude Code 数据驱动 feature rollout |

### 4.2 基于源码的潜在问题

- **Feature Gate 泛滥**: GrowthBook 100+ 实验开关, 代码中大量 `feature('X') ? require() : null` 模式, 增加测试矩阵和分支复杂度
- **Bun 特定风险**: 多处 Bun workaround (`typeof Bun !== 'undefined'`, USE_POLLING due to Bun deadlock), 降低跨运行时兼容性
- **React Compiler 引入**: 所有 .tsx 组件使用 `react/compiler-runtime`, 增加构建复杂度
- **Dead Code Elimination 过度**: `feature()` + conditional `require()` 模式导致某些代码路径在特定 build 中完全不可见 (如 workerAgent.ts)
- **I/O 耦合深**: sessionStorage 直接操作 JSONL 文件系统 (5105 行), sandbox 直接调用 rmSync, 测试困难

---

## 五、维度建议 (Dimension Evolution)

### 5.1 Agent vs Agent 平台判定

| 判定标准 | Claude Code 源码现状 | 判定 |
|---------|-----------------|------|
| **模板化** | AgentDefinition type 支持 tools/model/permissionMode/disallowedTools/omitClaudeMd/source/baseDir。CustomAgentDefinition 继承。Skills 支持 Markdown 声明。但 Agent 实例创建路径未完全可见 (闭源限制) | ✅ 部分达成 |
| **隔离化** | ✅ Sandbox 四层结构 (`@anthropic-ai/sandbox-runtime`); SubAgent 隔离上下文; Fork subagent 继承+隔离; Coordinator workers 独立 tool set; Worktree 文件系统隔离; PermissionMode per-agent | ✅ 完全达成 |

> **结论**: Claude Code 在**隔离化**维度完全达到 Agent 平台标准 (Sandbox + SubAgent + Perm + Worktree)，在**模板化**维度因 AgentDefinition 类型系统和 CustomAgent 支持显示出平台特征。从架构复杂度 (1884 文件) 和多入口设计判断，Claude Code 是一个**闭源 Agent 平台**而非简单 CLI 工具。

### 5.2 值得关注的架构特征 (供 CMA 参考)

| 特征 | 描述 | 参考价值 |
|------|------|---------| 
| **Sandbox 四层结构** | shouldUse → convertToConfig → PermissionRule → execute+cleanup | 分层决策模型 — 决策/配置/权限/执行完全解耦, 985 行 adapter |
| **6 个专业化 Built-in Agents** | GeneralPurpose + Plan(只读) + Explore(快速) + Verification(对抗) + Guide + Statusline | 不同角色有不同 system prompt, model, tool set, constraint — 专业化分工 |
| **Fork Subagent 的 Cache Sharing** | byte-identical prefix 所有 fork children, 仅 directive 不同 | 利用 Anthropic prompt cache 的巧妙设计 — 跨子 agent 共享缓存 |
| **3 种 Multi-Agent 拓扑共存** | Fork (父子继承) + Coordinator (层级) + Swarm (对等/tmux) | 不同任务类型匹配不同拓扑 — 灵活性与复杂度共存 |
| **Feature Gate 驱动的架构演化** | 100+ GrowthBook flags 控制所有核心功能是否启用 | 数据驱动功能发布, 但增加代码分支复杂度 |
| **Hooks 生命周期** | pre/post compact + session_start + post sampling hooks | 标准化注入点 — 审计/记忆/权限/日志均可横切 |
| **双路径 Teammate** | AsyncLocalStorage (in-process) + dynamicTeamContext (tmux) | 同一接口支持不同运行模式 (进程内 vs 进程外) |
| **MEMORY.md 双层 truncation** | 200 lines + 25KB bytes 双层上限 | 防止超大索引破坏 prompt, 含降级警告 |

---

## 六、依赖分析 (隐式能力推导)

基于 TypeScript 源码导入推导：

| 推断依赖 | 推导能力 | 确定性 | 源码出处 |
|---------|---------|--------|---------|
| React + Ink | 终端 TUI 渲染 (JSX) | **确定** | `entrypoints/cli.tsx`, all `commands/*.tsx` |
| React Compiler (`react/compiler-runtime`) | JSX 编译优化 | **确定** | 所有 .tsx 文件 `import { c as _c } from "react/compiler-runtime"` |
| `@anthropic-ai/sdk` | Claude API 调用 + types | **确定** | `query.ts:2,5`, `Tool.ts:1-3` |
| `@anthropic-ai/sandbox-runtime` | 沙箱引擎 | **确定** | `sandbox-adapter.ts:7-22` |
| `@modelcontextprotocol/sdk` | MCP 协议 | **确定** | `tools/MCPTool/MCPTool.ts:7-8` (ElicitRequestURLParams), `utils/mcpWebSocketTransport.ts:1-5` |
| `chokidar` | 文件监听 | **确定** | `utils/skills/skillChangeDetector.ts:1` |
| `zod/v4` | Schema 验证 | **确定** | `tools/MCPTool/MCPTool.ts:1` (z from 'zod/v4') |
| `lodash-es` | 工具函数 (uniqBy, memoize, last) | **确定** | 多处: `tools.ts:86`, `sessionStorage.ts:18`, `QueryEngine.ts:4` |
| `ws` (WebSocket) | Node WebSocket 实现 | **确定** | `utils/mcpWebSocketTransport.ts:6` (import type WsWebSocket from 'ws') |
| `strip-ansi` | ANSI 去除 | **确定** | `QueryEngine.ts:78` |
| Bun runtime | 构建 + 原生 WebSocket + MACRO | **确定** | `bun:bundle` feature imports, `typeof Bun !== 'undefined'` checks |
| `diff` library | 推断: patch 生成和应用 | **中-高** | FileEditTool 逻辑推断 |
| `ripgrep` / embedded ugrep | 代码搜索 | **确定** | `sandbox-adapter.ts:59` (ripgrepCommand import), `constants/tools.ts` hasEmbeddedSearchTools() |
| GrowthBook + Statsig + Datadog | 分析系统 | **确定** | `services/analytics/` (8 files) |

---

## 七、独特性汇总

1. **Sandbox 四层结构**: `utils/sandbox/sandbox-adapter.ts` — 决策层 → 配置层(convertToSandboxRuntimeConfig) → 权限层(PermissionRule) → 执行层(@anthropic-ai/sandbox-runtime + cleanup)。985 行完整实现，层间完全解耦。

2. **6 个专业化 Built-in Agents**: `tools/AgentTool/builtInAgents.ts:22-72` — 不同 agent 有独立的 system prompt (GENERAL: 通用多步, PLAN: 只读架构师, EXPLORE: 快速搜索 haiku/ant inherit, VERIFICATION: 对抗式验证 "try to break it"), 独立的 tool set (禁写/禁 agent/tools=['*']), 独立的 model 配置。这是目前所有已分析产品中 Agent 专业化程度最高的设计。

3. **Fork Subagent 的 Cache Sharing 设计**: `tools/AgentTool/forkSubagent.ts:107-169` — 所有 fork children 使用 byte-identical API request prefix (placeholder 结果相同), 仅 directive 不同。这是对 Anthropic prompt cache 的深度利用，在已知分析产品中未见。

4. **三套 Multi-Agent 模式并存**: Fork (父子继承, maxTurns=200) + Coordinator→Workers (层级协调者, feature gate) + Swarm/Teammate (对等 tmux/in-process/external 三 backend)。`utils/teammate.ts` 双路径 identity resolution (AsyncLocalStorage vs dynamicTeamContext)。

5. **多入口统一内核**: CLI (entrypoints/cli.tsx) / SDK / print (headless) / Remote (SessionsWebSocket) / MCP Server (claude-in-chrome) / Chrome Native Host — 六种入口共享同一个 `QueryEngine` + `query.ts` 内核。

6. **StreamingToolExecutor**: `services/tools/StreamingToolExecutor.ts` — 工具执行中实时流式返回中间输出，让 LLM 不等工具完成即可开始下一步推理。在已知分析产品中未见同类机制。

7. **GrowthBook 驱动的 Feature Gate 体系**: 100+ 实验开关 (`feature('X')`) 控制几乎所有核心功能。代码中充斥 `feature('X') ? require() : null` (dead code elimination) 模式 — 这是 data-driven product development 在大规模 TypeScript agent 项目中的典范。

8. **MEMORY.md 双层 Truncation**: `memdir/memdir.ts` — 200 行 + 25KB 双层上限，同时检查 line count 和 byte count，智能降级警告 (精确指出是 line cap 还是 byte cap)。

9. **Verification Agent 的 Adversarial 设计**: `tools/AgentTool/built-in/verificationAgent.ts` — "Your job is not to confirm the implementation works — it's to try to break it"。包含 9 种验证策略 (Frontend/Backend/CLI/Infra/Library/Bug/Mobile/Data/Migration) + adversarial probes (Concurrency/Boundary/Idempotency/Orphan ops)。要求每个 PASS check 必须包含实际命令运行。

10. **Skill Change Detector 的 Bun 死锁 Workaround**: `utils/skills/skillChangeDetector.ts:52-62` — Bun's fs.watch() 有 PathWatcherManager deadlock (oven-sh/bun#27469)，因此 force USE_POLLING under Bun。这是生产级工程对 runtime bug 的现实响应。

---

## 八、校准备忘录 (Calibration)

### 8.1 事实核查表

| 声明 | 验证状态 | 来源验证 |
|------|---------|---------|
| 源码语言: TypeScript | ✅ 准确 | src/ 目录 1884 .ts/.tsx 文件 |
| 6 个 Built-in Agents | ✅ 准确 | `tools/AgentTool/builtInAgents.ts:22-72` 明确列出 |
| Sandbox 四层结构 | ✅ 准确 | `utils/sandbox/sandbox-adapter.ts:985` 完整实现 |
| Fork Subagent | ✅ 准确 | `tools/AgentTool/forkSubagent.ts:210` 完整实现 |
| Swarm/Teammate 三 Backend | ✅ 准确 | `utils/swarm/` + `utils/teammate.ts:292` |
| Coordinator Mode | ✅ 准确 (feature gated) | `coordinator/coordinatorMode.ts:369` |
| Skill 变更检测 | ✅ 准确 | `utils/skills/skillChangeDetector.ts:311` (chokidar, 1000ms threshold) |
| Memory 5 层类型 | ✅ 准确 | `utils/memory/types.ts:3-10` |
| MCP WebSocket Transport | ✅ 准确 | `utils/mcpWebSocketTransport.ts:200` |
| StreamingToolExecutor | ✅ 准确 | `services/tools/StreamingToolExecutor.ts`, `query.ts:96` |
| 40+ Tools registry | ✅ 准确 | `tools.ts:3-97` (40+ explicit tool imports) |
| 非官方开源 | ✅ 准确 | 事实: 基于 npm source map 泄露 |

### 8.2 需修正/标注不确定项

| 声明 | 问题 | 修正/标注 |
|------|------|----------|
| workerAgent.ts | 通过 `feature('COORDINATOR_MODE') ? require() : null` 动态引入, 当前 build 中不存在 | 标注为「feature-gated, 不存在于当前源码 build」 |
| Browser 自动化 | 源码中未见原生 browser tools, 仅通过 MCP (claude-in-chrome MCP server) 提供 | 标注为「仅 MCP 路径」 |
| 完整 Tools 数量 | tools.ts 中 40+ 显式 import + MCP 动态 + Plugin 动态 | 标注为「~40+ 内置 + 动态」 |
| VerifyPlanExecutionTool | 由 `CLAUDE_CODE_VERIFY_PLAN` env 控制 | 标注为「条件引入」 |
| Skills 自我创建 | AutoMem 存在但未确认是否自动从经验创建 skills | 标注为「未确认」 |

### 8.3 遗漏项

| 遗漏项 | 原因 |
|--------|------|
| workerAgent.ts 完整实现 | Feature-gated, dead code eliminated from this build |
| Plugin 系统具体 API 表面 | `utils/plugins/pluginLoader.js` 未被完整读取 |
| Skills 加载机制完整细节 | `skills/loadSkillsDir.js` 未被完整读取 |
| Configuration 完整格式 | `utils/config.js` + `utils/settings/` 未被完整读取 |
| package.json 完整依赖树 | 仓库根目录可能不在 src/ 下 |
| Coordinator Mode worker scheduling logic | workerAgent.ts 不在当前 build |
| ALL custom agent definitions (非 built-in) | `tools/AgentTool/loadAgentsDir.js` 未被完整读取 |

### 8.4 总体准确性评分

- **源码直读直接支撑**: ~80% (绝大多数特征有具体源码文件/行号)
- **源码推理 (based on import graph)**: ~15%
- **合理推断 (标注)**: ~5%
- **总体评分: 高准确性** — 相比社区二手分析 (~40%)，源码直读提供了 ~80% 的直接支撑。核心特征 (6 agents, sandbox, fork, swarm, MCP, compact, memory, skill detector) 均有精确的文件和行号引用。

> ⚠️ **关键限制**: 本报告基于泄露源码的 src/ 目录直接阅读，非 Anthropic 官方文档。某些模块 (如 workerAgent) 因 feature gate + dead code elimination 不在当前 build 中。1884 个文件仅停留于关键架构模块的 depth-1 分析。

---

## 附录: 文件索引 (基于源码仓库)

| 源文件 | 路径 (src/ 下) | 源码行数 |
|--------|------|------| 
| 6 Built-in Agents 注册 | `tools/AgentTool/builtInAgents.ts` | 72 |
| GENERAL_PURPOSE Agent | `tools/AgentTool/built-in/generalPurposeAgent.ts` | 34 |
| EXPLORE Agent | `tools/AgentTool/built-in/exploreAgent.ts` | 83 |
| PLAN Agent | `tools/AgentTool/built-in/planAgent.ts` | 92 |
| VERIFICATION Agent | `tools/AgentTool/built-in/verificationAgent.ts` | 152 |
| CLAUDE_CODE_GUIDE Agent | `tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | — |
| STATUSLINE_SETUP Agent | `tools/AgentTool/built-in/statuslineSetup.ts` | — |
| Coordinator Mode | `coordinator/coordinatorMode.ts` | 369 |
| Fork Subagent | `tools/AgentTool/forkSubagent.ts` | 210 |
| Swarm/Teammate Identity | `utils/teammate.ts` | 292 |
| Swarm Constants | `utils/swarm/constants.ts` | 33 |
| Multi-Agent Spawn | `tools/shared/spawnMultiAgent.ts` | 1093 |
| Sandbox Adapter (4 层) | `utils/sandbox/sandbox-adapter.ts` | 985 |
| Sandbox Doctor | `components/sandbox/SandboxDoctorSection.tsx` | 45 |
| Sandbox Permission UI | `components/permissions/SandboxPermissionRequest.tsx` | 162 |
| Skill Change Detector | `utils/skills/skillChangeDetector.ts` | 311 |
| Memory Types | `utils/memory/types.ts` | 12 |
| Memory Entrypoint | `memdir/memdir.ts` | 507 |
| Session Storage | `utils/sessionStorage.ts` | 5105 |
| Compact Service | `services/compact/compact.ts` | 1705 |
| Compact Directory | `services/compact/` | 11 files |
| MCP Tool | `tools/MCPTool/MCPTool.ts` | 77 |
| MCP WebSocket Transport | `utils/mcpWebSocketTransport.ts` | 200 |
| Query Engine (class) | `QueryEngine.ts` | 1295 |
| Query Loop (generator) | `query.ts` | 1729 |
| Tool Type System | `Tool.ts` | 792 |
| Tool Registry | `tools.ts` | 389 |
| Tool Constants | `constants/tools.ts` | 112 |
| CLI Entrypoint | `entrypoints/cli.tsx` | 302 |
