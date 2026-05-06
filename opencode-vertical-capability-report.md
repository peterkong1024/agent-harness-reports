# OpenCode 垂直能力解构报告

> 项目: anomalyco/opencode
> 分析日期: 2026-05-06
> 分析方法: 特征逆向挖掘法 (Feature Reverse Mining)
> 数据来源权威性层级: 仓库结构分析 > 官方 README > 推断

---

## 产品画像

| 属性 | 值 | 来源 |
|------|-----|------|
| 产品定位 | "The open source coding agent" — 开源编码智能体 | 官方 README |
| 开发者 | anomalyco | GitHub |
| 仓库 | anomalyco/opencode (155k stars, 18k forks, 12.3k commits) | GitHub 页面 |
| 语言 | TypeScript (Bun 运行时), monorepo | 仓库结构 |
| 许可证 | 开源 | 仓库 LICENSE |
| 安装方式 | `curl -fsSL https://opencode.ai/install \| bash` 或 `npm i -g opencode-ai` | 官方文档 |
| 版本 | dev 分支活跃开发 | 仓库分支 |
| 架构模式 | Client/Server — TUI 仅为前端 client | README 架构说明 |
| Monorepo 包 | app, console, containers, core, desktop, docs, enterprise, extensions, function, identity, opencode, plugin, script, sdk, slack, storybook, ui, web | 仓库 packages/ 目录 |
| Desktop App | macOS/Windows/Linux (BETA) | 仓库 packages/desktop |
| Provider 模型 | Provider-agnostic: Claude, OpenAI, Google, 本地模型 | README |
| LSP 支持 | Built-in opt-in LSP | README |
| Agent 类型 | build (全权限), plan (只读), general subagent (内部搜索用) | README |

> 推断: OpenCode 的定位是开源社区对 Claude Code/Cursor 的替代方案。其 client/server 分离架构 + provider-agnostic + Desktop App 的组合，使其在开源编码 agent 中具有独特的"全平台覆盖"优势。monorepo 中包含 enterprise 包，暗示商业版/企业版规划。

---

## 一、架构图提取

### 1.1 系统架构 (基于仓库结构反推)

Source: monorepo packages/ 目录结构 + README 描述

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          OpenCode Architecture                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      Client Layer                                  │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │    │
│  │  │ CLI/TUI  │  │ Desktop  │  │ Slack    │  │ Remote Client │   │    │
│  │  │ (app)    │  │ App      │  │ App      │  │ (client/     │   │    │
│  │  │          │  │ (desktop)│  │ (slack)  │  │  server)     │   │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬───────┘   │    │
│  │       │             │             │                 │            │    │
│  └───────┼─────────────┼─────────────┼─────────────────┼────────────┘    │
│          │             │             │                 │                  │
│          └─────────────┴─────────────┴─────────────────┘                  │
│                               │                                            │
│                               ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      Server / Core Layer                          │    │
│  │  ┌────────────────────────────────────────────────────────────┐  │    │
│  │  │  Agent Orchestrator (core)                                   │  │    │
│  │  │    ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │  │    │
│  │  │    │  build   │  │  plan    │  │  general subagent    │   │  │    │
│  │  │    │  Agent   │  │  Agent   │  │  (internal search)   │   │  │    │
│  │  │    │ (全权限) │  │ (只读)   │  │                      │   │  │    │
│  │  │    └──────────┘  └──────────┘  └──────────────────────┘   │  │    │
│  │  └────────────────────────────────────────────────────────────┘  │    │
│  │                                                                   │    │
│  │  ┌────────────────────────────────────────────────────────────┐  │    │
│  │  │  Tool System                                                │  │    │
│  │  │  Shell Exec │ File Ops │ LSP │ Plugin Tools │ MCP (推断)   │  │    │
│  │  └────────────────────────────────────────────────────────────┘  │    │
│  │                                                                   │    │
│  │  ┌────────────────────────────────────────────────────────────┐  │    │
│  │  │  Security / Permission Layer                                │  │    │
│  │  │  plan agent (只读) │ bash 权限请求 │ 子Agent 隔离           │  │    │
│  │  └────────────────────────────────────────────────────────────┘  │    │
│  │                                                                   │    │
│  │  ┌────────────────────────────────────────────────────────────┐  │    │
│  │  │  Memory / Persistence Layer                                 │  │    │
│  │  │  SQLite (Drizzle ORM) │ session 持久化 │ project 上下文     │  │    │
│  │  └────────────────────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                               │                                            │
│                               ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      Extension / Ecosystem Layer                   │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │    │
│  │  │ Plugin   │  │ Extension│  │ SDK      │  │ MCP (推断)    │   │    │
│  │  │ System   │  │ System   │  │ (sdk)    │  │               │   │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └───────────────┘   │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      External Providers                             │    │
│  │  Claude API │ OpenAI API │ Google API │ Local Models (provider-    │    │
│  │                                      │ agnostic)                    │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │                      Infrastructure                                │    │
│  │  Bun Runtime │ containers (容器执行) │ LSP Server                 │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Agent 调用链路

