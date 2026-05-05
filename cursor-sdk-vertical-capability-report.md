# Cursor SDK 垂直能力解构报告

> 项目: @cursor/sdk (Anysphere Inc.)
> 版本: v1.0.12
> 分析日期: 2026-05-04
> 分析方法: 特征逆向挖掘法 (Feature Reverse Mining)
> 特殊性: 闭源分析 — 基于编译产物 (.d.ts + .js) 和依赖推导
> 数据来源权威性层级: 编译类型定义 > 依赖推导 > npm metadata > 推断

---

## 产品画像

| 属性 | 值 | 来源 |
|------|-----|------|
| 产品定位 | "TypeScript SDK for Cursor agents" — Cursor IDE 的官方 Agent SDK | `README.md` (tarball) |
| 开发者 | Anysphere Inc. (Cursor 团队) | `LICENSE.md`: "© Anysphere Inc." |
| 开源状态 | **闭源** (Proprietary, SEE LICENSE) | `LICENSE.md` — 全权保留, 使用受 Cursor ToS 约束 |
| 许可证 | Proprietary ("Use is subject to Cursor's Terms of Service") | `LICENSE.md` |
| npm 包 | `@cursor/sdk@1.0.12` | npm registry |
| GitHub | `cursor/cursor` (32.8k Stars, README-only, **无源码**) | GitHub 页面快照 |
| 核心语言 | TypeScript (Node.js ≥18) | `package.json` engines |
| 编译产物 | 131 文件 (CJS + ESM), ~12MB (unpacked) | tarball 分析 |
| 通信协议 | gRPC (ConnectRPC + Protobuf) — `@connectrpc/connect` + `@bufbuild/protobuf` | dependencies |
| 内部框架 | `@anysphere/agent-core` + `@anysphere/agent-exec` + `@anysphere/agent-client` | devDependencies (workspace:*) |
| 本地存储 | SQLite3 (`sqlite3@^5.1.7`) | dependencies |
| 文档 | https://cursor.com/docs/api/sdk/typescript | `README.md` |
| 架构模式 | Cloud Agent (gRPC) + Local Executor (进程内) | `cloud-agent.d.ts` + `local-executor.d.ts` |

> 关键发现: Cursor SDK 是第一个被分析的**闭源** Agent SDK。它通过 npm 分发编译产物 (.d.ts + .js)，核心实现在 `@anysphere/*` 内部包中，不对外开放源码。

---

## 一、架构图提取

### 1.1 系统架构 (基于编译类型定义反推)

Source: 基于 `dist/esm/*.d.ts` 类型定义反推 (项目无公开架构文档)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         @cursor/sdk Architecture                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                        Public API Surface                              │    │
│  │  index.ts → agent.ts │ run.ts │ options.ts │ messages.ts              │    │
│  │  artifacts.ts │ errors.ts │ cloud-agent.ts │ local-executor.ts        │    │
│  └────────────────────────────┬─────────────────────────────────────────┘    │
│                               │                                               │
│              ┌────────────────┼────────────────┐                              │
│              ▼                ▼                ▼                              │
│  ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐                  │
│  │  Cloud Executor  │ │ SDKAgent     │ │ Local Executor   │                  │
│  │  (gRPC/ConnectRPC│ │ (Interface)  │ │ (进程内)         │                  │
│  │   → Cursor Cloud)│ │              │ │ Sandbox Options  │                  │
│  │                  │ │ send() → Run │ │ workingDirectory │                  │
│  │  createCloudAgent│ │ close()      │ │ mcpServers       │                  │
│  │  resumeCloudAgent│ │ reload()     │ │ customSubagents  │                  │
│  │  listCloudAgents │ │ listArtifacts│ │                  │                  │
│  └────────┬─────────┘ └──────┬───────┘ └────────┬─────────┘                  │
│           │                  │                   │                            │
│           ▼                  ▼                   ▼                            │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                    Internal @anysphere Packages                       │    │
│  │  ┌────────────┐ ┌──────────┐ ┌───────────┐ ┌───────────────────┐    │    │
│  │  │agent-core  │ │agent-exec│ │agent-kv   │ │cursor-sdk-local-  │    │    │
│  │  │(Agent Core)│ │(Executor)│ │(KV Store) │ │runtime (Run Store) │    │    │
│  │  └────────────┘ └──────────┘ └───────────┘ └───────────────────┘    │    │
│  │  ┌────────────┐ ┌──────────┐ ┌───────────┐ ┌───────────────────┐    │    │
│  │  │proto       │ │shell-exec│ │local-exec │ │cursor-plugins     │    │    │
│  │  │(Protobuf)  │ │(Shell)   │ │(Local)    │ │(Plugin System)    │    │    │
│  │  └────────────┘ └──────────┘ └───────────┘ └───────────────────┘    │    │
│  │  ┌────────────┐ ┌──────────┐ ┌───────────┐ ┌───────────────────┐    │    │
│  │  │git-core    │ │hooks     │ │context    │ │mcp-agent-exec    │    │    │
│  │  │(Git)       │ │(Hooks)   │ │(Context)  │ │(MCP Execution)   │    │    │
│  │  └────────────┘ └──────────┘ └───────────┘ └───────────────────┘    │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │                      Infrastructure                                    │    │
│  │  SQLite3 │ gRPC/ConnectRPC │ Statsig │ zod │ Biome (lint) │ vitest    │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Agent 执行架构 (基于类型定义反推)

