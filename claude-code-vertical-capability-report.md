# Claude Code 垂直能力解构报告

> 项目: Anthropic Claude Code (社区分析)
> 分析源: liuup/claude-code-analysis (2.2k stars)
> 分析日期: 2026-05-06
> 分析方法: 特征逆向挖掘法 (Feature Reverse Mining)
> 数据来源权威性层级: 泄露源码社区分析 > 官方文档 > 推断
> 特殊性: 基于 npm 包 source map 泄露的源码逆向分析，非官方开源；源码语言 TypeScript (1902 文件, 513k 行)

---

## 产品画像

| 属性 | 值 | 来源 |
|------|-----|------|
| 产品定位 | "本地代码 Agent 平台" — 面向代码工作流的本地 agent 平台，非命令行聊天工具 | 社区分析 README |
| 开发者 | Anthropic | 官方产品 |
| 分析仓库 | liuup/claude-code-analysis (2.2k stars) | GitHub |
| 源码语言 | TypeScript (1902 文件, 513k 行) | 仓库统计 |
| 许可证 | 非开源 (source map 泄露逆向) | 事实陈述 |
| 核心入口 | `cli.tsx` → `main.tsx` → `launchRepl` | analysis/cli.tsx, analysis/main.tsx |
| 执行内核 | `query.ts` / `QueryEngine.ts` | analysis/query.ts |
| CLI 架构 | React/Ink 终端 UI (JSX 组件) | analysis/cli.tsx |
| Tool 系统 | `Tool.ts` → `toolOrchestration` → `StreamingToolExecutor` | analysis/Tool.ts |
| 权限系统 | `Perm` 层 + `bashPermissions` | analysis/Perm/ |
| 配置系统 | `init.ts` / `setup.ts` → 初始化引导 | analysis/init.ts, analysis/setup.ts |
| 上下文管理 | `compact` 压缩 + session 管理 | analysis/compact/ |
| Memory | 分层: sessionStorage / memdir / SessionMemory / hooks | analysis/sessionStorage, analysis/memdir |
| 扩展机制 | MCP / Plugin / Skills / Remote / Bridge / Swarm | analysis/MCP/, analysis/Plugin/, analysis/Swarm/ |

> 推断: Claude Code 的定位是 Anthropic 对 Cursor/Copilot 生态的降维打击 — 不做 IDE 插件，而是做一个本地 agent 平台，通过 CLI + REPL + SDK + MCP + Remote 多入口覆盖从个人开发者到团队的代码工作流。其架构复杂度 (1902 文件, 513k 行) 远超一般 CLI 工具，反映了一个完整的 agent 操作系统级设计。

---

## 一、架构图提取

### 1.1 系统架构 (基于社区分析 README 提取)