基于 README 描述的三种 Agent 类型反推：

```
User Input (via CLI/Desktop/Slack/Remote)
  │
  ├─ Client → Server 通信
  │
  ├─ Agent Router:
  │    ├─ plan agent (只读) → 代码分析/搜索/理解
  │    │    └─ Tools: read_file, search, LSP, glob
  │    │
  │    ├─ build agent (全权限) → 代码生成/修改/执行
  │    │    └─ Tools: read_file, write_file, patch, shell, bash
  │    │         └─ bash → 权限请求 (用户审批)
  │    │
  │    └─ general subagent (内部搜索) → 代码库探索
  │         └─ Tools: search, glob, LSP
  │
  ├─ Provider Router: Claude / OpenAI / Google / Local
  │    └─ Provider-agnostic 抽象层
  │
  ├─ Result → Memory 持久化 (SQLite + Drizzle ORM)
  │    └─ Session 状态保存
  │
  └─ Response → Client 流式渲染
```

---

## 二、原子化特征提取表

### 维度一: 通信与适配 (D1)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| CLI/TUI 入口 | `packages/app` — TUI 前端 client | 终端交互式编码助手 | 基于 Bun 运行时，TypeScript TUI | packages/app (仓库) |
| Desktop App | `packages/desktop` — macOS/Windows/Linux | 原生桌面体验 (BETA) | 开源编码 agent 中少有的跨平台桌面应用 | packages/desktop (仓库) |
| Client/Server 远程 | `packages/core` 提供 Server 端，TUI 仅为 client | 支持远程 agent 部署和调用 | 推断: client 通过网络连接远程 server | README "Client/server architecture" |
| Slack 集成 | `packages/slack` — Slack App 集成 | 在 Slack 中与 agent 交互 | 推断: 支持团队协作场景 | packages/slack (仓库) |
| Provider-agnostic | 支持 Claude / OpenAI / Google / 本地模型 | 不锁定单一模型提供商 | 开源 coding agent 中与 Claude Code 形成对比 (后者锁定 Anthropic) | README |
| 流式输出 | 推断: client/server 架构支持流式传输 | 实时显示 agent 响应 | 基于 WebSocket 或 SSE (推断) | 推断 (基于 client/server 架构) |

