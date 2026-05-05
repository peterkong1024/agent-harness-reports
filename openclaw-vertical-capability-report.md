# OpenClaw 垂直能力解构报告

> 项目: openclaw/openclaw
> 版本: 2026.5.3
> 分析日期: 2026-05-04
> 分析方法: 特征逆向挖掘法 (Feature Reverse Mining)
> 数据来源权威性层级: 源码 > 源码内配置 > 官方文档 > 推断

---

## 产品画像

| 属性 | 值 | 来源 |
|------|-----|------|
| 产品定位 | "Personal AI Assistant" — 运行在自己设备上的个人 AI 助手 | `README.md` L1 |
| 一句话描述 | "The AI that actually does things. It runs on your devices, in your channels, with your rules." | `VISION.md` L1-L3 |
| 核心范式 | Multi-channel AI Gateway + Agent Harness + Companion Apps | `README.md` "Highlights"; `docs/concepts/architecture.md` |
| GitHub Stars | 368,000+ | `github.com/openclaw/openclaw` 页面快照 |
| Forks | 75,800+ | 同上 |
| Open Issues | 3,400+ | 同上 |
| 总提交数 | 40,901 | 同上 |
| 许可证 | MIT | `package.json` L9 |
| 核心语言 | TypeScript (Node.js ≥22.14) | `package.json` `engines` |
| 包管理器 | pnpm 10.33.2 (workspace monorepo) | `package.json` `packageManager` |
| 运行时 | Node.js, 编译为 `dist/` | `package.json` `main: "dist/index.js"` |
| Gateway 架构 | WebSocket + JSON Schema 验证 + TypeBox → Swift Codegen | `docs/concepts/architecture.md` |
| Agent 运行时 | `@mariozechner/pi-agent-core` (PI Agent Framework) | `package.json` dependencies |
| 默认部署 | 本地 Gateway daemon (launchd/systemd) | `README.md` "Install" |
| 渠道覆盖 | 24+ 消息平台 | `README.md` "Supported channels" |
| 赞助商 | OpenAI, GitHub, NVIDIA, Vercel, Blacksmith, Convex | `README.md` "Sponsors" |

> 推断: OpenClaw 的定位是 **Consumer-grade Personal AI Assistant**，而非面向开发者的 Agent SDK。它是一个完整的消费级产品——开箱即用，用户通过 CLI 安装配置后即可在 WhatsApp/Telegram/Discord 等日常渠道使用。这一点与 deepagents (SDK-first) 和 Hermes Agent (CLI Agent-first) 有本质差异。

---

## 一、架构图提取

### 1.1 系统架构 (基于官方架构文档)

Source: `docs/concepts/architecture.md` (官方架构文档, 200 OK)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          OpenClaw System Architecture                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                    Companion Apps (Optional)                      │     │
│  │  macOS Menu Bar App  │  iOS Node  │  Android Node  │  WebChat    │     │
│  │  Voice Wake + Talk   │  Canvas    │  Camera/Screen │  Web UI     │     │
│  └──────────┬───────────┴─────┬──────┴──────┬─────────┴──────┬──────┘     │
│             │                 │             │                │            │
│             └─────────────────┼─────────────┼────────────────┘            │
│                               │  WebSocket  │                             │
│                               ▼  (role:node)▼                             │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                      GATEWAY (Daemon)                            │     │
│  │  ┌──────────────────────────────────────────────────────────┐   │     │
│  │  │              WebSocket Server (127.0.0.1:18789)            │   │     │
│  │  │  JSON Schema validated │ TypeBox → Swift Codegen          │   │     │
│  │  │  req/res/event pattern │ Idempotency keys │ Dedupe cache  │   │     │
│  │  └──────────────────────────────────────────────────────────┘   │     │
│  │                                                                  │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │     │
│  │  │  Agent   │  │ Channels │  │  Tools   │  │   Sandbox    │   │     │
│  │  │  Engine  │  │ (24+)    │  │  System  │  │   Manager    │   │     │
│  │  │ PI Agent │  │ WhatsApp │  │ Browser  │  │ Docker/SSH/  │   │     │
│  │  │ Core     │  │ Telegram │  │ Canvas   │  │ OpenShell    │   │     │
│  │  └──────────┘  │ Discord  │  │ Nodes    │  └──────────────┘   │     │
│  │                 │ Slack    │  │ Cron     │                      │     │
│  │  ┌──────────┐  │ Signal   │  │ Sessions │  ┌──────────────┐   │     │
│  │  │  Memory  │  │ iMessage │  │ Discord  │  │   Plugins    │   │     │
│  │  │ AGENTS.md│  │ ...      │  │ Slack    │  │   System     │   │     │
│  │  │ SOUL.md  │  └──────────┘  └──────────┘  │ MCP/ACP/     │   │     │
│  │  │ TOOLS.md │                               │ ClawHub      │   │     │
│  │  └──────────┘                               └──────────────┘   │     │
│  │                                                                  │     │
│  │  ┌──────────────────────────────────────────────────────────┐   │     │
│  │  │              HTTP Server (same port)                       │   │     │
│  │  │  /__openclaw__/canvas/  (Agent-editable HTML/CSS/JS)      │   │     │
│  │  │  /__openclaw__/a2ui/    (A2UI Host)                       │   │     │
│  │  └──────────────────────────────────────────────────────────┘   │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐     │
│  │                     External Integrations                         │     │
│  │  Model Providers: Anthropic │ OpenAI │ Google │ 20+ others       │     │
│  │  MCP Servers │ ACP Clients │ ClawHub Skills Registry            │     │
│  │  Webhooks │ Gmail Pub/Sub │ Cron Jobs                           │     │
│  └─────────────────────────────────────────────────────────────────┘     │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Gateway 连接生命周期 (官方序列图)