Source: `liuup/claude-code-analysis` README

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Claude Code Architecture                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      Entry Layer                                    │    │
│  │  CLI (cli.tsx) │ REPL (launchRepl) │ SDK │ MCP │ Bridge │ Remote │    │
│  └────────────┬──────────────────────┬───────────────────────────────┘    │
│               │                      │                                     │
│               ▼                      ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                    Initialization Layer                             │    │
│  │  init.ts → setup.ts → config resolution → auth → context prep    │    │
│  └────────────────────────────┬─────────────────────────────────────┘    │
│                               │                                            │
│                               ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                  TUI / REPL Layer                                   │    │
│  │  ┌─────────────────────────────────────────────────────────────┐ │    │
│  │  │  React/Ink Components (JSX) → terminal rendering              │ │    │
│  │  │  Command Registry (commands.ts) → slash commands             │ │    │
│  │  └─────────────────────────────────────────────────────────────┘ │    │
│  └────────────────────────────┬─────────────────────────────────────┘    │
│                               │                                            │
│                               ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                    Execution Kernel                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐    │    │
│  │  │  Query Engine (query.ts / QueryEngine.ts)                  │    │    │
│  │  │    LLM Request → Tool Call → Result → Iterate             │    │    │
│  │  │    Compact: context compression / truncation              │    │    │
│  │  └──────────────────────────────────────────────────────────┘    │    │
│  │                                                                     │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐   │    │
│  │  │  Tool System      │  │  Permission      │  │  Sandbox      │   │    │
│  │  │  Tool.ts          │  │  Perm/           │  │  四层结构      │   │    │
│  │  │  toolOrchestration│  │  bashPermissions │  │  shouldUse→    │   │    │
│  │  │  StreamingTool    │  │                  │  │  convertTo→   │   │    │
│  │  │  Executor         │  │                  │  │  Shell.ts→    │   │    │
│  │  │                   │  │                  │  │  cleanup      │   │    │
│  │  └──────────────────┘  └──────────────────┘  └───────────────┘   │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      Subsystems                                     │    │
│  │                                                                      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐  │    │
│  │  │  Memory Layer    │  │  Context Mgmt   │  │  Extension        │  │    │
│  │  │  sessionStorage  │  │  compact/       │  │  MCP/             │  │    │
│  │  │  memdir/         │  │  context        │  │  Plugin/          │  │    │
│  │  │  SessionMemory   │  │  window mgmt    │  │  Skills/          │  │    │
│  │  │  hooks/          │  │                 │  │  Remote/          │  │    │
│  │  │                  │  │                 │  │  Bridge/          │  │    │
│  │  │                  │  │                 │  │  Swarm            │  │    │
│  │  └─────────────────┘  └─────────────────┘  └──────────────────┘  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                    Multi-Agent Layer                                │    │
│  │  SubAgent (isolated context) │ Coordinator → Workers │ Swarm     │    │
│  │  (三套 multi-agent 协作模式并存)                                     │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Tool 调用链路

Source: `analysis/Tool.ts` → `analysis/toolOrchestration` → `analysis/StreamingToolExecutor`

```
User Input
  │
  ├─ QueryEngine.process(user_message)
  │    │
  │    ├─ Context assembly: system prompt + tools + memory + skills
  │    │
  │    ├─ LLM Request → response with tool_calls
  │    │
  │    ├─ Tool.ts dispatch:
  │    │    ├─ File tools: read_file / write_file / patch / search_files / glob
  │    │    ├─ Shell tools: bash / shell (via Shell.ts, with sandbox)
  │    │    ├─ Code tools: edit / replace / apply_patch
  │    │    ├─ SubAgent: task (spawn isolated agent)
  │    │    ├─ MCP tools: dynamic from connected MCP servers
  │    │    └─ Skill tools: skill-specific tools from loaded skills
  │    │
  │    ├─ toolOrchestration: streaming execution + multi-tool parallel
  │    │    └─ StreamingToolExecutor: real-time output streaming
  │    │
  │    └─ Result → messages.append(tool_result) → iterate
  │
  └─ Final response → user
```

### 1.3 Sandbox 四层结构

Source: `analysis/Sandbox/` 相关文件

```
shouldUseSandbox()                    ← 决策层: 是否需要沙箱
       │
       ▼
convertToConfig()                     ← 配置层: 生成沙箱配置
       │
       ▼
bashPermissions                       ← 权限层: 细粒度命令权限
       │
       ▼
Shell.ts / execute / cleanup          ← 执行层: 实际执行与清理
```

---

## 二、原子化特征提取表

### 维度一: 通信与适配 (D1)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 多入口架构 | CLI + REPL + SDK + MCP + Bridge + Remote 六种入口 | 覆盖交互式/脚本化/IDE集成/远程调用全场景 | 不是单一 CLI 工具，而是多形态 agent 平台 | analysis/cli.tsx, analysis/main.tsx, analysis/MCP/, analysis/Remote/ |
| Ink/React TUI | `cli.tsx` 使用 React/Ink 组件渲染终端 UI | JSX 组件化的终端界面，现代化交互体验 | 与 hermes-agent 的 Ink TUI 同源技术栈 | analysis/cli.tsx |
| Slash Command 系统 | `commands.ts` 统一命令注册 | 用户通过 `/` 命令控制 agent 行为 | 推断: 类似 hermes-agent 的 slash command 注册中心 | analysis/commands.ts |
| MCP 完整集成 | `analysis/MCP/` 目录 | Agent 消费外部 MCP Server 工具，动态扩展能力 | 原生 MCP 支持，非第三方适配 | analysis/MCP/ |
| Bridge 协议 | `analysis/Bridge/` 或相关文件 | agent-to-agent 通信桥梁 | 推断: 用于跨 agent 实例通信 | analysis/Bridge/ (推断) |
| Remote 远程调用 | `analysis/Remote/` 目录 | 远程 agent 执行与结果回收 | 推断: 支持分布式 agent 部署 | analysis/Remote/ |