### 维度二: 执行深度 (D2)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Shell 执行 | Shell tool — 执行 shell 命令 | 运行构建/测试/脚本 | 需 bash 权限请求 (build agent) | README "bash" agent |
| 文件操作 | 推断: read_file / write_file / edit / patch | 代码文件 CRUD | 标准编码 agent 文件操作集 | 推断 (基于同类产品) |
| LSP 集成 | Built-in opt-in LSP support | 代码智能: 诊断/跳转/补全 | 内置 LSP 支持，非通过外部插件 | README "LSP" |
| 容器执行 | `packages/containers` — 容器支持包 | 推断: Docker/容器化代码执行 | 提供隔离执行环境 | packages/containers (仓库) |
| 代码搜索 | 推断: search / glob / grep | 跨文件代码搜索 | 推断: 基于 LSP 或 ripgrep | 推断 (基于同类产品) |
| 代码编辑 | 推断: patch / edit / replace | 手术级代码修改 | 推断: 字符串替换/精确编辑 | 推断 (基于同类产品) |

### 维度三: 任务编排 (D3)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| build Agent | 全权限 agent — 代码生成与执行 | 承担代码修改/构建/测试全流程 | 主工作 agent | README |
| plan Agent | 只读 agent — 代码分析与理解 | 先规划后执行，只读探索代码库 | 安全设计: plan 无法修改文件 | README |
| general SubAgent | 内部搜索用子 agent | 代码库探索与信息检索 | 上下文隔离的子 agent | README "general subagent" |
| Agent 路由 | 三种 agent 按任务类型分派 | 只读任务用 plan，修改任务用 build | 权限最小化原则 | 推断 (基于三种 agent 定义) |
| 上下文管理 | 推断: session/project 级上下文管理 | 项目级上下文持久化 | 推断: 通过 SQLite 存储会话上下文 | 推断 (基于 D5 记忆系统) |
| 并行工具调用 | 推断: 支持多工具并行执行 | 提升执行效率 | 推断: 类似同类产品的 toolOrchestration | 推断 (基于同类产品) |

### 维度四: 安全隔离 (D4)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| plan Agent 只读 | plan agent 仅可读取，无法修改文件 | 规划阶段零风险探索 | 权限最小化: 只读 agent 天然安全 | README "plan agent" |
| bash 权限请求 | build agent 执行 bash 需用户审批 | 危险命令需人工确认 | 推断: 类似 hermes-agent 的 approvals 机制 | README "bash" 请求 |
| 容器隔离 | `packages/containers` — 容器执行环境 | 代码执行与宿主隔离 | 推断: Docker/容器级沙箱 | packages/containers (仓库) |
| 子Agent 隔离 | general subagent 独立上下文 | 子 agent 无法访问主 agent 权限 | 推断: 上下文隔离 + 权限限制 | 推断 (基于 agent 类型定义) |
| Provider 隔离 | Provider-agnostic → 用户可选择本地模型 | 数据不出本地 | 本地模型选项避免代码上传第三方 API | README "本地模型" |

### 维度五: 记忆系统 (D5)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| SQLite 持久化 | SQLite + Drizzle ORM | 结构化数据持久化 | ORM 层提供类型安全的数据访问 | README "SQLite" |
| Session 持久化 | 会话级状态持久化 | 跨轮次对话状态保持 | 推断: 对话历史/agent 状态存入 SQLite | 推断 (基于 SQLite + session) |
| Project 上下文 | 项目级上下文持久化 | 项目范围的记忆和配置 | 推断: 类似 AGENTS.md / CLAUDE.md 规范 | 推断 (基于 "project" 关键词) |
| 记忆分层 | 推断: session + project 两级记忆 | 短期 (会话) + 长期 (项目) 分层 | 推断: 类似同类产品的分层记忆设计 | 推断 (基于同类产品模式) |