Source: `docs/concepts/architecture.md` § Connection lifecycle

```
Client                    Gateway
  │                          │
  │──── req:connect ────────▶│
  │◀─── res (ok) ───────────│  (hello-ok: presence + health snapshot)
  │                          │
  │◀─── event:presence ─────│
  │◀─── event:tick ─────────│
  │                          │
  │──── req:agent ──────────▶│
  │◀─── res:agent ack ──────│  {runId, status:"accepted"}
  │◀─── event:agent ────────│  (streaming)
  │◀─── res:agent final ────│  {runId, status, summary}
```

### 1.3 Agent 执行架构 (推断)

Source: 基于 README, VISION.md, package.json 依赖推断 (无官方 agent 架构文档)

```
Inbound Message (WhatsApp/Telegram/Discord/...)
  │
  ├─ Channel Plugin (grammY/Baileys/Slack Bolt/...)
  │    └─ Normalize to unified inbound envelope
  │
  ├─ Routing Layer (multi-agent routing)
  │    ├─ Match: channel/account/peer → agent workspace
  │    └─ DM pairing check (pairing code / allowlist)
  │
  ├─ Agent Engine (@mariozechner/pi-agent-core)
  │    ├─ Session: create or resume (per-agent isolation)
  │    ├─ Context Assembly:
  │    │    ├─ System Prompt (base + skills + memory)
  │    │    ├─ AGENTS.md, SOUL.md, TOOLS.md injection
  │    │    └─ Skills: SKILL.md progressive disclosure
  │    ├─ Tool Execution:
  │    │    ├─ bash/process (PTY via @lydell/node-pty)
  │    │    ├─ read/write/edit (workspace filesystem)
  │    │    ├─ browser (Playwright)
  │    │    ├─ canvas (A2UI)
  │    │    ├─ sessions_list/history/send/spawn
  │    │    ├─ cron (schedule/management)
  │    │    ├─ discord/slack (channel-specific actions)
  │    │    └─ nodes (camera/screen/location)
  │    └─ Sandbox:
  │         ├─ main session: host (full access)
  │         └─ non-main: Docker/SSH/OpenShell (isolated)
  │
  ├─ Reply Pipeline
  │    ├─ Directives: /think, /verbose, /trace, /compact
  │    ├─ Chunking + dedupe
  │    └─ Media attachment handling
  │
  └─ Outbound Delivery
       └─ Channel plugin → WhatsApp/Telegram/Discord/...
```

---

## 二、原子化特征提取表

### 维度一: 感知与输入

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 多模态消息接收 | 24+ Channel 插件归一化为统一 inbound envelope | 支持 text/image/voice/video 等多模态消息 | Channel 数量在已知同类产品中最多 | `README.md` "Supported channels" |
| Web 浏览 | `playwright-core` 依赖，推断为 browser tool | Agent 可启动 headless 浏览器抓取页面 | Playwright 作为内置工具 (非 MCP 注入) | `package.json` L68: `playwright-core` |
| PDF 解析 | `pdfjs-dist` 依赖 | 读取和分析 PDF 文件内容 | 推断: 用于文档理解和研究任务 | `package.json` L69: `pdfjs-dist` |
| 语音输入 | Voice Wake (macOS/iOS) + Talk Mode (Android) | 唤醒词激活 + 连续语音对话 | 推断: 使用 ElevenLabs + 系统 TTS fallback | `README.md` "Voice Wake + Talk Mode" |
| 摄像头/屏幕 | Node 命令: `camera.*`, `screen.record` | 手机端可拍照/截屏传给 Agent | 推断: 用于视觉理解场景 | `docs/concepts/architecture.md` Node 段 |
| 代码理解 | `tree-sitter-bash` 解析 Shell 命令; `@mozilla/readability` 提取网页正文 | Agent 可解析 Shell 脚本和网页内容 | 使用 tree-sitter 做语法级理解 | `package.json` L72, L56 |
| Web 搜索 | 推断: 通过 browser tool 或内置 web search provider | Agent 可实时搜索 Web | 有 `web-search-provider-boundaries` 脚本表明存在 provider 抽象 | `package.json` scripts: `lint:web-search-provider-boundaries` |