### 维度二: 执行深度 (D2)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 文件操作 | read_file / write_file / patch / search_files / glob (5+ tools) | 完整的代码文件 CRUD 能力 | `patch` 支持字符串级精确替换 (推断 9 种模糊匹配策略) | analysis/Tool.ts, analysis/tools/ |
| Shell 执行 | `bash` / `shell` tools, 通过 Shell.ts 管理 | 执行任意 shell 命令，支持超时/PID 控制 | 集成 sandbox 四层结构 | analysis/Shell.ts |
| 流式工具执行 | `StreamingToolExecutor` | 工具执行实时流式输出，LLM 可见中间结果 | 非一次性返回 — 支持流式消费工具输出 | analysis/StreamingToolExecutor |
| 并行工具调用 | `toolOrchestration` 模块 | 多工具并行执行再聚合 | 推断: 类似 hermes-agent 的 execute_code 脚本模式 | analysis/toolOrchestration |
| 查询引擎 | `QueryEngine.ts` — 核心 agent 循环 | while loop: LLM Request → Tool Call → Result → Iterate | 推断: 类似 hermes-agent 的 run_conversation() | analysis/query.ts, analysis/QueryEngine.ts |
| 代码编辑 | edit / replace / apply_patch tools | 手术级代码修改 (surgical edits) | 与 DeepAgents 的 `edit_file` 同类 | analysis/tools/ (推断) |

### 维度三: 任务编排 (D3)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| SubAgent 委派 | `task` tool — 派生隔离上下文的子 agent | 主 agent 委派复杂子任务给独立 agent | 上下文隔离，独立 tool set | analysis/SubAgent/ (推断) |
| Coordinator → Workers | 协调者-工作者模式 | 一个 coordinator 分配任务给多个 worker | 推断: 用于并行代码分析/测试 | analysis/Swarm/ 或相关 |
| Swarm Teammates | 群体协作模式 | 多个对等 agent 协作完成任务 | 与 coordinator 模式互补: 扁平化 vs 层级化 | analysis/Swarm/ |
| 三套多 Agent 共存 | SubAgent / Coordinator→Workers / Swarm 并行存在 | 同一平台支持三种协作拓扑 | 已分析产品中未见同时支持三种模式 | 社区分析 README |
| 上下文压缩 | `compact/` 模块 | Token 超限时自动压缩历史消息 | 推断: 类似 hermes 的 trajectory_compressor | analysis/compact/ |
| 任务追踪 | 推断: 内置 todo/write_todos 机制 | Agent 结构化追踪子任务进度 | 推断: 类似 DeepAgents 的 TodoListMiddleware | 推断 (基于同类产品模式) |

### 维度四: 安全隔离 (D4)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Sandbox 四层结构 | shouldUseSandbox → convertToConfig → bashPermissions → Shell.ts/cleanup | 分层决策-配置-权限-执行沙箱 | 四层递进设计，每层职责清晰 | 社区分析 README |
| Bash 权限细粒度 | `bashPermissions` 模块 | 细粒度控制哪些命令可在沙箱中执行 | 推断: 支持 allowlist/denylist | analysis/bashPermissions/ (推断) |
| Perm 权限层 | `Perm/` 目录 | 独立权限管理系统 | 推断: 工具级/操作级权限控制 | analysis/Perm/ |
| Sandbox 生命周期 | `cleanup` 阶段 | 沙箱执行完毕自动清理资源 | 推断: 容器/进程级清理 | analysis/Shell.ts cleanup |
| 命令审批 | 推断: 基于 Perm 层 | 危险命令需人工审批 | 推断: 类似 hermes-agent 的 approvals.mode | 推断 |
| 环境隔离 | `convertToConfig` + `shouldUseSandbox` | 代码执行与宿主环境隔离 | 推断: 可能支持容器化 (Docker) 隔离 | 推断 |