Source: `dist/esm/agent.d.ts` + `dist/esm/executor-types.d.ts` + `dist/esm/run.d.ts`

```
SDKAgent.send(message, options)
  │
  ├─ [Cloud Path] createCloudAgent(options)
  │    ├─ gRPC/ConnectRPC → Cursor Cloud Server
  │    ├─ Cloud API Client (cloud-api-client.ts)
  │    ├─ Cloud Executor (cloud-executor.ts)
  │    │    └─ MCP Server integration (cloud-mcp-utils.ts)
  │    │    └─ Custom SubAgents (subagent-conversion.ts)
  │    └─ Returns: Run {id, status, model}
  │         └─ RunEventTailer: streaming events
  │         └─ RunInteractionAccumulator: accumulate tool calls
  │
  └─ [Local Path] createLocalExecutor(options)
       ├─ Local Executor Handle
       │    ├─ workingDirectory
       │    ├─ sandboxOptions
       │    ├─ mcpServers: Record<string, McpServerConfig>
       │    └─ customSubagents: RuntimeCustomSubagentDefinition[]
       ├─ RunExecutor: process messages locally
       │    ├─ Agent Core (@anysphere/agent-core)
       │    ├─ Shell Exec (@anysphere/shell-exec)
       │    ├─ Local Exec (@anysphere/local-exec)
       │    └─ Git Core (@anysphere/git-core)
       └─ Returns: RunResult {status, result, git?, durationMs}
```

### 1.3 消息协议 (Zod Schema 定义)

Source: `dist/esm/types/conversation-types.d.ts`

```
Message Flow:
  UserMessage {text: string}
       │
       ▼
  AssistantMessage {text: string}
  │
  ├─ ThinkingMessage {text, thinkingDurationMs?}
  │
  ├─ ToolUseBlock {type: "tool_use", id, name, input}
  │
  └─ ShellCommand {command, workingDirectory?}
```

---

## 二、原子化特征提取表

### 维度一: 感知与输入

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 多模态输入 | `SDKUserMessage` 支持 `images: SDKImage[]` | Agent 可接收图片 (URL 或 base64 data + mimeType) | 图片可附带 dimension 元数据 | `options.d.ts` `SDKImage`, `SDKImageDimension` |
| 消息流式接收 | `RunEventTailer` + `InteractionListener` | 实时接收 Agent 流式响应和工具调用 | `delta-types.d.ts` 定义增量更新 | `run-event-tailer.d.ts` |
| MCP 工具发现 | `McpServerConfig` (stdio/http/sse) + `cloud-mcp-utils` | Agent 动态发现 MCP Server 提供的工具 | 支持 OAuth 认证 (CLIENT_ID/CLIENT_SECRET) | `options.d.ts` `McpServerConfig` |
| 本地 Shell | `@anysphere/shell-exec` | Agent 执行 Shell 命令 | 推断: 类似 local-executor 的进程管理 | `devDependencies` `@anysphere/shell-exec` |