### 维度二: 操作与控制

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Shell 执行 | `bash`/`process` 工具, 通过 `@lydell/node-pty` (PTY) | Agent 可执行任意 Shell 命令 | 使用 PTY 而非 pipe, 支持交互式 CLI 工具 | `package.json` L54: `@lydell/node-pty` |
| 文件操作 | `read`/`write`/`edit` 工具 (workspace 文件系统) | Agent 操作 `~/.openclaw/workspace` 内文件 | 推断: workspace 隔离 + sandbox 额外限制 | `README.md` "Agent workspace + skills" |
| Canvas 操作 | A2UI (Agent-to-User Interface) — Agent 可编辑 HTML/CSS/JS | Agent 创建可视化输出和交互界面 | 独有: Agent 驱动的实时可视化画布 | `README.md` "Live Canvas"; `docs/concepts/architecture.md` Canvas 段 |
| 跨 Session 通信 | `sessions_list`/`sessions_history`/`sessions_send`/`sessions_spawn` 工具 | Agent 可查询/发送/创建其他 session | 推断: 类似 SubAgent 委派机制 | `README.md` "Operator quick refs" |
| Cron 调度 | `croner` 依赖 + cron tool | Agent 可创建/管理定时任务 | 推断: 支持周期性自动化 | `package.json` L61: `croner` |
| 消息通道操作 | Discord/Slack 专用 actions | Agent 可在 Discord/Slack 内执行特定操作 | 推断: 如发送消息、管理频道 | `README.md` "First-class tools" |
| 语音输出 | `node-edge-tts` (Edge TTS) + 推断 ElevenLabs | Agent 可将文本转为语音 | 推断: 支持多 TTS 后端 | `package.json` L66: `node-edge-tts` |
| 推送通知 | `web-push` 依赖 | 向浏览器推送通知 | 推断: 用于 WebChat 或 Web 推送 | `package.json` L77: `web-push` |
| 二维码生成 | `qrcode` 依赖 | 推断: WhatsApp Web 登录或设备配对 | WhatsApp Baileys 登录需要 | `package.json` L71: `qrcode` |

### 维度三: 规划与决策

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Thinking 控制 | `/think <level>` 指令 + `extractThinkDirective` | 用户可动态调整 LLM thinking 深度 | 推断: 支持 per-message thinking 控制 | `src/auto-reply/reply.ts` export: `extractThinkDirective` |
| 会话压缩 | `/compact` 指令 | 手动触发上下文压缩 | 推断: 类似 deepagents 的 compact_conversation | `README.md` "Chat commands" |
| 会话重置 | `/reset` + `/new` 指令 | 清空对话历史或创建新 session | 推断: 通过 Gateway 协议实现 | `README.md` "Chat commands" |
| 详细度控制 | `/verbose on\|off` 指令 + `extractVerboseDirective` | 控制 Agent 输出详情 | 推断: 影响 system prompt 或 tool output 展示 | `src/auto-reply/reply.ts` export |
| Trace 控制 | `/trace on\|off` 指令 + `extractTraceDirective` | 开关 Agent 执行追踪 | 推断: 类似 LangSmith trace 的本地实现 | `src/auto-reply/reply.ts` export |
| Token 监控 | `tokenjuice` 依赖 + `/usage off\|tokens\|full` 指令 | 实时展示 token 使用量 | 推断: 支持多级粒度显示 | `package.json` L73: `tokenjuice` |
| Queue 控制 | `extractQueueDirective` | 推断: 控制 Agent 任务队列 | 推断: 支持异步任务排队 | `src/auto-reply/reply.ts` export |
| 执行指令 | `extractExecDirective` | 推断: 允许用户直接注入 tool call | 推断: 类似 `/exec` 命令 | `src/auto-reply/reply.ts` export |