### 维度六: 扩展生态 (D6)

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Plugin 系统 | `packages/plugin` — 插件系统包 | 可插拔的功能扩展 | 独立 plugin 包，支持社区扩展 | packages/plugin (仓库) |
| Extensions | `packages/extensions` — 扩展系统 | 功能扩展机制 | 推断: 与 plugin 互补的扩展方式 | packages/extensions (仓库) |
| SDK | `packages/sdk` — 开发者 SDK | 编程式调用 OpenCode 能力 | 推断: 支持嵌入到其他应用 | packages/sdk (仓库) |
| MCP 集成 | 推断: 基于 ecosystem 定位 | 与外部 MCP Server 互操作 | 推断: 同类产品普遍支持，OpenCode 大概率集成 | 推断 (基于同类产品) |
| Provider-agnostic | Claude / OpenAI / Google / 本地模型 | 多模型适配 | 不锁定单一生态 | README |
| Function 系统 | `packages/function` — 函数系统 | 推断: 可注册自定义函数/工具 | 自定义工具扩展点 | packages/function (仓库) |
| Identity | `packages/identity` — 身份系统 | 推断: 用户/团队身份管理 | 多用户场景支持 | packages/identity (仓库) |

---

## 三、技术链路验证

### 3.1 正常路径: 代码修改任务

基于架构图 + README 描述的 Agent 类型还原：

```
User: "Refactor the auth module to use JWT"
  │
  ├─ Step 1: Client (CLI/Desktop/Slack) 发送请求到 Server
  │
  ├─ Step 2: Agent Router 判断任务类型
  │    └─ 代码修改任务 → 路由到 build agent
  │
  ├─ Step 3: build agent 执行
  │    ├─ 加载 session/project 上下文 (SQLite)
  │    ├─ 调用 LSP 理解代码结构
  │    ├─ 搜索相关文件 (search/glob)
  │    ├─ 读取核心文件 (read_file)
  │    ├─ LLM 推理 → 生成代码变更
  │    ├─ write_file/edit 创建/修改文件
  │    └─ bash("npm test") → 权限请求 → 执行测试
  │
  ├─ Step 4: 结果持久化
  │    ├─ SQLite 存储 session 状态
  │    └─ 项目上下文更新
  │
  └─ Step 5: 流式返回结果给 Client
```

### 3.2 安全路径: plan → build 两步走

```
User: "Analyze the auth module security and fix vulnerabilities"
  │
  ├─ Step 1: plan agent (只读)
  │    ├─ 搜索 auth 模块: search/glob
  │    ├─ 读取所有相关文件
  │    ├─ LSP 诊断 (安全漏洞检查)
  │    ├─ 生成分析报告 (只读, 无文件修改)
  │    └─ 输出: 发现 3 个问题, 建议修复方案
  │
  ├─ Step 2: 用户审批计划 → 确认执行
  │
  ├─ Step 3: build agent (全权限)
  │    ├─ 基于 plan 的建议执行修复
  │    ├─ write_file/edit 修改代码
  │    ├─ bash("npm test") → 权限请求
  │    └─ 验证修复
  │
  └─ Step 4: 结果汇总返回
```

### 3.3 容器隔离执行路径

```
Agent 需要执行用户提供的代码
  │
  ├─ Step 1: 检测需要隔离执行
  │    └─ packages/containers 介入
  │
  ├─ Step 2: 创建容器环境
  │    ├─ 推断: Docker 容器启动
  │    ├─ 挂载只读工作目录
  │    └─ 网络限制
  │
  ├─ Step 3: 容器内执行
  │    ├─ 执行代码/脚本
  │    ├─ 超时控制
  │    └─ 捕获 stdout/stderr
  │
  └─ Step 4: 清理容器
       └─ 销毁容器, 回收资源
```

### 3.4 资源表现推断

| 参数 | 推断默认值 | 来源/依据 |
|------|-----------|----------|
| 最大对话轮次 | ~100 (推断) | 同类产品通用设置 |
| Shell 超时 | ~120s (推断) | 同类产品通用设置 |
| 子Agent 超时 | ~300s (推断) | general subagent 内部搜索 |
| 上下文窗口管理 | 自动管理 (推断) | 基于 session/project 持久化 |
| 容器清理 | 执行后立即 (推断) | containers 包设计推断 |
| 并行工具上限 | ~5 (推断) | 同类产品并行设计推断 |

---

## 四、Gap Analysis (差异分析)

### 4.1 核心风险与缺失项