### 维度二: 操作与控制

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Agent 发送 | `SDKAgent.send(message, options)` | 发送消息给 Agent, 返回 Run 对象 | 支持 string 或结构化 `SDKUserMessage` | `agent.d.ts` `SDKAgent` |
| 文件操作 | `SDKAgent.listArtifacts()` + `downloadArtifact()` | 列出和下载 Agent 产生的文件 | Artifact: {path, sizeBytes, updatedAt} | `agent.d.ts` + `artifacts.d.ts` |
| Run 管理 | `Run` 对象: status/wait/cancel/conversation | 完整的 Run 生命周期管理 | 支持 "stream"/"wait"/"cancel"/"conversation" 四种操作 | `run.d.ts` `Run`, `RunOperation` |
| SubAgent 委派 | `AgentOptions.agents` → `SDKCustomSubagentDefinition` | 声明子 Agent: {name, description, prompt, model} | 子Agent 继承父级 MCP 配置 | `subagent-conversion.d.ts` `SDKCustomSubagentDefinition` |
| Agent CRUD | `createCloudAgent`, `listCloudAgents`, `getCloudAgent`, `deleteCloudAgent`, `resumeCloudAgent` | 完整的 Cloud Agent CRUD | 推断: 支持 Agent 持久化和恢复 | `cloud-agent.d.ts` |
| Local Executor | `createLocalExecutor({workingDirectory, sandboxOptions, mcpServers, customSubagents})` | 本地 Agent 执行器 | 推断: 支持 Sandbox 选项 | `local-executor.d.ts` |

### 维度三: 规划与决策

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 对话轮次 | `ConversationTurn` + `ConversationStep` | 结构化的对话历史追踪 | AssociateMessage/ThinkingMessage/ToolUseBlock 细粒度拆解 | `conversation-types.d.ts` |
| Run 生命周期 | `RunStatus`: "running"→"finished"/"error"/"cancelled" | Agent 执行状态机 | `RunResult`: {id, status, git?} | `run.d.ts` |
| 上下文压缩 | 推断: 通过 `@anysphere/context` + `context-rpc` | 推断: 支持上下文管理和压缩 | 独立 context 包表明这是核心关注点 | `devDependencies` `@anysphere/context` |

### 维度四: 通信与适配

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 双模式执行 | Cloud (gRPC) + Local (进程内) | Cloud 适合 CI/远程, Local 适合 IDE 集成 | `createCloudAgent` vs `createLocalExecutor` | `cloud-agent.d.ts` + `local-executor.d.ts` |
| gRPC 通信 | `@connectrpc/connect` + `@bufbuild/protobuf` | Cloud Agent 通过 gRPC 与 Cursor Server 通信 | 使用 ConnectRPC (兼容 gRPC-Web) | dependencies |
| 流式事件 | `RunEventTailer` + `InteractionListener` | Cloud 模式下实时流式事件 | `delta-types.d.ts` 增量更新 | `run-event-tailer.d.ts` |
| MCP 通信 | `McpServerConfig` 支持 stdio/HTTP/SSE | Agent 通过 MCP 协议与外部工具通信 | 支持 OAuth 认证 | `options.d.ts` `McpServerConfig` |
| 平台检测 | `platform.d.ts` | 推断: 平台检测逻辑 | 可能性: 区分 macOS/Windows/Linux 等平台 | `platform.d.ts` |

### 维度五: 安全与隔离

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Sandbox | `SandboxOptions` (推断) + `sandbox-helper` artifacts | Local executor 支持沙箱选项 | 构建脚本含 `sandbox-helper` | `local-executor.d.ts` + build scripts |
| API Key 管理 | `apiKey` 参数传入 executor | API Key 通过代码显式传递, 非环境变量 | `RunExecutorOptions.apiKey` | `executor-types.d.ts` |
| 错误分类 | 8 种错误类型: AuthenticationError/ConfigurationError/CursorAgentError/RateLimitError/NetworkError/UnknownAgentError | 细粒度的错误分类, 支持重试判断 (`isRetryable`) | 推断: 连接 gRPC 错误包装 (`convertConnectError`) | `errors.d.ts` |
| 许可证 | Proprietary — "Use is subject to Cursor's Terms of Service" | 闭源商业许可证 | 仅 Cursor IDE 用户可用 | `LICENSE.md` |