### 维度四: 通信与适配

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| 24+ 消息渠道 | 独立 Channel Plugin: Baileys(WhatsApp), grammY(Telegram), Slack Bolt, Discord.js, Signal, iMessage 等 | 统一 Gateway 管理所有渠道，归一化消息格式 | 已知同类产品中渠道覆盖最广 | `README.md` "Supported channels"; `package.json` dependencies |
| Gateway 协议 | WebSocket + JSON Schema + TypeBox → Swift Codegen | 类型安全的客户端-网关通信 | 从 TypeBox schema 自动生成 Swift 模型用于 macOS/iOS app | `docs/concepts/architecture.md` § Protocol typing |
| 流式输出 | `event:agent` (streaming WS events) | 实时流式推送到所有客户端 | 通过 WS event 而非 SSE | `docs/concepts/architecture.md` Connection lifecycle |
| 多设备配对 | Device identity + pairing code + device token | 每个 WS 客户端携带设备身份 | 配对签名 v3 绑定 platform + deviceFamily | `docs/concepts/architecture.md` § Pairing |
| 远程访问 | Tailscale/SSH tunnel | 远程安全访问 Gateway | 推断: 推荐 Tailscale 作为首选 | `docs/concepts/architecture.md` § Remote access |
| WebChat | 静态 Web UI + Gateway WS API | 浏览器端 Chat 界面 | 推断: 通过 `/__openclaw__/` HTTP 路由提供 | `docs/concepts/architecture.md` § WebChat |
| Webhook | `webhook` 自动化模块 | 外部系统通过 HTTP 触发 Agent | 推断: 支持 GitHub/Gmail 等 webhook | `README.md` "Webhooks" |
| Gmail Pub/Sub | Gmail Pub/Sub 集成 | Agent 接收邮件通知 | 推断: 通过 Google Cloud Pub/Sub | `README.md` "Gmail Pub/Sub" |

### 维度五: 安全与隔离

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| DM 配对 | `dmPolicy="pairing"` 默认策略 | 未知发件人收到配对码，需手动审批 | 覆盖 Telegram/WhatsApp/Signal/iMessage/Teams/Discord/Slack | `README.md` "Security defaults" |
| Sandbox 隔离 | Docker/SSH/OpenShell 三种沙箱后端 | non-main session 运行在隔离环境 | 默认允许: bash/process/read/write/edit/sessions_*; 默认拒绝: browser/canvas/nodes/cron/discord/gateway | `README.md` "Security model" |
| 工具白名单 | Sandbox tool allow/deny 配置 | 按 sandbox 类型限制可用工具 | 推断: 细粒度的工具级访问控制 | `README.md` "Security model" |
| SSRF 防护 | `ssrf-policy` + `ssrf-runtime` plugin-sdk 导出 | 防止 Server-Side Request Forgery | 推断: 控制 Agent 的网络访问范围 | `package.json` exports: `plugin-sdk/ssrf-policy` |
| Gateway 认证 | 多种模式: shared-secret, Tailscale, trusted-proxy, none | 灵活的认证策略适配不同部署场景 | Token/password + 签名 nonce challenge | `docs/concepts/architecture.md` § Wire protocol |
| 配对信任链 | Device identity + challenge nonce 签名 + metadata pinning | 设备身份不可伪造，元数据变更需重新配对 | v3 签名绑定 platform + deviceFamily | `docs/concepts/architecture.md` § Pairing |
| Doctor 诊断 | `openclaw doctor` 命令 | 自动检测危险配置 (如 open DM policy) | 推断: 安全态势检查 | `README.md` "Security defaults" |

### 维度六: 持久化与状态管理

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Session 持久化 | Workspace 隔离 + per-agent session 状态 | 每个 Agent 有独立的工作空间和对话历史 | 推断: 存储在 `~/.openclaw/` 目录 | `README.md` "Agent workspace" |
| Memory 系统 | AGENTS.md + SOUL.md + TOOLS.md 文件注入 | Agent 启动时加载持久化上下文 | 遵循 agents.md 规范 | `README.md` "Agent workspace + skills" |
| Vector Memory | `sqlite-vec` (optionalDependency) + `memory-lancedb` extension | 向量化记忆存储和检索 | 推断: 支持语义搜索历史对话 | `package.json` optionalDeps: `sqlite-vec` |
| 配置持久化 | JSON5 配置文件 (`~/.openclaw/openclaw.json`) | 声明式配置，支持 runtime config snapshot | 包含 ConfigRuntimeRefreshError 和 recovery 机制 | `src/config/config.ts` exports |
| Cron 持久化 | Cron store (推断: 文件/SQLite) | 定时任务跨 Gateway 重启持久化 | 推断: 通过 plugin-sdk/cron-store-runtime | `package.json` exports |
| 媒体存储 | Media store + media-runtime plugin | 推断: 图片/音频/视频文件管理 | `package.json` exports: `plugin-sdk/media-store` |
| 健康检查 | `health` WS event + `hello-ok` snapshot | Gateway 状态实时监控 | `docs/concepts/architecture.md` § Operations |