### 维度五: 记忆系统 (D5)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 分层记忆 | sessionStorage / memdir / SessionMemory 三层 | 从短期到长期的分层记忆架构 | 三层递进: 临时 → 会话 → 持久 | analysis/sessionStorage, analysis/memdir |
| SessionStorage | 会话级临时存储 | 单次对话内的状态保持 | 推断: 可能基于内存或临时文件 | analysis/sessionStorage/ |
| MemDir 持久化 | `memdir/` 目录 | 跨 session 的持久化记忆 | 推断: 类似 hermes 的 AGENTS.md/SOUL.md 文件注入 | analysis/memdir/ |
| SessionMemory | 会话记忆对象 | 结构化记忆表示 | 推断: 可能支持 key-value 或向量检索 | analysis/SessionMemory/ (推断) |
| Hooks 机制 | `hooks/` 目录 | 事件驱动的记忆触发 (pre/post tool execution) | 推断: 类似 lifecycle hooks，可注入记忆保存逻辑 | analysis/hooks/ |
| 上下文压缩 | `compact/` 模块 | Token 超限时自动压缩历史，保留关键信息 | 压缩后写入持久化存储 | analysis/compact/ |

### 维度六: 扩展生态 (D6)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| MCP 协议 | `analysis/MCP/` — 完整 MCP 客户端实现 | 动态发现和调用外部 MCP Server 工具 | 原生 MCP 集成，非第三方适配层 | analysis/MCP/ |
| Plugin 插件系统 | `analysis/Plugin/` 目录 | 可插拔的功能扩展 | 推断: 支持自定义插件注册 | analysis/Plugin/ |
| Skills 扩展 | `analysis/Skills/` 或相关 | 渐进式信息披露的 skill 加载 | 推断: 遵循 Anthropic Agent Skills 规范 | analysis/Skills/ (推断) |
| Remote 远程代理 | `analysis/Remote/` | 远程 agent 部署和调用 | 推断: 支持远程执行和结果回收 | analysis/Remote/ |
| Bridge 桥接 | `analysis/Bridge/` 或相关 | 跨 agent 实例桥接 | 推断: agent-to-agent 通信协议 | analysis/Bridge/ (推断) |
| SDK 入口 | SDK 模式入口 | 编程式调用 Claude Code 能力 | 推断: 类似 DeepAgents 的 Python SDK | analysis/SDK/ (推断) |
| Swarm 集群 | `analysis/Swarm/` | 多 agent 集群协作 | 推断: 分布式 agent 网络 | analysis/Swarm/ |

---

## 三、技术链路验证

### 3.1 正常路径: CLI 对话 → 多轮工具调用

基于架构图 + 社区分析 README 还原：

```
User types: "Refactor the auth module to use JWT instead of sessions"
  │
  ├─ Step 1: cli.tsx 接收用户输入 → launchRepl
  │    └─ React/Ink 组件渲染输入界面
  │
  ├─ Step 2: QueryEngine.process(message)
  │    ├─ Context assembly:
  │    │    ├─ System prompt: 从配置/memory 加载
  │    │    ├─ Tools: Tool.ts 注册的所有工具
  │    │    ├─ Memory: SessionMemory + memdir 加载上下文
  │    │    ├─ Skills: 匹配的 skills 注入 name+description
  │    │    └─ MCP tools: 从连接的外部 MCP server 动态发现
  │    │
  │    ├─ Compact check: 如果 token 超限 → compact/ 压缩
  │    │
  │    └─ LLM Request → 返回 tool_calls
  │
  ├─ Step 3: Tool Orchestration
  │    ├─ toolOrchestration 调度多个 tool calls
  │    │    ├─ search_files("auth") → 找到 auth 模块文件
  │    │    ├─ read_file("src/auth/index.ts") → 读取核心文件
  │    │    ├─ read_file("src/auth/session.ts") → 读取 session 实现
  │    │    └─ glob("src/auth/**/*.ts") → 发现所有相关文件
  │    │
  │    ├─ StreamingToolExecutor: 逐个流式返回结果
  │    │
  │    └─ Results → messages.append
  │
  ├─ Step 4: LLM 推理 → 生成代码变更
  │    ├─ write_file("src/auth/jwt.ts") → 创建 JWT 实现
  │    ├─ patch("src/auth/index.ts", diff) → 替换入口
  │    └─ bash("npm run test -- --auth") → 运行测试验证
  │         └─ Shell.ts → shouldUseSandbox? → 沙箱执行
  │
  └─ Step 5: Post-processing
       ├─ compact/: 压缩上下文 (如需要)
       ├─ SessionMemory: 保存关键信息
       └─ hooks/: 触发 post-execution hooks
```