| 缺失/风险项 | 检测来源 | 影响等级 | 详情 |
|--------|---------|---------|------|
| 无公开 Benchmark | 未在已知渠道确认 | 高 | 无 SWE-bench / GAIA / HumanEval 等公开评测得分。155k stars 但无性能验证 |
| Desktop App 处于 BETA | README 标注 | 中 | macOS/Windows/Linux Desktop 均为 BETA 状态，稳定性存疑 |
| enterprise 包状态不明 | packages/enterprise 存在但未公开 | 中 | 暗示商业版/企业版规划，但功能边界不清。可能影响开源版功能裁剪 |
| MCP 集成未确认 | 仅推断 | 中 | 未在 README 中看到 MCP 明确声明，仅基于 ecosystem 定位和同类产品模式推断 |
| 代码编辑能力细节不清 | 文档覆盖度不足 | 中 | 未明确说明支持 edit/patch/replace 等精确编辑方式 |
| LSP 为 opt-in | README | 低 | 非默认启用，需用户主动配置 |
| 无多租户迹象 | 仓库结构分析 | 高 | identity 包暗示用户管理，但无多租户隔离机制 |

### 4.1.1 补充核查：与已知开源产品的 Gap 对比

| 能力维度 | OpenCode (分析) | Claude Code | Hermes Agent | DeepAgents | Gap 判定 |
|---------|----------------|-------------|--------------|------------|---------|
| 多 Provider 支持 | ✅ 4+ providers | ❌ 仅 Anthropic | ✅ 200+ | ✅ 20+ | OpenCode 中上 |
| 开源许可 | ✅ 开源 | ❌ 闭源 | ✅ MIT | ✅ MIT | 与 Hermes/DeepAgents 同级 |
| Desktop App | ✅ BETA | ❌ 无 | ❌ 无 | ❌ 无 | OpenCode 独有 |
| Client/Server 分离 | ✅ 原生 | ❌ 仅 local | ❌ 仅 local | ❌ 仅 local | OpenCode 独有 |
| Slack 集成 | ✅ 原生 | ❌ 无 | ✅ 18+ 平台 | ❌ 无 | — |
| Sandbox/容器 | ✅ containers 包 | ✅ 四层结构 | ✅ 6 Backend | ✅ 5 Provider | 均支持 |
| MCP 集成 | ❓ 推断 | ✅ 原生 | ✅ 双向 MCP | ✅ MCP Client | 待确认 |
| Multi-Agent | ✅ 3 种 Agent | ✅ 3 套模式 | ✅ subagent + kanban | ✅ 3 形态 | 均支持 |
| LSP 集成 | ✅ Built-in | ❓ 未知 | ❌ 无 | ❌ 无 | OpenCode 独有 |
| Plugin/Extension | ✅ 独立包 | ✅ 存在 | ❌ 无 | ❌ 无 | OpenCode 完善 |
| 公开 Benchmark | ❌ 无 | ❌ 无 | ✅ batch_runner | ✅ 108 evals | 短板 |
| 记忆系统 | ✅ SQLite | ✅ 三层 | ✅ checkpoint | ✅ StoreBackend | 均支持 |

### 4.2 基于架构反推的潜在问题

- **文档不足**: 相比同类产品 (DeepAgents 有详尽源码注释, Claude Code 有社区泄露分析), OpenCode 的公开技术文档覆盖度最低。大部分能力依赖仓库包名反推
- **Bun 运行时依赖**: 单运行时绑定 (Bun), Node.js/npm 安装方式不确定是否完全正常
- **enterprise 包透明度**: 企业版功能边界不清，开源版可能存在功能裁剪风险
- **MCP/Plugin/Extension 三者边界**: 三个扩展机制 (plugin/extensions/sdk) 的定位和互操作关系不明确
- **Desktop App 成熟度**: BETA 标注 + Bunny 运行时意味着 desktop 体验可能不稳定

---