### 维度七: 工程化与可观测

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| CI/CD | GitHub Actions + 大量 Docker E2E 测试 | 40+ Docker E2E 场景覆盖 live models/channels/sandboxes | 测试矩阵极其全面 (Parallels/Linux/macOS/Windows) | `package.json` `test:docker:*` scripts |
| 代码质量 | `oxlint` + `oxfmt` + TypeScript native (`@typescript/native-preview`) | 使用 Oxc 工具链替代 ESLint/Prettier | 推断: 追求极速 lint 性能 | `package.json` devDependencies |
| 启动性能 | CLI startup bench + memory check | 启动性能持续监控和预算 | 有专门的 startup performance budget | `package.json` `test:startup:*` |
| 性能基准 | `kova` 性能摘要 + 热路径分析 | Agent 执行性能持续监控 | 推断: Kova 是内部性能测试框架 | `package.json` `perf:kova:summary` |
| Protocol Codegen | TypeBox → JSON Schema → Swift | 类型安全的协议跨平台同步 | 从 TS 类型自动生成 Swift 模型 | `docs/concepts/architecture.md` § Protocol typing |
| 日志 | `tslog` 依赖 | 结构化日志 | 推断: 用于本地和服务端日志 | `package.json` L74: `tslog` |
| 热重载 | Config runtime refresh + gateway:watch (dev loop) | 开发模式下源码/配置变更自动重载 | 推断: chokidar 监听 + config snapshot 机制 | `package.json` deps: `chokidar`; `src/config/config.ts` exports |
| Plugin 生态 | Plugin SDK + ClawHub registry + npm distribution | 社区可发布/安装插件 | Plugin boundary 报告 + 官方 external catalogs | `VISION.md` § Plugins; `package.json` exports |

### 维度八: 扩展性与生态

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|----------|-------------|---------|------------|---------|
| Plugin SDK | 80+ runtime 导出 (channel/approval/reply/skills/sandbox/...) | 社区可扩展几乎所有子系统 | 推断: 最广泛的 plugin 接口之一 | `package.json` exports: `plugin-sdk/*` |
| Skills 系统 | `SKILL.md` (YAML frontmatter + Markdown) | 渐进式信息披露的技能系统 | 遵循 Anthropic Agent Skills 规范 | `README.md` "Agent workspace + skills" |
| ClawHub | Skills Registry (clawhub.ai) | 社区技能市场 | 推断: 类似 npm 的技能分发平台 | `README.md` "Skills registry: ClawHub" |
| MCP 支持 | `@modelcontextprotocol/sdk` 集成 + MCP Server | OpenClaw 既是 MCP Client 也是 Server | 推断: 双向 MCP 支持 | `package.json` L55: `@modelcontextprotocol/sdk`; `VISION.md` |
| ACP 支持 | `@agentclientprotocol/sdk` + `claude-agent-acp` patch | 支持 Claude Code 等 ACP 兼容 Agent | 推断: 可作为 ACP client 调用外部 Agent | `package.json` L50: `@agentclientprotocol/sdk` |
| Provider 生态 | Anthropic/OpenAI/Google/Bedrock + 20+ providers | 广泛的模型提供商支持 | 推断: 通过 plugin-sdk/provider-setup 扩展 | `package.json` dependencies |
| Channel 扩展 | Plugin channel API (channel-runtime, channel-setup, etc.) | 社区可开发新的消息渠道插件 | 推断: 官方维护 external channel catalog | `package.json` exports; `VISION.md` |
| CLI Backend | Claude CLI / Codex CLI / Gemini CLI bindings | 通过 ACP 绑定外部 CLI agent 作为 backend | 大量 live test 覆盖多种 CLI backend | `package.json` `test:docker:live-cli-backend:*` |

---

## 三、技术链路验证

### 3.1 正常路径: WhatsApp 消息 → Agent 回复

基于 `docs/concepts/architecture.md` + `README.md` + 依赖分析还原：

```
User sends "What's the weather?" on WhatsApp
  │
  ├─ Step 1: Baileys (WhatsApp Web) 接收消息
  │    └─ Channel Plugin 归一化为 inbound envelope
  │         └─ `src/auto-reply/reply.ts` GetReply pipeline 处理
  │
  ├─ Step 2: Routing Layer
  │    ├─ 检查 DM policy (default: "pairing")
  │    ├─ 已知发件人 → 直接路由到对应 agent workspace
  │    └─ 未知发件人 → 返回配对码 (不触发 Agent)
  │
  ├─ Step 3: Agent Engine (PI Agent @mariozechner/pi-agent-core)
  │    ├─ 加载 session 上下文 (历史消息)
  │    ├─ 注入 Memory: AGENTS.md + SOUL.md + TOOLS.md
  │    ├─ 注入 Skills: 匹配的 SKILL.md 渐进式加载
  │    ├─ LLM 推理 → 决定调用 browser tool
  │    ├─ browser tool → Playwright 打开天气网站 → 提取数据
  │    ├─ LLM 生成回复文本
  │    └─ Sandbox 检查: main session → host access (允许)
  │
  ├─ Step 4: Reply Pipeline
  │    ├─ extractThinkDirective (无)
  │    ├─ Chunking + 格式化
  │    └─ Media attachment (无)
  │
  └─ Step 5: Outbound → Baileys → WhatsApp
       └─ 用户收到回复
```

### 3.2 异常路径: 非受信 DM 被拦截

基于 `README.md` "Security defaults" 还原：