### 3.2 异常路径: Sandbox 隔离执行

基于 Sandbox 四层结构还原：

```
Agent 决定执行用户提供的脚本
  │
  ├─ Step 1: shouldUseSandbox()
  │    ├─ 检查: 是否是用户代码？是否涉及网络/文件系统？
  │    └─ 决策: yes → 启用沙箱
  │
  ├─ Step 2: convertToConfig()
  │    ├─ 生成沙箱配置: 容器类型/资源限制/网络策略
  │    └─ 配置: 只读 /src, 可写 /tmp, 无网络
  │
  ├─ Step 3: bashPermissions 验证
  │    ├─ 检查命令: "node /tmp/user_script.js" ✓
  │    ├─ 拒绝命令: "curl evil.com" ✗ (网络被禁)
  │    └─ 拒绝命令: "rm -rf /src" ✗ (只读文件系统)
  │
  └─ Step 4: Shell.ts 执行 + cleanup
       ├─ 在沙箱中执行
       ├─ 超时: 120s 硬限制
       ├─ 结果: stdout/stderr 返回
       └─ cleanup: 销毁容器/清理临时文件
```

### 3.3 Multi-Agent 协作路径

```
User: "Audit the entire codebase for security vulnerabilities"
  │
  ├─ Coordinator Agent 收到任务
  │    │
  │    ├─ 分解任务: 按模块拆分 → 5 个子审计任务
  │    │
  │    ├─ Spawn Workers (Coordinator → Workers 模式):
  │    │    ├─ Worker 1: audit("src/auth/")
  │    │    │    └─ SubAgent: 独立上下文 + 只读工具集
  │    │    ├─ Worker 2: audit("src/api/")
  │    │    ├─ Worker 3: audit("src/db/")
  │    │    ├─ Worker 4: audit("src/frontend/")
  │    │    └─ Worker 5: audit("src/config/")
  │    │
  │    ├─ 并行执行 (toolOrchestration 支持)
  │    │
  │    └─ 结果聚合 → 综合审计报告
  │
  └─ 或使用 Swarm 模式:
       ├─ 5 个对等 agent 协作
       ├─ 共享发现 + 交叉验证
       └─ 共识审计报告
```

### 3.4 资源表现推断 (基于架构反推)

| 参数 | 推断默认值 | 来源/依据 |
|------|-----------|----------|
| 最大对话轮次 | ~100 (推断) | 同类产品 (hermes: 90, deepagents: 9999 recursion) |
| Shell 超时 | ~120s (推断) | 同类产品通用设置 |
| 子Agent 超时 | ~600s (推断) | 类似 hermes delegation.child_timeout_seconds |
| 压缩触发阈值 | ~85% 上下文窗口 (推断) | 同类产品 (DeepAgents: 0.85 fraction) |
| 并行工具上限 | ~5 (推断) | toolOrchestration 设计推断 |
| Sandbox 清理 | 执行后立即 (推断) | 四层结构 cleanup 阶段 |

---

## 四、Gap Analysis (差异分析)

### 4.1 核心风险与缺失项