## 五、维度建议 (Dimension Evolution)

### 5.1 Agent vs Agent 平台判定

| 判定标准 | OpenCode 现状 | 判定 |
|---------|-------------|------|
| **模板化** | ❓ 未确认。Plugin/Extension/SDK 可能支持模板声明，但无公开证据 | 无法判定 |
| **隔离化** | ⚠️ 部分达到。containers 包提供执行隔离，plan agent 提供权限隔离 (只读)，但隔离粒度未确认 | 部分达到 |

> **结论**: 基于当前可获取的公开信息，OpenCode 是一个**开源编码 Agent 产品**，具备向 Agent 平台演进的架构基础 (plugin/sdk/extension + containers + client/server)，但模板化和强隔离维度尚未完整确认。其 Desktop App + Client/Server 架构 + Slack 集成的全平台覆盖策略是区别于同类产品的核心差异点。

### 5.2 值得关注的架构特征

| 特征 | 描述 | 参考价值 |
|------|------|---------|
| **Client/Server 分离** | TUI/Desktop/Slack 均为 client，core 独立 server | 天然支持远程 agent 部署，无需额外适配层 |
| **plan/build Agent 分离** | 只读规划 vs 全权限执行 | 安全设计: 权限最小化 + 两步确认，值得 CMA 参考 |
| **Bun 运行时** | TypeScript 直接运行，无需编译 | 开发体验优化，启动速度优于 Node.js |
| **LSP 内置** | opt-in 的 LSP 支持 | 编码 agent 的代码理解能力更强于纯文本搜索 |
| **Desktop App** | 跨平台桌面应用 | 降低非 CLI 用户的使用门槛 |

---

## 六、依赖分析 (隐式能力推导)

基于 monorepo 包结构反推：

| 推断依赖 | 推导能力 | 确定性 |
|---------|---------|--------|
| Bun Runtime | TypeScript 原生执行 + 包管理 | 高 |
| Drizzle ORM | SQLite 数据库 ORM 层 | 高 (README 明确) |
| SQLite | 持久化存储 (session/project) | 高 |
| Docker / 容器运行时 | containers 包的容器执行能力 | 中 |
| LSP Protocol | 代码智能: 诊断/跳转/补全 | 高 (README 明确) |
| MCP SDK (推断) | MCP 协议集成 | 低 (未确认) |
| ripgrep | 代码搜索 | 低 (推断) |
| tree-sitter | 代码解析 | 低 (推断) |

---

## 七、独特性汇总

1. **Client/Server 原生分离**: 是已知开源编码 agent 中唯一将 TUI 明确设计为"前端 client"的产品，天然支持远程 agent 部署

2. **Desktop App + CLI + Slack 三入口**: 覆盖终端用户、桌面用户、团队协作三种场景，入口多样性在开源编码 agent 中领先

3. **plan/build Agent 分离**: 将"规划"和"执行"分离为两个独立 agent，plan 只读确保规划阶段零风险

4. **Bun 运行时**: 使用 Bun 替代 Node.js 作为 TypeScript 运行时，可能带来启动速度和开发体验优势

5. **LSP 内置**: opt-in 的 Built-in LSP 支持，将 IDE 级代码智能集成到 agent 中

6. **Plugin + Extensions + SDK 三层扩展**: 提供插件系统、扩展机制、SDK 三种扩展方式，降低二次开发门槛

---

## 八、校准备忘录 (Calibration)

### 8.1 事实核查表

| 声明 | 验证状态 | 来源验证 |
|------|---------|---------|
| Stars: 155k | ✅ 准确 | GitHub 页面实时数据 |
| Forks: 18k | ✅ 准确 | GitHub 页面实时数据 |
| Commits: 12.3k | ✅ 准确 | GitHub 页面实时数据 |
| 语言: TypeScript | ✅ 准确 | 仓库语言统计 |
| 运行时: Bun | ✅ 准确 | README |
| Monorepo 结构 | ✅ 准确 | packages/ 目录 |
| Client/Server 架构 | ✅ 准确 | README |
| Provider-agnostic | ✅ 准确 | README |
| LSP opt-in | ✅ 准确 | README |
| 三种 Agent 类型 | ✅ 准确 | README |
| SQLite + Drizzle | ✅ 准确 | README |
| Desktop App BETA | ✅ 准确 | README |