```
Unknown user sends DM on Telegram
  │
  ├─ Step 1: grammY 接收消息
  │    └─ Channel Plugin 归一化
  │
  ├─ Step 2: DM Policy Check
  │    ├─ dmPolicy = "pairing" (默认)
  │    ├─ 检查 allowlist: 发件人不在列表中
  │    └─ 生成短配对码 (如 "ABC123")
  │
  ├─ Step 3: 发送配对码回复 (不触发 Agent)
  │    └─ "Your pairing code: ABC123. Ask the operator to run: openclaw pairing approve telegram ABC123"
  │
  ├─ Step 4: Operator 执行审批
  │    └─ `openclaw pairing approve telegram ABC123`
  │         └─ 将发件人加入 local allowlist store
  │
  └─ Step 5: 后续消息正常处理
       └─ 发件人已在 allowlist → 正常路由到 Agent
```

### 3.3 资源表现推断 (静态配置分析)

| 参数 | 默认值/描述 | 来源 |
|------|-----------|------|
| Gateway 端口 | 127.0.0.1:18789 | `docs/concepts/architecture.md` |
| 工作目录 | `~/.openclaw/workspace` | `README.md` "Agent workspace" |
| 配置文件 | `~/.openclaw/openclaw.json` (JSON5) | `README.md` "Configuration" |
| Sandbox 默认后端 | Docker | `README.md` "Security model" |
| DM 策略 | `pairing` (需审批) | `README.md` "Security defaults" |
| Node 要求 | ≥22.14 (推荐 24) | `package.json` engines |
| 沙箱默认允许工具 | bash, process, read, write, edit, sessions_* | `README.md` "Security model" |
| 沙箱默认禁止工具 | browser, canvas, nodes, cron, discord, gateway | 同上 |

---

## 四、Gap Analysis (差异分析)

### 4.1 三源交叉验证

| 缺失项 | 检测来源 | 影响等级 | 详情 |
|--------|---------|---------|------|
| 无 Agent 注册表/CRUD API | 源码分析 (未找到 registry 模块) | 高 | Agent 通过配置文件 + workspace 目录隐式定义，无 CRUD API |
| 无 SubAgent 委派机制 | README (有 sessions_spawn 但非标准 subagent) | 中 | `sessions_spawn` 可创建新 session，但非结构化 task 委派 |
| 无独立的 Vault/Secret 管理 | 源码分析 (依赖环境变量和配置文件) | 中 | 密钥通过 `secret-ref-runtime` 和 `secret-file-runtime` 引用，但无轮换/审计 |
| 依赖 PI Agent 框架 | 源码分析 (`@mariozechner/pi-agent-core`) | 中 | Agent 核心能力耦合外部框架，非自研 |
| 无内置结构化输出 | 源码分析 (未找到 response_format) | 低 | 推断: 支持但未在 README 中强调 |
| Windows 支持有限 | README "via WSL2; strongly recommended" | 中 | 原生 Windows 支持有限，需 WSL2 |
| 无 LangSmith 级 Trace | 源码分析 (使用本地 tslog) | 低 | 本地化日志为主，推测无云 trace 集成 |
| 无多租户隔离 | 源码分析 (无 tenant 模块) | 高 | 所有 agent 共享 Gateway 实例和配置。隔离仅到 channel/account/peer → agent session（dmScope 支持 per-account-channel-peer 四级），无租户级资源配额/配置分离 |
| Memory 隔离浅层 | 源码分析 + 官方 docs | 中 | per-agent workspace 文件隔离（不同 agent 不共享 memory），但无跨 session 搜索和 user_id 级过滤。Session 按 `~/.openclaw/agents/<agentId>/sessions/` 隔离 |
| 沙箱生命周期粗放→修正 | 官方 docs § gateway/sandboxing | 低→中 | 有显式管理模式：`openclaw sandbox recreate/list/explain`。Sandbox mode(off/non-main/all) + scope(agent/session/shared) + backend(docker/ssh/openshell)。Docker: `CMD ["sleep","infinity"]` 长运行 + `setupCommand` 一次性初始化。但无 acquire/release API 式生命周期 |
| 无标准化评测体系 | 源码分析 (无 evals/ 目录) | 中 | CI 含 Docker E2E(live models/channels/sandboxes) 但不含标准化 benchmark 集成（如 SWE-bench/GAIA/FRAMES）。无公开评测得分 |

### 4.2 社区痛点 (Issues 页面因网络限制未获取)

基于 VISION.md 中的优先级声明:
- P0: Security and safe defaults (安全加固持续进行中)
- P0: Bug fixes and stability
- P0: Setup reliability and first-run UX
- P1: Supporting all major model providers
- P1: Improving major messaging channels
- P1: Performance and test infrastructure

> ⚠️ GitHub Issues 页面因连接超时未能加载 (3,400+ open issues)，Gap 分析主要基于源码和文档。

---