| 缺失/风险项 | 检测来源 | 影响等级 | 详情 |
|--------|---------|---------|------|
| 闭源风险 — 非官方开源 | 事实 | **致命** | 所有分析基于 npm source map 泄露，非官方开源。Anthropic 可能随时修改/关闭/收费。无开源社区贡献路径，无 fork 自由 |
| 无公开 Benchmark | 社区分析 README | 高 | 无 SWE-bench / GAIA / HumanEval 等公开评测得分。能力声明完全依赖 Anthropic 官方营销 |
| 依赖 Anthropic API | 源码分析 (单模型) | 高 | 深度绑定 Anthropic Claude 模型。不支持 OpenAI/Llama/其他 provider 切换。api 中断 = 完全不可用 |
| 源码分析覆盖度有限 | 方法论限制 | 中 | 社区分析基于 source map 泄露的 TypeScript 源码 (1902 文件, 513k 行)，但分析仓库 (liuup/claude-code-analysis) 的 analysis/ 目录未必覆盖全部模块 |
| 无多模型 Transport | 源码分析 (单 provider) | 中 | 与 hermes-agent 的 4 种 Transport Adapter 形成对比 — Claude Code 仅支持 Anthropic API |
| 无法确定多租户能力 | 源码覆盖度不足 | 中 | 社区分析未涉及多租户/多用户隔离相关模块 |
| 安装路径受限 | 产品事实 | 低 | 通过 npm 安装, 需要 Node.js 环境。无 pip/brew/curl 一键安装 |

### 4.1.1 补充核查：与已知开源产品的 Gap 对比

| 能力维度 | Claude Code (泄露分析) | Hermes Agent (开源) | DeepAgents (开源) | Gap 判定 |
|---------|----------------------|-------------------|-----------------|---------|
| 多 Provider 支持 | ❌ 仅 Anthropic | ✅ 200+ 模型 (OpenRouter) | ✅ 20+ providers | Claude Code 锁定单生态 |
| 开源许可 | ❌ 闭源 | ✅ MIT | ✅ MIT | Claude Code 不可自托管 |
| Gateway 多平台 | ❌ 无 | ✅ 18+ 消息平台 | ❌ 无 | — |
| Skills 自我创建 | ❓ 未知 | ✅ 自动从经验创建 | ❌ 手动声明 | — |
| Sandbox 后端 | ✅ 四层结构 | ✅ 6 种 Backend | ✅ 5 种 Provider | 三者均强 |
| MCP 集成 | ✅ 原生 | ✅ 双向 MCP | ✅ MCP Client | Claude Code 可能是最深度集成 |
| Multi-Agent | ✅ 三套模式 | ✅ subagent + kanban | ✅ 三形态 subagent | Claude Code 可能最复杂 |
| 公开 Benchmark | ❌ 无 | ✅ batch_runner + mini_swe_runner | ✅ 108 evals × 7 类别 | DeepAgents 评测最成熟 |
| Browser 自动化 | ❓ 未知 | ✅ 12 browser tools | ❌ 无 | — |

### 4.2 基于架构反推的潜在问题

- **单点故障**: 依赖 Anthropic API — 任何 API 中断/限流/定价变更直接影响所有用户
- **Vendor Lock-in**: 深度绑定 Claude 模型的 prompt engineering/tool descriptions 可能无法迁移到其他模型
- **审计不透明**: 闭源意味着无法独立审计安全性和隐私合规性
- **社区贡献为零**: 无开源社区反馈循环，bug 修复和功能迭代完全由 Anthropic 内部控制
- **Swarm/Bridge/Remote 未验证**: 三套 multi-agent 模式在社区分析中是目录级发现，具体实现细节和可靠性未被充分分析

---

## 五、维度建议 (Dimension Evolution)

### 5.1 Agent vs Agent 平台判定

| 判定标准 | Claude Code 现状 | 判定 |
|---------|-----------------|------|
| **模板化** | ❓ 未知。Skills 可能支持 Markdown 声明。但 Agent 本身的配置方式不透明 (闭源限制) | 无法判定 |
| **隔离化** | ✅ 达成。Sandbox 四层结构提供强隔离；SubAgent 提供上下文隔离；Perm 层提供权限隔离 | 达成 |

> **结论**: 基于泄露源码分析，Claude Code 在**隔离化**维度达到 Agent 平台标准 (Sandbox + SubAgent + Perm)，但在**模板化**维度因闭源限制无法完整评估。从架构复杂度 (1902 文件) 和多入口设计判断，Claude Code 更接近一个**闭源 Agent 平台**而非简单 CLI 工具。

### 5.2 值得关注的架构特征 (供 CMA 参考)