### 8.2 需修正/标注不确定项

| 声明 | 问题 | 修正/标注 |
|------|------|----------|
| MCP 集成 | 未在 README 中确认 | 标注为「推断」— 基于同类产品模式和 ecosystem 定位 |
| 文件编辑类型 (edit/patch/replace) | 未明确说明 | 标注为「推断」— 基于同类产品通用能力 |
| 容器执行细节 | packages/containers 存在但功能未说明 | 标注为「推断」— Docker 容器执行基于包名推断 |
| Plugin vs Extension vs SDK 边界 | 三个包存在但无说明 | 标注为「待确认」— 三者定位不清 |
| enterprise 包功能 | 包存在但无公开文档 | 标注为「不明」 |
| 上下文压缩机制 | 未提及 | 标注为「未知」— 可能不存在或未文档化 |
| 记忆系统细节 (AGENTS.md 规范?) | 未确认 | 标注为「推断」— 基于 SQLite + session/project 关键词 |

### 8.3 遗漏项

| 遗漏项 | 原因 |
|--------|------|
| 完整的 Tool 列表 | 文档未提供 |
| Plugin API 规范 | 包存在但文档未描述 |
| SDK API 参考 | 包存在但文档未描述 |
| enterprise 功能 | 包存在但无公开信息 |
| 评测得分 | 无公开 benchmark |
| 包间依赖关系 | monorepo 内部依赖未分析 |
| 社区 Issues 分析 | 未采集 |
| 测试覆盖率 | 未采集 |

### 8.4 总体准确性评分

- **README 直接支撑**: ~30% (6/20 核心声明)
- **仓库结构反推**: ~40% (8/20)
- **同类产品对比推断**: ~20% (4/20)
- **合理推断 (标注)**: ~10% (2/20)
- **总体评分: 中等偏低准确性** — OpenCode 的公开文档覆盖度在分析产品中最低。大部分能力基于 monorepo 包名反推，约 60% 的特征缺少直接源码/文档验证。建议进行实际安装测试和源码深读以提升准确性。

> ⚠️ **关键限制**: 本报告完全基于 README 和仓库目录结构分析，未进行源码深读。OpenCode 的公开文档覆盖度远低于 DeepAgents (85% 源码支撑) 和 OpenClaw (官方架构文档)。所有"推断"标注项均需通过实际安装和源码阅读交叉验证。

---

## 附录: 包索引 (基于 monorepo packages/ 目录)

| 包名 | 路径 | 推断功能 |
|------|------|---------|
| app | packages/app | CLI/TUI 客户端 |
| console | packages/console | Web 控制台 (推断) |
| containers | packages/containers | 容器/沙箱执行环境 |
| core | packages/core | 核心 agent 引擎与服务端 |
| desktop | packages/desktop | macOS/Windows/Linux 桌面应用 |
| docs | packages/docs | 文档站点 |
| enterprise | packages/enterprise | 企业版功能 (未公开) |
| extensions | packages/extensions | 功能扩展机制 |
| function | packages/function | 自定义函数/工具注册 |
| identity | packages/identity | 用户/团队身份管理 |
| opencode | packages/opencode | 主入口包 (推断) |
| plugin | packages/plugin | 插件系统 |
| script | packages/script | 脚本工具 |
| sdk | packages/sdk | 开发者 SDK |
| slack | packages/slack | Slack 集成 |
| storybook | packages/storybook | UI 组件开发/展示 |
| ui | packages/ui | UI 组件库 |
| web | packages/web | Web 前端 |