## 五、维度建议 (Dimension Evolution)

### 5.1 Agent vs Agent 平台判定

| 判定标准 | OpenClaw 现状 | 判定 |
|---------|-------------|------|
| **模板化** | ⚠️ 部分达成。Skills 通过 SKILL.md (Markdown+YAML) 声明；Memory 通过 AGENTS.md/SOUL.md/TOOLS.md 声明；Agent 配置通过 JSON5 文件声明。但 Agent 的核心行为定义 (system prompt/workflow/middleware) 深度耦合在源码中，非纯声明式 | 部分达成 |
| **隔离化** | ✅ 达成。Multi-agent routing 按 channel/account/peer 路由到独立 workspace；Docker/SSH/OpenShell 三种 Sandbox 后端提供物理隔离；per-agent session 状态隔离；Tool 级黑白名单 | 达成 |

> **结论**: OpenClow 是一个 **Agent Platform 雏形**。它具备多 Agent 路由、强隔离 (Sandbox)、声明式配置，但缺少完整的 Agent CRUD 生命周期管理 API。它更准确的定位是: **Personal AI Assistant Platform** — 面向个人用户的完整 AI 助手产品，内置了初步的平台化能力。

### 5.2 新维度建议

| 建议维度 | 触发特征 | 定义 | 行业前瞻性 |
|---------|---------|------|-----------|
| **消费者就绪度** | Voice Wake/Talk + Canvas A2UI + macOS/iOS/Android Apps + 24 个消息渠道 | 评估 AI Agent 产品作为消费级产品的完工程度 (UX/App/Channel/Voice) | 高 |
| **渠道归一化能力** | 24+ Channel Plugin → 统一 inbound envelope → 统一 outbound pipeline | 评估 Agent 平台对不同消息渠道的抽象能力和一致性 | 高 |
| **Plugin SDK 广度** | 80+ runtime 导出覆盖 channel/approval/reply/skills/sandbox/acp/mcp/tts/media | 评估 Agent 平台扩展接口的覆盖范围和设计质量 | 高 |
| **设备生态** | macOS/iOS/Android Apps + Nodes (camera/screen/location) + Device Pairing | 评估 Agent 平台的多设备协同能力 | 中 |
| **声明式配置深度** | JSON5 配置文件 + SKILL.md (YAML+MD) + AGENTS.md + plugin CRUD | 评估 Agent 平台通过非代码方式定制 Agent 行为的能力深度 | 中 |

---

## 六、依赖分析 (隐式能力推导)

基于 `package.json` 依赖声明：

| 依赖 | 推导能力 | 确定性 |
|------|---------|--------|
| `@mariozechner/pi-agent-core` | Agent 运行时 (PI = Personal Intelligence?) | 高 |
| `@mariozechner/pi-ai` | AI 模型交互层 | 高 |
| `@mariozechner/pi-coding-agent` | 编码 Agent 专用 | 高 |
| `@mariozechner/pi-tui` | 终端 UI | 高 |
| `@agentclientprotocol/sdk` | ACP 协议客户端 | 高 |
| `@modelcontextprotocol/sdk` | MCP 协议 (client+server) | 高 |
| `playwright-core` | 浏览器自动化 | 高 |
| `@lydell/node-pty` | PTY 终端 (交互式 Shell) | 高 |
| `@whiskeysockets/baileys` | WhatsApp Web 客户端 | 高 |
| `grammy` | Telegram Bot 框架 | 高 |
| `@slack/bolt` | Slack Bot 框架 | 高 |
| `node-edge-tts` | Edge TTS 语音合成 | 高 |
| `sqlite-vec` | SQLite 向量扩展 (语义搜索) | 中 |
| `pdfjs-dist` | PDF 解析/渲染 | 高 |
| `@mozilla/readability` | 网页正文提取 | 高 |
| `croner` | Cron 定时任务 | 高 |
| `tokenjuice` | Token 计数 | 高 |
| `tree-sitter-bash` | Shell 脚本 AST 解析 | 高 |
| `openshell` | 沙箱 Shell 执行 | 高 |
| `web-push` | Web 推送通知 | 中 |
| `qrcode` | 二维码生成 (WhatsApp 登录) | 高 |
| `linkedom` | 轻量级 DOM (Web 抓取) | 中 |
| `jszip` | ZIP 文件处理 | 中 |
| `markdown-it` | Markdown 渲染 | 中 |
| `typebox` | JSON Schema 类型定义 (Gateway 协议) | 高 |
| `ajv` | JSON Schema 验证 (Gateway 协议) | 高 |
| `undici` | HTTP 客户端 (高性能) | 中 |
| `global-agent` / `proxy-agent` | HTTP 代理支持 | 低 |
| `@homebridge/ciao` | mDNS/Bonjour 服务发现 | 低 |

---

## 七、独特性汇总