| 特征 | 描述 | 参考价值 |
|------|------|---------|
| **Sandbox 四层结构** | shouldUseSandbox → convertToConfig → bashPermissions → Shell.ts/cleanup | 分层决策模型值得借鉴 — 决策/配置/权限/执行解耦 |
| **三套 Multi-Agent 并存** | SubAgent + Coordinator→Workers + Swarm 同时存在 | 不同协作拓扑适用于不同任务类型 — 层级化 vs 扁平化 |
| **多入口统一内核** | CLI/REPL/SDK/MCP/Bridge/Remote 共享同一个 QueryEngine | 入口多样化但内核统一 — 避免重复实现 |
| **Hooks 事件机制** | pre/post tool execution hooks | 生命周期钩子支持记忆保存/审计/权限检查等横切关注点 |
| **流式工具执行** | StreamingToolExecutor — 工具执行中实时可见中间输出 | 改善 LLM 可见性 — 不等工具完成即可开始下一步推理 |

---

## 六、依赖分析 (隐式能力推导)

基于 TypeScript 源码 + 架构反推 (社区分析未提供 package.json 细节):

| 推断依赖 | 推导能力 | 确定性 |
|---------|---------|--------|
| React + Ink | 终端 TUI 渲染 (JSX 组件化) | 高 (cli.tsx) |
| Anthropic Node SDK (`@anthropic-ai/sdk`) | Claude API 调用 | 高 |
| MCP SDK (`@modelcontextprotocol/sdk`) | MCP 协议客户端 | 高 |
| Node.js ≥18 | 运行时 (TypeScript → ESM/CJS) | 高 |
| ripgrep / grep 库 | 代码搜索 (search_files) | 中 |
| diff 库 | patch 生成和应用 | 中 |
| glob 库 | 文件模式匹配 | 中 |
| tree-sitter | 代码解析 (推断) | 低 |

---

## 七、独特性汇总

1. **Sandbox 四层结构**: shouldUseSandbox → convertToConfig → bashPermissions → Shell.ts/cleanup 的分层解耦设计在已知分析产品中最为精细。每一层有独立职责且可独立替换。

2. **三套 Multi-Agent 模式并存**: SubAgent (层级) + Coordinator→Workers (协调者) + Swarm (对等群体) 三种协作拓扑同时存在于同一平台，覆盖从简单委派到复杂群体协作的全谱场景。

3. **多入口统一内核**: CLI / REPL / SDK / MCP / Bridge / Remote 六种入口共享同一个 QueryEngine 内核，是入口多样性最高的分析产品。

4. **流式工具执行器 (StreamingToolExecutor)**: 工具执行过程中实时流式返回中间输出，让 LLM 不等工具完成即可开始推理下一步。在已知分析产品中未见同类机制。

5. **Hooks 生命周期机制**: pre/post tool execution hooks 为横切关注点 (审计/记忆/权限/日志) 提供了标准注入点。

6. **MemDir 持久化记忆**: 文件系统级的跨 session 记忆持久化 (memdir/), 与 sessionStorage (临时) + SessionMemory (结构化) 形成三层记忆体系。

7. **Bridge + Remote + Swarm 远程协作**: 完整的远程/分布式 agent 能力栈 — Bridge (点对点), Remote (远程调用), Swarm (群体协作)。

---

## 八、校准备忘录 (Calibration)

### 8.1 事实核查表

| 声明 | 验证状态 | 来源验证 |
|------|---------|---------|
| 源码语言: TypeScript | ✅ 准确 | 社区分析仓库 README |
| 文件数: 1902 | ✅ 准确 | 同上 |
| 代码行数: 513k | ✅ 准确 | 同上 |
| 分析仓库 Stars: 2.2k | ✅ 准确 | GitHub |
| Sandbox 四层结构 | ✅ 准确 | 社区分析 README 明确描述 |
| 三套 Multi-Agent | ✅ 准确 | 同上 |
| CLI/REPL/SDK/MCP/Bridge/Remote 多入口 | ✅ 准确 | 同上 |
| 非官方开源 | ✅ 准确 | 事实: 基于 npm source map 泄露 |

### 8.2 需修正/标注不确定项