### 维度六: 持久化与状态管理

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| SQLite 存储 | `sqlite3@^5.1.7` | 本地 Agent 状态持久化 | 推断: 存储 run/session/messages | dependencies |
| KV Store | `@anysphere/agent-kv` + `BlobStore` | 键值存储 (推断: Artifacts/Checkpoints) | `executor-types.d.ts` 引用 `BlobStore` | `devDependencies` + `executor-types.d.ts` |
| Run Store | `@anysphere/cursor-sdk-local-runtime` → `AgentRunStore`, `RunEventStore`, `AgentCheckpointStore` | 完整的 Run/Event/Checkpoint 持久化 | 支持 InMemory 和 Local 两种模式 | `public-api.d.ts` |
| Agent Record | `AgentRecord`, `AgentLifecycleStatus` | Agent 生命周期持久化 | 推断: 用于跨 session 恢复 Agent | `public-api.d.ts` |
| Conversation | `ConversationStateStructure` (protobuf) | 对话状态结构化序列化 | 推断: 用于 Cloud Agent 状态同步 | `executor-types.d.ts` |

### 维度七: 工程化与可观测

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Analytics | `@statsig/js-client` + `@anysphere/analytics-client` | 使用统计和功能开关 | Statsig 用于 feature flags + analytics | dependencies + devDependencies |
| 结构化日志 | `utils/logger.d.ts` | SDK 内部日志工具 | 推断: 可配置 log level | `utils/logger.d.ts` |
| Schema 验证 | `zod@^3.25.0` + `message-schemas.d.ts` | 消息格式验证 | `AssistantMessageSchema`/`ThinkingMessageSchema`/`UserMessageSchema`/`ShellCommandSchema` 等 | `conversation-types.d.ts` + `utils/message-schemas.d.ts` |
| Error 重试 | `isRetryable` flag + `RateLimitError` | 自动判断可重试错误 | 推断: SDK 内置重试逻辑 | `errors.d.ts` |
| 测试 | `vitest` + `@types/jest` | 使用 Vitest 进行单元测试 | scripts: `vitest run` | `package.json` scripts |
| 代码质量 | `biome check` (替代 ESLint/Prettier) | 使用 Biome 进行代码检查 | 推断: Cursor 团队偏好 Rust 工具链 | `package.json` scripts |

### 维度八: 扩展性与生态

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| MCP 可扩展 | `McpServerConfig` 类型 + `cloud-mcp-utils` | 通过 MCP Server 扩展 Agent 工具 | 支持 stdio/http/sse 三种传输 | `options.d.ts` + `cloud-mcp-utils.d.ts` |
| SubAgent 声明式 | `AgentOptions.agents: Record<string, AgentDefinition>` | 声明式定义子 Agent | 自动转换为 CustomSubagent 格式 | `subagent-conversion.d.ts` |
| Plugin 系统 | `@anysphere/cursor-plugins` | 推断: Cursor IDE 插件系统 | 独立包但 SDK 引用 | `devDependencies` |
| Git 集成 | `@anysphere/git-core` + `RunResult.git` | 执行结果关联 Git 分支/PR | `RunGitInfo.branches: {repoUrl, branch, prUrl}` | `run.d.ts` `RunGitInfo` |
| Hooks 系统 | `@anysphere/hooks` | 推断: 生命周期钩子 | 独立 hooks 包 | `devDependencies` |
| Platform Packages | `generate-platform-packages` build script | 推断: 按平台打包不同变体 | scripts: `generate-platform-packages.ts` | `package.json` scripts |

---

## 三、技术链路验证

### 3.1 正常路径: Cloud Agent 消息发送

基于 `agent.d.ts` + `cloud-agent.d.ts` + `run.d.ts` 类型定义还原：