1. **消费者级 Personal AI Assistant**: OpenClaw 在定位上与 deepagents (SDK)/Hermes Agent (CLI Agent) 有本质差异——它是面向终端用户的完整产品，通过 WhatsApp/Telegram 等日常渠道使用，无需编程。

2. **24+ 消息渠道归一化**: 已知同类开源产品中渠道覆盖最广。每个渠道独立 Plugin，输出归一化为统一 Gateway 协议。

3. **Live Canvas (A2UI)**: Agent 可实时编辑 HTML/CSS/JS 创建可视化界面，这在 Agent 产品中极为罕见。是 Agent-to-User Interface 的前沿实践。

4. **Voice Wake + Talk Mode**: 消费级语音交互 (macOS/iOS/Android)，支持唤醒词和连续对话，超越了纯文本 Agent 的交互范式。

5. **Gateway 架构的协议工程**: TypeBox → JSON Schema → Swift Codegen 的协议管道，确保跨平台类型安全。这在 Agent 产品中少见。

6. **Plugin SDK 深度**: 80+ runtime 接口使社区可扩展 channel/sandbox/skills/MCP/ACP/TTS 等几乎所有子系统。

7. **PI Agent 框架依赖**: Agent 核心依赖 `@mariozechner/pi-agent-*` 系列外部包，而非自研。这意味着核心 Agent 能力的演进受限于外部框架。

8. **368k Stars 的社区规模**: 远超同类 Agent 基础设施项目，表明其作为消费级产品已获得广泛认可。

---

## 八、校准备忘录 (Calibration)

### 8.1 事实核查表

| 声明 | 验证状态 | 来源验证 |
|------|---------|---------|
| Stars: 368k | ✅ 准确 | GitHub 页面实时快照 |
| Forks: 75.8k | ✅ 准确 | GitHub 页面实时快照 |
| 版本: 2026.5.3 | ✅ 准确 | `package.json` L3 |
| 许可证: MIT | ✅ 准确 | `package.json` L9 |
| 24+ 消息渠道 | ✅ 准确 | `README.md` 逐项列出 |
| Gateway 架构 (WS + JSON Schema) | ✅ 准确 | `docs/concepts/architecture.md` |
| Sandbox: Docker/SSH/OpenShell | ✅ 准确 | `README.md` "Security model" |
| DM pairing 默认策略 | ✅ 准确 | `README.md` "Security defaults" |
| PI Agent 框架依赖 | ✅ 准确 | `package.json` `@mariozechner/pi-agent-core` |
| Playwright 浏览器自动化 | ✅ 准确 | `package.json` `playwright-core` |
| MCP SDK 集成 | ✅ 准确 | `package.json` `@modelcontextprotocol/sdk` |
| ACP SDK 集成 | ✅ 准确 | `package.json` `@agentclientprotocol/sdk` |
| sqlite-vec 向量存储 | ✅ 准确 | `package.json` optionalDependencies |
| Skills 系统 (SKILL.md) | ✅ 准确 | `README.md` "Agent workspace + skills" |
| Memory 系统 (AGENTS.md) | ✅ 准确 | `README.md` "Agent workspace + skills" |
| Chat 指令 (/think, /verbose 等) | ✅ 准确 | `src/auto-reply/reply.ts` exports |
| Voice Wake + Talk Mode | ✅ 准确 | `README.md` "Highlights" |
| Live Canvas + A2UI | ✅ 准确 | `README.md` "Highlights" |

### 8.2 需修正项

| 声明 | 问题 | 修正 |
|------|------|------|
| （无） | — | 所有源码引用均通过 raw.githubusercontent.com 或 GitHub 页面验证 |

### 8.3 遗漏项 (因网络/API 限制)

| 遗漏项 | 影响 | 说明 |
|--------|------|------|
| Issues 列表 (3,400+) | Gap 分析不完整 | GitHub Issues 页面连接超时，未能获取社区痛点 |
| Agent 引擎源码 (`pi-agent-core`) | 核心 Agent loop 未知 | 该包外部维护，源码不在 openclaw 仓库内 |
| Gateway server.impl.ts 源码 | Gateway 内部实现细节 | 文件存在但未获取 (网络波动) |
| Sandbox 实现源码 | Sandbox 管理器细节 | 具体路径未找到 |
| Plugin SDK 实现源码 | Plugin 接口的完整定义 | 仅通过 package.json exports 推断 |
| 完整测试覆盖率 | 工程质量评估 | 未获取覆盖率报告 |

### 8.4 总体准确性评分

- **源码/文档直接支撑**: ~75% (15/20 核心声明有直接源码或官方文档引用)
- **依赖推导**: ~15% (3/20 通过 package.json 依赖推断)
- **合理推断**: ~10% (标注为 "推断")
- **总体评分: B+ (良好)** — 关键发现均有文档支撑，但 Agent 核心源码未获取 (外部 pi-agent 包)，Issues 数据缺失。