| 声明 | 问题 | 修正/标注 |
|------|------|----------|
| Skills 系统存在 | 未在社区分析 README 中直接确认 | 标注为「推断」— 基于 Anthropic Agent Skills 规范和 MCP/Plugin 生态反推 |
| Browser 自动化 | 未在社区分析 README 中确认 | 标注为「未知」— 不在已知特征列表中 |
| 压缩触发阈值 | 无源码直接验证 | 标注为「推断」— 基于同类产品对比 |
| Hooks 内存保存 | 功能存在但细节未验证 | 标注为「推断」— hooks/ 目录存在但具体钩子列表未知 |
| 无多 Provider 支持 | 基于 Anthropic 产品策略推断 | 标注为「高确定性」— Claude Code 作为 Anthropic 产品深度绑定 Claude 模型是合理的商业策略 |

### 8.3 遗漏项

| 遗漏项 | 原因 |
|--------|------|
| 完整的 Tool 列表 (所有工具名称和参数) | 社区 analysis/ 目录覆盖度有限 |
| Plugin 系统具体 API | 同上 |
| Skills 加载机制细节 | 同上 |
| Swarm/Bridge/Remote 通信协议 | 同上 |
| 配置系统完整结构 (config 文件格式) | 同上 |
| package.json 完整依赖树 | 社区分析仓库未提供 |

### 8.4 总体准确性评分

- **社区分析 README 直接支撑**: ~40% (8/20 核心声明)
- **架构反推 (目录名+标准模式)**: ~35% (7/20)
- **同类产品对比推断**: ~15% (3/20)
- **合理推断 (标注)**: ~10% (2/20)
- **总体评分: 中等准确性** — 受限于社区分析的覆盖度，约 40% 的特征有直接来源支撑。Sandbox/多Agent/Session/MCP 等核心特征来源明确，但细粒度工具/配置/Plugin 等细节不足。

> ⚠️ **关键限制**: 本报告完全基于社区对泄露源码的分析，非 Anthropic 官方文档。所有发现均受限于 liuup/claude-code-analysis 仓库的 analysis/ 目录覆盖度。建议结合 Anthropic 官方文档和产品实际使用进行交叉验证。

---

## 附录: 文件索引 (基于社区分析仓库)

| 源文件 | 路径 (analysis/ 下) | 推断行数 |
|--------|------|------|
| CLI 入口 | `analysis/cli.tsx` | ~500+ |
| REPL 启动 | `analysis/main.tsx` | ~300+ |
| 初始化引导 | `analysis/init.ts` | ~200+ |
| 配置设置 | `analysis/setup.ts` | ~200+ |
| 命令注册 | `analysis/commands.ts` | ~300+ |
| 查询引擎 | `analysis/query.ts` | ~500+ |
| 查询引擎核心 | `analysis/QueryEngine.ts` | ~800+ |
| 工具定义 | `analysis/Tool.ts` | ~600+ |
| 工具编排 | `analysis/toolOrchestration` (目录或文件) | ~400+ |
| 流式工具执行器 | `analysis/StreamingToolExecutor` | ~300+ |
| Shell 执行 | `analysis/Shell.ts` | ~400+ |
| 权限系统 | `analysis/Perm/` (目录) | ~500+ |
| Bash 权限 | `analysis/bashPermissions/` (目录) | ~300+ |
| Sandbox | `analysis/Sandbox/` (目录) | ~600+ |
| 会话存储 | `analysis/sessionStorage/` (目录) | ~200+ |
| 持久记忆 | `analysis/memdir/` (目录) | ~200+ |
| 会话记忆 | `analysis/SessionMemory/` (目录) | ~200+ |
| Hooks 钩子 | `analysis/hooks/` (目录) | ~200+ |
| 上下文压缩 | `analysis/compact/` (目录) | ~300+ |
| MCP 集成 | `analysis/MCP/` (目录) | ~500+ |
| Plugin 系统 | `analysis/Plugin/` (目录) | ~300+ |
| Skills 扩展 | `analysis/Skills/` (目录) | ~300+ |
| Remote 远程 | `analysis/Remote/` (目录) | ~300+ |
| Bridge 桥接 | `analysis/Bridge/` (目录) | ~200+ |
| Swarm 集群 | `analysis/Swarm/` (目录) | ~400+ |