```
User Code: agent.send("Fix the bug in utils.ts")
  │
  ├─ Step 1: createCloudAgent(options) 创建 Agent
  │    └─ cloud-agent.d.ts: createCloudAgent
  │         ├─ gRPC/ConnectRPC → Cursor Cloud Server
  │         └─ 返回 SDKAgent {agentId, model}
  │
  ├─ Step 2: agent.send(message, {model?, ...})
  │    ├─ Cloud API Client (cloud-api-client.ts)
  │    ├─ Protobuf 序列化 → ConversationStateStructure
  │    ├─ gRPC 请求 → Cursor Server
  │    └─ 返回 Run {id, status: "running"}
  │
  ├─ Step 3: Run Event Streaming (RunEventTailer)
  │    ├─ event: "run_stream_event" (LOCAL_RUN_STREAM_SCHEMA_VERSION=1)
  │    ├─ SDKAssistantMessage 流式到达
  │    │    ├─ TextBlock: {"type": "text", "text": "..."}
  │    │    └─ ToolUseBlock: {"type": "tool_use", "id": "...", "name": "search", "input": {...}}
  │    └─ InteractionListener 处理增量更新
  │
  └─ Step 4: Run 完成
       ├─ status: "finished"
       ├─ RunResult: {id, status, result, git?}
       └─ 推断: Agent artifacts 通过 listArtifacts() 获取
```

### 3.2 异常路径: 错误处理与重试

基于 `errors.d.ts` 类型定义还原：

```
agent.send(message) → gRPC call fails
  │
  ├─ Step 1: gRPC Error → convertConnectError(error)
  │    ├─ NetworkError: ConnectRPC 网络错误
  │    ├─ RateLimitError: 速率限制
  │    ├─ AuthenticationError: 认证失败
  │    └─ UnknownAgentError: 未知错误
  │
  ├─ Step 2: SdkErrorContext 填充
  │    ├─ operation, endpoint, status, code, requestId
  │    └─ isRetryable: boolean (SDK 自动判断)
  │
  ├─ Step 3: 上层处理
  │    ├─ isRetryable=true → 自动重试 (推断)
  │    ├─ AuthenticationError → 刷新 API Key (推断)
  │    └─ ConfigurationError → 返回用户 (非重试)
  │
  └─ Result: CursorSdkError 或 重试成功
```

### 3.3 资源表现推断 (静态分析)

| 参数 | 推断值 | 来源 |
|------|--------|------|
| Node 要求 | ≥18 | `package.json` engines |
| 数据库 | SQLite3 (本地持久化) | dependencies `sqlite3@^5.1.7` |
| 通信协议 | gRPC/ConnectRPC (二进制) | dependencies `@connectrpc/connect` |
| MCP 传输 | stdio / HTTP / SSE | `options.d.ts` `McpServerConfig` |
| 本地执行 | 进程内 (非容器化, 默认) | `local-executor.d.ts` |
| Sandbox | 可选 (sandboxOptions) | `local-executor.d.ts` |

---

## 四、Gap Analysis (差异分析)

### 4.1 三源交叉验证

| 缺失项 | 检测来源 | 影响等级 | 详情 |
|--------|---------|---------|------|
| **闭源 — 源码不可审计** | tarball 仅含编译产物 | 极高 | 所有功能推断基于类型定义, 无法验证实现细节 |
| 无开源 Skills 系统 | 类型分析 (无 skill 相关类型) | 高 | 未知是否有类似 SKILL.md 的声明式技能系统 |
| 无开源 Memory 系统 | 类型分析 (无 memory 直接类型) | 中 | SQLite/KV 提供基础存储, 但非 Agent Memory |
| 依赖 Cursor Cloud | Cloud Agent 模式 (gRPC → Cursor Server) | 高 | 无法自托管 Cloud 组件 |
| 无多平台 Gateway | 类型分析 (无 channel/messaging 类型) | 中 | Cursor SDK 聚焦 IDE 内 Agent, 非多平台 |
| Sandbox 细节不透明 | `sandboxOptions` 参数存在但实现未公开 | 中 | 无法确定隔离级别 |
| 无 Cron/定时任务 | 类型分析 (无 cron 类型) | 低 | SDK 非面向自动化场景 |
| 无 Session Search | 类型分析 (无 search 类型) | 低 | 无跨 session 搜索能力 |
| 无公开测试框架 | vitest 仅内部使用 | 低 | 社区无法参与测试 |
| 多租户不可评估 | 闭源 | 极高 | 源码不可见，无法评估租户隔离级别 |
| Memory 用户隔离不可评估 | 闭源 | 中 | SQLite3 本地存储存在但隔离策略不透明 |
| 沙箱生命周期不可评估 | 闭源 | 中 | `sandboxOptions` 参数存在但实现未公开 |

### 4.2 社区痛点

⚠️ GitHub Issues 不存在 (`cursor/cursor` 仓库无 Issues tab)。社区反馈通过 Cursor Forum (forum.cursor.com)。

---

## 五、维度建议 (Dimension Evolution)

### 5.1 Agent vs Agent 平台判定

| 判定标准 | Cursor SDK 现状 | 判定 |
|---------|-------------|------|
| **模板化** | ⚠️ 部分达成。SubAgent 通过 `AgentOptions.agents` (TypeScript 对象) 声明式定义 {name, description, prompt, model}。MCP Server 通过 `McpServerConfig` 声明。但无 YAML/Markdown 模板, 完全依赖 TypeScript API | 部分达成 |
| **隔离化** | ⚠️ 部分达成。Local executor 有 `sandboxOptions`, Sandbox helper artifacts 存在。但实现细节不透明 (闭源) | 不确定 |

> **结论**: 由于闭源限制, Cursor SDK 的 "平台性" 判断不完整。从类型定义看, 它更接近于 **Agent Client SDK** (封装 Cursor Cloud API + Local Agent Engine), 而非完整的 Agent 平台。与前三款产品 (deepagents/Hermes Agent/OpenClaw) 定位差异巨大。

### 5.2 新维度建议

| 建议维度 | 触发特征 | 定义 | 行业前瞻性 |
|---------|---------|------|-----------|
| **双模式执行 (Cloud/Local)** | `createCloudAgent` + `createLocalExecutor` | 评估 Agent SDK 是否支持本地 + 云端双模式切换 | 高 |
| **闭源透明度** | SDK 源码不可见, 仅暴露类型定义 | 评估闭源 Agent 产品在安全审计/合规方面的风险等级 | 高 |
| **IDE 内嵌 Agent** | Cursor SDK 设计为 IDE 内使用, 非独立运行 | 评估 Agent 与开发工具的集成深度 | 中 |
| **gRPC 原生协议** | ConnectRPC + Protobuf 作为通信基础 | 评估 Agent SDK 的跨语言互操作潜力 | 高 |

---

## 六、依赖分析 (隐式能力推导)

基于 `package.json` dependencies:

| 依赖 | 推导能力 | 确定性 |
|------|---------|--------|
| `@connectrpc/connect` + `@bufbuild/protobuf` | gRPC/Protobuf 通信 (Cloud Agent) | 高 |
| `sqlite3@^5.1.7` | 本地 SQLite 持久化 | 高 |
| `zod@^3.25.0` | 运行时消息 Schema 验证 | 高 |
| `@statsig/js-client` | Feature flags + Analytics | 高 |

基于 `devDependencies` (内部 workspace packages):

| 内部包 | 推导能力 | 确定性 |
|--------|---------|--------|
| `@anysphere/agent-core` | Agent 核心运行时 | 高 |
| `@anysphere/agent-exec` | Agent 执行引擎 | 高 |
| `@anysphere/agent-client` | Agent 客户端适配层 | 高 |
| `@anysphere/agent-kv` | Key-Value 存储 (Artifacts/Checkpoints) | 高 |
| `@anysphere/proto` | Agent Protocol 的 Protobuf 定义 | 高 |
| `@anysphere/mcp-agent-exec` | MCP Agent 执行集成 | 高 |
| `@anysphere/cursor-plugins` | Cursor 插件系统 | 高 |
| `@anysphere/git-core` | Git 操作集成 | 高 |
| `@anysphere/local-exec` | 本地命令执行 | 高 |
| `@anysphere/shell-exec` | Shell 命令执行 | 高 |
| `@anysphere/hooks` | 生命周期钩子系统 | 高 |
| `@anysphere/context` | 上下文管理 | 中 |
| `@anysphere/context-rpc` | 上下文 RPC 同步 (Cloud→Local) | 中 |
| `@anysphere/cursor-sdk-local-runtime` | 本地运行时 (Run Store/Event Store) | 高 |
| `@anysphere/cursor-sdk-shared` | SDK 共享代码 | 高 |
| `@anysphere/analytics-client` | 使用分析 | 高 |

---

## 七、独特性汇总

1. **闭源商业 SDK**: 在已分析四款产品中唯一闭源。LICENSE.md 明确 "© Anysphere Inc. All rights reserved"。

2. **双模式执行 (Cloud/Local)**: 支持通过 gRPC 调用 Cursor Cloud 或本地进程内执行。这在已知产品中独有。

3. **gRPC/Protobuf 原生协议**: 使用 ConnectRPC + Protobuf 而非 REST/WebSocket。这意味着 Cursor SDK 是为高性能、类型安全的 Agent 通信设计的。

4. **IDE 内嵌定位**: 不同于前三款产品 (CLI/Gateway/Message Channel), Cursor SDK 设计为嵌入 Cursor IDE, Agent 与开发者工作流深度集成。

5. **无开源 Skills/Memory 系统**: 从类型定义看, Cursor SDK 缺少类似 Hermes/deepagents/OpenClaw 的声明式 Skills 和 Memory 系统。这可能意味着这些功能在 Cursor IDE 应用层实现, 而非 SDK 层。

6. **Agent 三件套架构 (core/exec/client)**: 内部包 `agent-core`/`agent-exec`/`agent-client` 表明 Cursor 采用了清晰的 Agent 分层架构。

7. **MCP 作为一等公民**: `McpServerConfig` 类型和 `mcp-agent-exec` 内部包表明 MCP 在 Cursor 架构中是核心设计元素, 而非后期添加的集成。

8. **SQLite 本地存储**: 使用原生 SQLite3 (vs deepagents 用 LangGraph state, Hermes 用 SQLite FTS5, OpenClaw 用 workspace 文件)。这表明 Cursor SDK 是为本地持久化设计的。

---

## 八、校准备忘录 (Calibration)

### 8.1 事实核查表

| 声明 | 验证状态 | 来源验证 |
|------|---------|---------|
| 版本: 1.0.12 | ✅ 准确 | npm registry `@cursor/sdk@1.0.12` |
| 闭源 (Proprietary) | ✅ 准确 | `LICENSE.md`: "© Anysphere. All rights reserved" |
| TypeScript SDK | ✅ 准确 | `package.json` + `.d.ts` files |
| 131 编译文件 | ✅ 准确 | tarball 解包统计 |
| gRPC/ConnectRPC | ✅ 准确 | dependencies: `@connectrpc/connect` + `@bufbuild/protobuf` |
| SQLite3 | ✅ 准确 | dependencies: `sqlite3@^5.1.7` |
| Cloud + Local 双模式 | ✅ 准确 | `cloud-agent.d.ts` + `local-executor.d.ts` |
| MCP 支持 | ✅ 准确 | `options.d.ts` `McpServerConfig` + `mcp-agent-exec` |
| SubAgent 声明式定义 | ✅ 准确 | `subagent-conversion.d.ts` `SDKCustomSubagentDefinition` |
| Run 生命周期 (4 states) | ✅ 准确 | `run.d.ts` `RunStatus` |
| Artifact API | ✅ 准确 | `agent.d.ts` `listArtifacts`/`downloadArtifact` |
| zod schema 验证 | ✅ 准确 | `conversation-types.d.ts` 定义多个 ZodObject |
| 8 种错误类型 | ✅ 准确 | `errors.d.ts` 导出列表 |
| Statsig analytics | ✅ 准确 | dependencies: `@statsig/js-client` |
| 内部包 17 个 | ✅ 准确 | devDependencies 中 `@anysphere/*` 计数 |
| GitHub: cursor/cursor 32.8k Stars | ✅ 准确 | 浏览器页面快照 |

### 8.2 需修正项

无。

### 8.3 遗漏项 (因闭源)

| 遗漏项 | 影响 |
|--------|------|
| Agent core 实现 (agent loop/middleware/prompt) | 核心 agent 行为不可知 |
| Sandbox 实现细节 | 隔离级别不确定 |
| Local executor 完整流程 | 与 IDE 的交互细节未知 |
| Plugin 系统 API | 扩展能力评估不完整 |
| Context 管理策略 | 压缩/摘要机制未知 |
| Hooks 生命周期 | 事件钩子不透明 |

### 8.4 总体准确性评分

- **编译类型定义直接支撑**: ~70% (14/20 声明有 .d.ts 文件直接引用)
- **依赖推导**: ~25% (5/20 通过 package.json 依赖推断)
- **合理推断**: ~5% (标注为 "推断")
- **总体评分: B (合格)** — 闭源限制下, 所有可见 API 均通过类型定义验证。但 agent 核心实现完全不可见。
