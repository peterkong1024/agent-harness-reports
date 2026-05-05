## DeerFlow 2.0 垂直能力解构报告

### 基于"特征逆向挖掘法"的 Harness Agent 深度分析

---

**分析版本**: v2.0 (config_version: 8, ground-up rewrite)  
**分析日期**: 2026-05-03  
**执行人**: AI Agent (Hermes)  
**仓库地址**: https://github.com/bytedance/deer-flow  
**官方站点**: https://deerflow.tech  
**许可证**: MIT  
**Star**: ~64.6k | **Fork**: ~8.5k | **Commits**: 2,036+ | **Issues**: 449 | **PRs**: 258  

> **方法论声明**: 本报告遵循"特征逆向挖掘法"——不预设评价维度，由底层代码实现、文档语义和社区反馈反向推导产品的能力原子。

> **数据来源权威性层级**:
> 1. **源码** (`.py` 文件 — 最权威): `backend/packages/harness/deerflow/` 下的 Python 实现代码
> 2. **源码内配置与开发文档** (第二权威): `config.example.yaml`、`pyproject.toml`、`CLAUDE.md`、`backend/docs/` 目录
> 3. **官方文档** (第三权威): `README.md`、`backend/README.md`、GitHub Issues
> 4. **推断** (需标注): 基于源码逻辑的合理推导，以 "推断:" 前缀注明
>
> 报告中"资料来源"列严格遵循此层级。标注为 `CLAUDE.md` / `config.example.yaml` 的声明，均可在此文件中找到原文；标注为 `*.py` 的声明，均可在此源码文件中找到对应实现。

---

## 一、产品画像与核心定位 (Identity & Positioning)

### 官方定义

> "DeerFlow (Deep Exploration and Efficient Research Flow) is an open-source super agent harness that orchestrates sub-agents, memory, and sandboxes to do almost anything — powered by extensible skills."
>
> — [`README.md`](https://github.com/bytedance/deer-flow/blob/main/README.md)

### 演化路径

| 阶段 | 定位 | 代码状态 |
|------|------|---------|
| v1.x (main-1.x 分支) | Deep Research 框架 | 独立维护，社区仍可贡献 |
| v2.0 (main 分支) | SuperAgent Harness | 从零重写，与 v1 无代码共享 |

> **Source**: `README.md` — "DeerFlow 2.0 is a ground-up rewrite. It shares no code with v1."

### 技术栈选型

| 层次 | 选型 | 出处 |
|------|------|------|
| **Agent 编排** | LangGraph >=1.1.9 + LangChain >=1.2.15 | `pyproject.toml` § dependencies |
| **后端语言** | Python >=3.12 (type hints, ruff lint/format) | `CLAUDE.md` § Code Style |
| **前端** | Next.js (Node.js >=22) | `CLAUDE.md` § Project Overview |
| **API 网关** | FastAPI + Nginx + LangGraph Server (3 进程) | `ARCHITECTURE.md` § System Architecture |
| **包管理** | uv (Python) / pnpm (Node) | `CLAUDE.md` § Commands |
| **容器化** | Docker + Docker Compose + Apple Container (macOS 自动检测) | `README.md` § Quick Start; `APPLE_CONTAINER.md` |
| **数据库** | SQLite (默认, WAL mode) / PostgreSQL (生产) | `config.example.yaml` § Database |
| **关键依赖** | kubernetes>=30, duckdb>=1.4.4, langfuse>=3.4.1, agent-sandbox>=0.0.19, agent-client-protocol>=0.4.0 | `pyproject.toml` |
| **可选依赖** | ollama (langchain-ollama), postgres (asyncpg+psycopg), pymupdf (pymupdf4llm) | `pyproject.toml` § optional-dependencies |

### 接入形态

| 形态 | 描述 | 出处 |
|------|------|------|
| **Web UI** | Next.js 前端 (localhost:2026) | `README.md` § Quick Start |
| **HTTP API** | Gateway REST API (localhost:8001) | `CLAUDE.md` § Gateway API |
| **Embedded Client** | `DeerFlowClient` — 无需 HTTP 服务的 Python API | `client.py` |
| **IM Channels** | Telegram / Slack / 飞书 / 微信 / 企业微信 / 钉钉 | `README.md` § IM Channels |
| **Claude Code** | `claude-to-deerflow` Skill — 在 Claude Code 中直接交互 | `README.md` § Claude Code Integration |
| **ACP 协议** | `invoke_acp_agent` — 跨进程 Agent 互操作 | `config.example.yaml` § acp_agents |

### 系统架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              Client (Browser)                             │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          Nginx (Port 2026)                               │
│                    Unified Reverse Proxy Entry Point                      │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  /api/langgraph/*  →  LangGraph Server (2024)                      │  │
│  │  /api/*            →  Gateway API (8001)                           │  │
│  │  /*                →  Frontend (3000)                               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│   LangGraph Server  │ │    Gateway API      │ │     Frontend        │
│     (Port 2024)     │ │    (Port 8001)      │ │    (Port 3000)      │
│                     │ │                     │ │                     │
│  - Agent Runtime    │ │  - Models API       │ │  - Next.js App      │
│  - Thread Mgmt      │ │  - MCP Config       │ │  - React UI         │
│  - SSE Streaming    │ │  - Skills Mgmt      │ │  - Chat Interface   │
│  - Checkpointing    │ │  - File Uploads     │ │                     │
│                     │ │  - Thread Cleanup   │ │                     │
│                     │ │  - Artifacts        │ │                     │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
          │                       │
          │     ┌─────────────────┘
          │     │
          ▼     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         Shared Configuration                              │
│  ┌─────────────────────────┐  ┌────────────────────────────────────────┐ │
│  │      config.yaml        │  │      extensions_config.json            │ │
│  │  - Models               │  │  - MCP Servers                         │ │
│  │  - Tools                │  │  - Skills State                        │ │
│  │  - Sandbox              │  │                                        │ │
│  │  - Summarization        │  │                                        │ │
│  └─────────────────────────┘  └────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### Agent 架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           make_lead_agent(config)                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            Middleware Chain                              │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 1. ThreadDataMiddleware  - Initialize workspace/uploads/outputs  │   │
│  │ 2. UploadsMiddleware     - Process uploaded files               │   │
│  │ 3. SandboxMiddleware     - Acquire sandbox environment          │   │
│  │ 4. SummarizationMiddleware - Context reduction (if enabled)     │   │
│  │ 5. TitleMiddleware       - Auto-generate titles                 │   │
│  │ 6. TodoListMiddleware    - Task tracking (if plan_mode)         │   │
│  │ 7. ViewImageMiddleware   - Vision model support                 │   │
│  │ 8. ClarificationMiddleware - Handle clarifications              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Agent Core                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │
│  │      Model       │  │      Tools       │  │    System Prompt     │   │
│  │  (from factory)  │  │  (configured +   │  │  (with skills)       │   │
│  │                  │  │   MCP + builtin) │  │                      │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

> **注**: ARCHITECTURE.md 中展示简化版 8 层 Middleware，CLAUDE.md 中有完整 18 层（见 § 3.2）。

### Sandbox 架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § Sandbox System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Sandbox Architecture                           │
└─────────────────────────────────────────────────────────────────────────┘

                      ┌─────────────────────────┐
                      │    SandboxProvider      │ (Abstract)
                      │  - acquire()            │
                      │  - get()                │
                      │  - release()            │
                      └────────────┬────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
┌─────────────────────────┐ ┌─────────────────────┐ ┌──────────────────┐
│  LocalSandboxProvider   │ │ AioSandboxProvider  │ │ (Apple Container │
│  (local.py)             │ │ + Docker            │ │  auto-detected)  │
│                         │ │ + Provisioner (K8s) │ │                  │
│  - Singleton instance   │ │ - Docker-based      │ │ - macOS ARM64    │
│  - Direct execution     │ │ - Isolated container│ │ - Virtualization │
│  - Development use      │ │ - Production use    │ │ - Better perf    │
└─────────────────────────┘ └─────────────────────┘ └──────────────────┘

                      ┌─────────────────────────┐
                      │        Sandbox          │ (Abstract)
                      │  - execute_command()    │
                      │  - read_file()          │
                      │  - write_file()         │
                      │  - list_dir()           │
                      └─────────────────────────┘
```

### Tool 系统架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § Tool System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Tool Sources                                  │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   Built-in Tools    │  │  Configured Tools   │  │     MCP Tools       │
│  (tools/)           │  │  (config.yaml)      │  │  (extensions.json)  │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│ - present_files     │  │ - web_search        │  │ - github            │
│ - ask_clarification │  │ - web_fetch         │  │ - filesystem        │
│ - view_image        │  │ - bash              │  │ - postgres          │
│                     │  │ - read_file         │  │ - brave-search      │
│                     │  │ - write_file        │  │ - puppeteer         │
│                     │  │ - str_replace       │  │ - ...               │
│                     │  │ - ls                │  │                     │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
           │                       │                       │
           └───────────────────────┴───────────────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   get_available_tools() │
                      │   (tools/__init__)      │
                      └─────────────────────────┘
```

### Model Factory 架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § Model Factory

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Model Factory                                   │
│                     (models/factory.py)                                  │
└─────────────────────────────────────────────────────────────────────────┘

config.yaml:
┌─────────────────────────────────────────────────────────────────────────┐
│ models:                                                                  │
│   - name: gpt-4                                                         │
│     display_name: GPT-4                                                 │
│     use: langchain_openai:ChatOpenAI                                    │
│     model: gpt-4                                                        │
│     api_key: $OPENAI_API_KEY                                            │
│     supports_thinking: false                                            │
│     supports_vision: true                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   create_chat_model()   │
                      │  - name: str            │
                      │  - thinking_enabled     │
                      └────────────┬────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   resolve_class()       │
                      │  (reflection system)    │
                      └────────────┬────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   BaseChatModel         │
                      │  (LangChain instance)   │
                      └─────────────────────────┘
```

### MCP 集成架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § MCP Integration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MCP Integration                                 │
│                        (mcp/manager.py)                                  │
└─────────────────────────────────────────────────────────────────────────┘

extensions_config.json:
┌─────────────────────────────────────────────────────────────────────────┐
│ {                                                                        │
│   "mcpServers": {                                                       │
│     "github": {                                                         │
│       "enabled": true,                                                  │
│       "type": "stdio",                                                  │
│       "command": "npx",                                                 │
│       "args": ["-y", "@modelcontextprotocol/server-github"],           │
│       "env": {"GITHUB_TOKEN": "$GITHUB_TOKEN"}                          │
│     }                                                                   │
│   }                                                                     │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │  MultiServerMCPClient   │
                      │  (langchain-mcp-adapters)│
                      └────────────┬────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
       ┌───────────┐        ┌───────────┐        ┌───────────┐
       │  stdio    │        │   SSE     │        │   HTTP    │
       │ transport │        │ transport │        │ transport │
       └───────────┘        └───────────┘        └───────────┘
```

### Skills 系统架构图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § Skills System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Skills System                                   │
│                       (skills/loader.py)                                 │
└─────────────────────────────────────────────────────────────────────────┘

Directory Structure:
┌─────────────────────────────────────────────────────────────────────────┐
│ skills/                                                                  │
│ ├── public/                        # Public skills (committed)           │
│ │   ├── pdf-processing/                                                 │
│ │   │   └── SKILL.md                                                    │
│ │   ├── frontend-design/                                                │
│ │   │   └── SKILL.md                                                    │
│ │   └── ...                                                             │
│ └── custom/                        # Custom skills (gitignored)          │
│     └── user-installed/                                                 │
│         └── SKILL.md                                                    │
└─────────────────────────────────────────────────────────────────────────┘

SKILL.md Format:
┌─────────────────────────────────────────────────────────────────────────┐
│ ---                                                                      │
│ name: PDF Processing                                                     │
│ description: Handle PDF documents efficiently                            │
│ license: MIT                                                            │
│ allowed-tools:                                                          │
│   - read_file                                                           │
│   - write_file                                                          │
│   - bash                                                                │
│ ---                                                                      │
│                                                                          │
│ # Skill Instructions                                                     │
│ Content injected into system prompt...                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 二、原子化特征提取表 (Atomic Feature Extraction)

> 方法: 通过扫描 `packages/harness/deerflow/` 目录树 (`CLAUDE.md` § Project Structure)、30+ 项 `pyproject.toml` 依赖、以及 `config.example.yaml` 的完整 Schema，提取出以下原子能力。每一项只记录"有什么"，不做定性判断。

> **注**: `CLAUDE.md` 列出完整 18 个 Middleware（权威版），`backend/README.md` 列出简化 9 个。本报告以 CLAUDE.md 为准。

### 2.1 感知与输入维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| 视觉理解 (Vision) | `ViewImageMiddleware` — 在 LLM 调用前注入 Base64 图像数据；`view_image_tool` 读取图像 | Agent 可"看到"用户上传的截图/照片并基于视觉内容做出决策，如分析 UI 截图、识别图表数据 | ModelConfig 含 `supports_vision` 开关，Provider 级适配；条件注入（仅 vision-capable 模型） | `CLAUDE.md` § Middleware #14; `config.example.yaml` § models.supports_vision |
| 多格式文档解析 | `markitdown[all,xlsx]` — PDF/PPT/Excel/Word 自动转 Markdown；可选 `pymupdf4llm` 加速 PDF | 用户拖入 PDF/Word/PPT 后自动转为 Agent 可理解的 Markdown，支持表格和图片提取；含 `pdf_converter: auto` 智能选最优引擎 | 上传时自动转换，`auto_convert_documents` 默认关闭（安全考虑—Parser 在 Host 侧运行） | `pyproject.toml` § markitdown; `config.example.yaml` § uploads; `CLAUDE.md` § File Upload |
| Web 内容抓取 | Jina AI Reader API (`readabilipy` 可读性提取) / Exa / InfoQuest / Firecrawl — 含 `timeout` 和 `fetch_time` 配置 | Agent 可实时获取任意 URL 的清洗后正文内容（去除广告/导航），支持超时控制和导航等待 | 4 种独立 Provider，可按需替换；InfoQuest 为 ByteDance 自研爬虫引擎 | `CLAUDE.md` § Community Tools; `config.example.yaml` § tools.web_fetch |
| 图像搜索 | DuckDuckGo / InfoQuest — 支持 `image_size` 过滤和 `image_search_time_range` | Agent 在生成图片前可搜索参考图作为风格/构图依据，支持按尺寸(L/M/I)和时间范围过滤 | 预置生成前参考图搜索；DuckDuckGo 零 API Key 门槛 | `config.example.yaml` § tools.image_search; `CLAUDE.md` § Community Tools |
| 代码库理解 | `glob` + `grep` 工具 — 文件模式匹配 (max 200) + 内容正则搜索 (max 100) | Agent 可像开发者一样在项目目录中按文件名模式查找文件、按正则搜索代码内容，不消耗 LLM Token 做盲目扫描 | 非 LLM 依赖的代码感知；`grep` 返回结构化 `GrepMatch` 对象（文件+行号+内容） | `backend/packages/harness/deerflow/sandbox/tools.py` (glob_tool, grep_tool); `config.example.yaml` § tools.glob/grep |

### 2.2 操作与控制维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| Shell 执行 | `bash` tool — 自动虚拟路径翻译；`LocalSandboxProvider` 默认禁用 host bash | Agent 可在隔离环境中执行任意 Shell 命令完成编译、安装依赖、运行脚本等操作；输出自动中间截断（20,000 chars）防止 Token 爆炸 | 四层隔离模式控制 Shell 可达范围；Local 模式含命令注入防御（`_SHELL_COMMAND_SEPARATORS` 检测） | `sandbox/tools.py` (bash_tool); `config.example.yaml` § sandbox.allow_host_bash; `config.example.yaml` § sandbox.bash_output_max_chars |
| 文件读写 | `read_file`(偏移+截断)、`write_file`(自动 mkdir)、`str_replace`(per-sandbox锁) | Agent 可像编辑器一样精确替换文件中的字符串（支持单次/全部替换），写文件时自动创建父目录 | `str_replace` 按 `(sandbox.id, path)` 序列化，防止不同 Sandbox 对同虚拟路径的并发写冲突 | `sandbox/tools.py`; `CLAUDE.md` § Sandbox Tools; `sandbox/file_operation_lock.py` |
| 目录操作 | `ls` tool — 树形格式，最大2层 | Agent 可浏览文件系统结构，输出为人类可读的树形格式（含文件类型标识），自动截断防止大目录淹没问题 | 输出截断策略差异化：bash 中间截断（首尾保留）、ls/read_file 头部截断 | `sandbox/tools.py` (ls_tool); `config.example.yaml` § sandbox.ls_output_max_chars |
| 跨应用调用 | `invoke_acp_agent` — 启动外部 ACP Agent (Claude Code/Codex)；ACP workspace 卷挂载 | Lead Agent 可委派编码任务给 Claude Code 或 Codex CLI，ACP workspace 在 Sandbox 内以 `/mnt/acp-workspace` (只读) 挂载，结果自动回传 | ACP 适配器独立于模型 Provider—Claude Code OAuth 读 `~/.claude/.credentials.json`，Codex 读 `~/.codex/auth.json` | `config.example.yaml` § acp_agents; `CLAUDE.md` § ACP Agent Tools |
| 产物输出 | `present_files` tool — 将 `/mnt/user-data/outputs/` 文件暴露给用户 | Agent 生成报告/图表/代码后主动"呈现"文件给用户，而非默默写入磁盘 | 白名单限制仅 outputs 目录—Agent 无法通过此工具暴露 workspace 或 uploads 内容 | `CLAUDE.md` § Built-in Tools (present_files) |
| 人机交互中断 | `ask_clarification` tool → `ClarificationMiddleware` 拦截 → `Command(goto=END)` 中断 | Agent 遇到歧义时主动中断执行并向用户提问，用户回复后 Agent 从断点继续；避免"盲目猜测"导致的错误级联 | Middleware #18 必须置于管道末尾—否则中断后后续 Middleware 仍会执行 | `CLAUDE.md` § Middleware Chain #18; `lead_agent/agent.py` (_build_middlewares) |
| Agent 个性管理 | `agents_api.enabled` — Gateway 暴露 Custom Agent SOUL/USER.md 管理端点 | 管理员可通过 HTTP API 远程管理自定义 Agent 的行为定义文件（SOUL/USER.md），无需直接操作文件系统 | 默认关闭—需确保 Gateway 在可信认证管理边界后方可启用 | `config.example.yaml` § agents_api |

### 2.3 规划与决策维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| 任务分解 (Delegation) | `task()` tool → `SubagentExecutor` — 双线程池 (scheduler 3 + execution 3) | 主 Agent 可将复杂任务分解为最多 3 个并行子任务，每个子 Agent 拥有独立上下文和工具集；超时 900s 后自动终止 | `MAX_CONCURRENT_SUBAGENTS=3`，`SubagentLimitMiddleware` 截断超量调用；ContextVar 跨线程保留 | `subagents/executor.py` (791 行); `CLAUDE.md` § Subagent System; `config.example.yaml` § subagents |
| Plan Mode | `TodoListMiddleware` — `write_todos` 工具；system prompt 定义"立即完成/保持一个 in_progress"规则 | Agent 处理 3+ 步骤的复杂任务时自动创建 Todo 列表，实时更新进度，用户可在 Web UI 看到任务进展 | 条件启用 (`is_plan_mode=True`)；规则约束"Exactly ONE task in_progress at any time"防止并发混乱 | `CLAUDE.md` § Middleware #10; `lead_agent/agent.py` (_create_todo_list_middleware) |
| Loop 检测与终止 | `LoopDetectionMiddleware` — 检测重复 tool-call 循环 | 当 Agent 陷入死循环（反复调用同一工具无进展），Middleware 硬停止并强制生成文本回答，防止无限 Token 消耗 | 硬停止时同时清除 `tool_calls` 和 `additional_kwargs["tool_calls"]` 中的 raw provider 元数据 | `CLAUDE.md` § Middleware Chain #17 |
| Thinking 动态切换 | `supports_thinking` + `when_thinking_enabled/disabled` — runtime 可控 | 用户可按任务复杂度动态开关推理模式：简单任务关 thinking 提速，复杂任务开 thinking 提升质量 | 每个模型独立配置 extra_body；vLLM Qwen 模型通过 `chat_template_kwargs.enable_thinking` 控制 | `config.example.yaml` § models; `CLAUDE.md` § Model Factory |
| 推理模型补丁 | `PatchedChatDeepSeek`, `PatchedChatOpenAI` — 修复特定 Provider 的 reasoning/thought_signature 丢失 | 用户使用 DeepSeek V3 或 Gemini Thinking 时不会因 tool-call 序列中缺少 thought_signature 而收到 HTTP 400 错误 | 透明修复—用户只需在 `config.yaml` 中将 `use` 指向 Patched Provider，其余无感知 | `CLAUDE.md` § vLLM Provider; `config.example.yaml` § Models (Gemini thought_signature 注释) |
| 上下文感知规划 | System Prompt 注入 Skills、Memory、Subagent 指令 | Agent 在规划时可参考用户历史偏好 (Memory)、可用能力清单 (Skills) 和并行执行选项 (Subagents)，产出更精准的执行计划 | `apply_prompt_template()` 动态组装所有上下文源，非硬编码 prompt | `lead_agent/agent.py` (apply_prompt_template); `CLAUDE.md` § Agent System |

### 2.4 通信与适配维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| 多 IM 通道 | Telegram (长轮询)、Slack (Socket Mode)、飞书 (WebSocket)、微信 (iLink 长轮询)、企业微信 (WebSocket)、钉钉 (Stream Push) | 用户可在 6 个主流 IM 平台上直接与 Agent 对话，无需打开 Web UI；所有通道均为 Outbound Connection | 全部 Outbound Connection—无需公网 IP，企业内网即可部署；`/new` `/status` `/models` `/memory` `/help` 统一命令集；channels 配置在 `config.example.yaml` 中为注释模板，需手动取消注释并配置后方可启用 | `README.md` § IM Channels; `CLAUDE.md` § IM Channels System; `config.example.yaml` § channels (注释配置模板) |
| 飞书流式卡片 | 单卡片 `config.update_multi=true` 原地流式更新 | 用户在飞书中看到 Agent 回复像打字机一样逐字出现，而非等待完整回复后一次性显示 | 存储 `message_id` per inbound，持续 patch 同一卡片；`runs.stream(["messages-tuple","values"])` 增量推送 | `CLAUDE.md` § IM Channels System (feishu.py); `README.md` § IM Channels |
| 钉钉 AI Card | `card_template_id` → `PUT /v1.0/card/streaming` 打字机效果 | 钉钉用户看到流式 AI Card 打字机效果；需申请 `Card.Streaming.Write` 和 `Card.Instance.Write` 权限 | 可选 `sampleMarkdown` fallback—card 创建或流式失败时自动降级为纯文本 | `README.md` § DingTalk Setup; `config.example.yaml` § channels.dingtalk |
| 微信自举登录 | QR Code 模式: 无 token 时自动生成 QR，用户扫码绑定 | 首次部署时无需手动申请微信 Token—扫码后自动获取并持久化，后续重启直接复用 | `state_dir` 持久化 token 和 `get_updates_buf` cursor；Docker 部署需挂载持久卷 | `README.md` § WeChat Setup; `config.example.yaml` § channels.wechat |
| Claude Code 互操作 | `claude-to-deerflow` Skill — `/claude-to-deerflow` 命令直接交互 | 开发者可在 Claude Code 终端中直接向 DeerFlow 发送研究任务、查看状态、管理线程 | 支持 flash/standard/pro/ultra 四种执行模式；通过 `npx skills add` 一键安装 | `README.md` § Claude Code Integration; `skills/public/claude-to-deerflow/SKILL.md` |
| Message Bus | `message_bus.py` — async pub/sub (`InboundMessage` → queue → dispatcher) | 新增 IM 通道时只需实现 Channel 基类并注册，消息路由和线程管理由 Bus 自动处理 | Channel 实现与消息路由完全解耦；异步队列确保消息不丢失 | `CLAUDE.md` § IM Channels System (message_bus.py, manager.py, base.py) |

### 2.5 安全与隔离维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| 四层 Sandbox | `LocalSandboxProvider` (线程目录) / `AioSandboxProvider` + Apple Container (macOS ARM64 原生) / `AioSandboxProvider` + Docker (容器) / `AioSandboxProvider` + Provisioner (K3s Pod) | 开发用 Local（零依赖），macOS 生产环境自动检测 Apple Container（无 Rosetta 损耗），Linux/Windows 用 Docker，多租户切 K8s | Apple Container 使用 macOS Virtualization.framework 原生 ARM64 执行，AioSandboxProvider 自动检测可用运行时并 fallback；四种模式共享同一 `Sandbox` 抽象接口 | `APPLE_CONTAINER.md`; `config.example.yaml` § sandbox; `CLAUDE.md` § Sandbox System |
| Guardrails 授权 | `GuardrailMiddleware` (#6) — 可插拔 Provider: `AllowlistProvider` / OAP Passport / Custom | 管理员可禁止 Agent 调用 `bash` 或 `write_file`（Allowlist），或基于 OAP Passport JSON 声明 Agent 的身份和权限边界进行策略评估 | 三种 Provider 覆盖零依赖到企业策略；OAP 是开放标准，可与任意 OAP-compliant 授权服务集成 | `backend/docs/GUARDRAILS.md`; `config.example.yaml` § guardrails; `CLAUDE.md` § Middleware Chain #6 |
| Sandbox 审计 | `SandboxAuditMiddleware` (#7) — 审计沙箱 Shell/文件操作 | 每次 Shell 命令执行和文件操作均被记录，可用于事后安全审计和异常行为分析 | 独立中间件，置于 GuardrailMiddleware 之后—先授权再审计 | `CLAUDE.md` § Middleware Chain #7 |
| XSS 防御 | HTML/SVG 等 Active Content 强制 `Content-Disposition: attachment` | Agent 生成的 HTML/SVG 产物不会被浏览器内联渲染执行，防止恶意脚本注入 | Gateway 级别过滤—不是 Trust on First Use；`?download=true` 仍可触发下载 | `CLAUDE.md` § Gateway API Artifacts; `README.md` § Security Notice |
| 配置版本管理 | `config_version` 字段 — 启动时检测过期配置，`make config-upgrade` 自动合并 | 升级 DeerFlow 后不会因配置 Schema 变更导致启动失败—自动检测并提示用户升级，保留现有值 | 企业级配置漂移防护；`.bak` 备份确保可回滚 | `CLAUDE.md` § Configuration System; `config.example.yaml` § config_version |
| Circuit Breaker | 可配置的 LLM 调用熔断器 (`failure_threshold`, `recovery_timeout_sec`) | 当 LLM Provider 连续失败达到阈值后自动熔断，后续调用快速失败，防止资源耗尽和 Rate Limit 封禁 | 配置项存在于 `config.example.yaml` 中，但**默认被注释** (`# circuit_breaker:`)，需手动取消注释并配置后方可启用 | `config.example.yaml` § circuit_breaker (注释配置)
| 认证集成 | IM Channel 内部 auth + CSRF cookie/header pair | IM 通道 worker 无需浏览器 cookie 即可发起 state-changing 请求（线程创建、运行管理） | 内部 auth 注入 + CSRF pair 确保 Gateway 安全策略不被 IM 通道绕过 | `CLAUDE.md` § IM Channels System (Architecture) |

### 2.6 持久化与状态管理维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| 跨会话 Memory | LLM 驱动的异步 Memory 更新 — debounced queue (30s) → 事实提取 → 去重 → atomic write | 用户关闭浏览器后再打开，Agent 仍记得用户的姓名、技术栈偏好、工作背景等；重复偏好不会因多次提及而重复存储 | `fact_confidence_threshold: 0.7` 过滤低质量提取；`max_injection_tokens: 2000` 控制 System Prompt 膨胀 | `CLAUDE.md` § Memory System; `config.example.yaml` § memory; `memory/updater.py` |
| Thread 隔离 | `ThreadDataMiddleware` (#1) — `{base_dir}/users/{uid}/threads/{tid}/` 三级目录结构 | 不同会话的 Agent 文件操作完全隔离，互不干扰；删除 Thread 时自动清理本地线程目录 | `user_id` 通过 ContextVar 传递；无认证模式 fallback 为 `"default"` 用户 | `CLAUDE.md` § Middleware Chain #1; `thread_state.py` |
| State Reducer | `merge_artifacts` (dict.fromkeys 去重保持顺序) + `merge_viewed_images` (空字典清除语义) | 多次工具调用产生的 artifacts 自动去重合并保持顺序；一轮图片处理完成后可批量清除 viewed_images | LangGraph `Annotated` 模式—Reducer 函数由框架在每个状态更新时自动调用 | `thread_state.py` (merge_artifacts, merge_viewed_images) |
| 双数据库 Backend | SQLite (WAL, 单节点) / PostgreSQL (生产多节点) | 单机部署零配置（自动创建 deerflow.db），扩展到多节点只需改 `DATABASE_URL` 指向 PostgreSQL | 同一 deerflow.db 文件同时用于 LangGraph checkpointer + DeerFlow app data（runs/feedback/events） | `config.example.yaml` § database; `CLAUDE.md` § Configuration System |
| Run Events 持久化 | `memory` / `db` / `jsonl` 三种 backend | 开发者可选择消息和执行轨迹的持久化级别：memory（调试用/重启丢失）、jsonl（单节点追加持久化）、db（结构化查询） | `track_token_usage: true` 可累积每次运行的 Token 计数至 RunRow | `config.example.yaml` § run_events |
| 反馈系统 | `PUT/DELETE/POST/GET /api/threads/{id}/runs/{rid}/feedback` — 含 `GET /stats` 聚合 | 用户可对 Agent 的每次运行点赞/点踩/评分，管理员可通过 `/stats` 查看全局反馈统计 | 完整 CRUD + 聚合统计；feedback 数据存储在统一持久化层 | `CLAUDE.md` § Gateway API Routers (Feedback) |

### 2.7 工程化与可观测维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| 双追踪 Provider | LangSmith + Langfuse 可同时启用 | 开发者可在 LangSmith 中查看详细的 Agent 执行轨迹（每次 LLM 调用、工具调用、中间件拦截），同时在 Langfuse 中做成本分析 | 相同模型活动同时上报至两个系统—LangSmith 用于调试，Langfuse 用于成本监控 | `README.md` § LangSmith/Langfuse Tracing |
| Token Usage 追踪 | `TokenUsageMiddleware` (#11) — 记录每次调用的 input/output/total | 用户可在 Web UI 和 API 中查看每个 Thread 的 Token 消耗总量，用于成本核算和优化 | `GET /api/threads/{id}/token-usage` 端点暴露聚合数据；条件启用 (`token_usage.enabled`) | `CLAUDE.md` § Middleware Chain #11; `config.example.yaml` § token_usage |
| 自动标题生成 | `TitleMiddleware` (#12) — LLM 驱动的首轮后标题生成 | 用户的历史会话列表自动显示语义化标题（如"DeerFlow 架构分析"），而非"Untitled Thread #42" | 可配置 max_words/max_chars/prompt_template；首轮完整交换后才触发 | `CLAUDE.md` § Middleware Chain #12; `config.example.yaml` § title |
| 配置热重载 | `get_app_config()` — mtime 检测自动重载 config.yaml | 修改 `config.yaml` 添加新模型或调整参数后，无需重启 Gateway 即可生效 | Cache 失效策略：仅在 config 文件路径变化或 mtime 更新时重新读取 | `CLAUDE.md` § Configuration System (Config Caching) |
| 依赖防火墙 | `tests/test_harness_boundary.py` — CI 强制执行 Harness → App 单向依赖 | 任何开发者试图在 Harness 包中 `import app.*` 都会触发 CI 失败，防止架构腐化 | CI 级别架构约束—非文档约定，无法绕过 | `CLAUDE.md` § Harness/App Split; `tests/test_harness_boundary.py` |
| Gateway Conformance | `TestGatewayConformance` — 验证 Embedded Client 输出与 Gateway Pydantic 模型一致 | 使用 `DeerFlowClient`（嵌入式）和 HTTP Gateway API 的代码行为完全一致，可互换 | Pydantic 自动验证—Gateway 新增 required field 时，Client 测试立即失败，CI 捕获漂移 | `CLAUDE.md` § Embedded Client; `tests/test_client.py` |
| 启动向导 | `make setup` — 交互式配置 LLM Provider、搜索 Provider、Sandbox 模式 | 新用户 2 分钟内完成从 clone 到首次运行，无需手动编辑 YAML | 生成最小化 `config.yaml` + `.env`；`make doctor` 诊断配置问题 | `README.md` § Configuration; Issue #2030 (RFC: Setup Wizard) |
| 流式输出双路径 | Gateway SSE 路径 (async+Queue+JSON+RunManager+Last-Event-ID 断连恢复) 与 Embedded Client 路径 (sync generator+原生 LangChain 对象) 并行 | 浏览器/IM 渠道享受 HTTP SSE 流式输出和断连恢复；Jupyter/脚本享受零基础设施的同步 generator 调用 | 刻意不合并—两条路径服务不同消费者模型；`seen_ids`/`streamed_ids`/`counted_usage_ids` 三组独立去重不变式 | `backend/docs/STREAMING.md` § 为什么有两条流式路径; `client.py` |

### 2.8 扩展性与生态维度

| 原子功能 | 技术实现方案 | 具体效果 | 独特性/发现 | 资料来源 |
|---------|-------------|---------|-----------|---------|
| Skills 系统 | Markdown 驱动的渐进式加载能力模块 — `public/` + `custom/` 双目录 | 社区贡献者只需写一个 `SKILL.md` 文件即可为 DeerFlow 添加新能力（如 PDF 解析、数据库查询）；用户通过 API 一键安装 `.skill` 压缩包 | Public 目录 Git 跟踪供社区共享，Custom 目录 gitignore 保护私有 Skill | `CLAUDE.md` § Skills System; `skills/types.py`; `README.md` § Skills & Tools |
| MCP 集成 | `langchain-mcp-adapters` — stdio/SSE/HTTP 三传输 | 一键接入任何 MCP Server（GitHub API、文件系统、数据库等），工具自动注册到 Agent | OAuth token flow 支持 (client_credentials/refresh_token)；cache 按 mtime 自动失效 | `CLAUDE.md` § MCP System; `config.example.yaml` § extensions_config.json |
| Deferred Tool Loading | 工具 Schema 延迟加载，`tool_search` 按需发现 | 当 MCP Server 暴露 50+ 工具时，Agent 的 System Prompt 仅列工具名，按需加载 Schema，避免上下文窗口被工具定义占满 | `DeferredToolFilterMiddleware` (#15) — 仅当 `tool_search.enabled: true` 时激活 | `CLAUDE.md` § Middleware Chain #15; `config.example.yaml` § tool_search |
| ACP Agent | 外部 Agent 进程通过 ACP 协议集成 | DeerFlow 可调用 Claude Code 或 Codex CLI 作为外挂编码 Agent，实现"主 Agent 分配任务 → 专业 Agent 执行 → 结果回传"的异构 Agent 协作 | ACP 协议 (`agent-client-protocol>=0.4.0`) 独立于内置 Sub-Agent 体系—可接入任何 ACP-compliant Agent | `config.example.yaml` § acp_agents; `CLAUDE.md` § ACP Agent Tools; `pyproject.toml` |
| Skill Evolution | `skill_evolution.enabled` — Agent 自主创建/改进 Skills | Agent 发现某个工作流重复执行后，可自动提取为可复用的 Skill 写入 `skills/custom/` | 含 LLM 安全扫描 (`moderation_model_name`)；默认关闭需手动开启 | `config.example.yaml` § skill_evolution |
| 自定义 Sub-Agent | `custom_agents` — 独立 system_prompt / tools 白名单 / skills 白名单 / model 覆盖 | 管理员可定义专门 Agent 类型（如"数据分析师"用便宜模型、"代码审查员"用强模型 + bash 工具） | 每个 Agent 独立 timeout 和 max_turns；Model 可设为 `inherit` 或指定其他模型名 | `config.example.yaml` § subagents.custom_agents |
| 反射式 Provider 加载 | `resolve_variable(path)` / `resolve_class(path, base_class)` | 模型/Tools/Guardrails/Sandbox/ACP 全部通过 `config.yaml` 中的字符串路径动态加载，无需修改 DeerFlow 源码即可接入第三方实现 | 用户自定义 Guardrail Provider 只需实现 `evaluate/aevaluate` 接口并在 YAML 中声明路径 | `CLAUDE.md` § Reflection System; `reflection/__init__.py` |
| 社区 Web Search | DuckDuckGo / Tavily / Serper / Exa / Firecrawl / InfoQuest — 6 种 Provider | 用户可根据价格、延迟、结果质量自由选择搜索引擎；DuckDuckGo 零配置开箱即用 | InfoQuest 为 ByteDance 自研，支持 `search_time_range` 时间过滤；Serper 为最新合并的社区贡献 (PR #2630) | `config.example.yaml` § tools; `CLAUDE.md` § Community Tools; PR #2630 |

> **依赖分析补充**: `pyproject.toml` 共声明 30+ 个直接依赖 + 3 组可选依赖。引入 `kubernetes>=30` 说明已为 K8s 级部署做好准备；引入 `duckdb>=1.4.4` 暗示潜在的嵌入式分析能力；`agent-client-protocol>=0.4.0` 确认了跨 Agent 通信的协议层投入。

---

### 关键业务流程图 (原图)

> **Source**: `backend/docs/ARCHITECTURE.md` § Request Flow, File Upload Flow, Thread Cleanup Flow, Configuration Reload

#### Request Flow — Agent 消息处理全链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    User sends message to agent                           │
└─────────────────────────────────────────────────────────────────────────┘

1. Client → Nginx
   POST /api/langgraph/threads/{thread_id}/runs
   {"input": {"messages": [{"role": "user", "content": "Hello"}]}}

2. Nginx → LangGraph Server (2024)
   Proxied to LangGraph server

3. LangGraph Server
   a. Load/create thread state
   b. Execute middleware chain:
      - ThreadDataMiddleware: Set up paths
      - UploadsMiddleware: Inject file list
      - SandboxMiddleware: Acquire sandbox
      - SummarizationMiddleware: Check token limits
      - TitleMiddleware: Generate title if needed
      - TodoListMiddleware: Load todos (if plan mode)
      - ViewImageMiddleware: Process images
      - ClarificationMiddleware: Check for clarifications

   c. Execute agent:
      - Model processes messages
      - May call tools (bash, web_search, etc.)
      - Tools execute via sandbox
      - Results added to messages

   d. Stream response via SSE

4. Client receives streaming response
```

#### File Upload Flow — 文件上传与文档转换

```
1. Client uploads file
   POST /api/threads/{thread_id}/uploads
   Content-Type: multipart/form-data

2. Gateway receives file
   - Validates file
   - Stores in .deer-flow/threads/{thread_id}/user-data/uploads/
   - If document: converts to Markdown via markitdown

3. Returns response
   {
     "files": [{
       "filename": "doc.pdf",
       "path": ".deer-flow/.../uploads/doc.pdf",
       "virtual_path": "/mnt/user-data/uploads/doc.pdf",
       "artifact_url": "/api/threads/.../artifacts/mnt/.../doc.pdf"
     }]
   }

4. Next agent run
   - UploadsMiddleware lists files
   - Injects file list into messages
   - Agent can access via virtual_path
```

#### Thread Cleanup Flow — 对话删除

```
1. Client deletes conversation via LangGraph
   DELETE /api/langgraph/threads/{thread_id}

2. Web UI follows up with Gateway cleanup
   DELETE /api/threads/{thread_id}

3. Gateway removes local DeerFlow-managed files
   - Deletes .deer-flow/threads/{thread_id}/ recursively
   - Missing directories are treated as a no-op
   - Invalid thread IDs are rejected before filesystem access
```

#### Configuration Reload — 配置热更新

```
1. Client updates MCP config
   PUT /api/mcp/config

2. Gateway writes extensions_config.json
   - Updates mcpServers section
   - File mtime changes

3. MCP Manager detects change
   - get_cached_mcp_tools() checks mtime
   - If changed: reinitializes MCP client
   - Loads updated server configurations

4. Next agent run uses new tools
```

### Guardrails 架构图 (原图)

> **Source**: `backend/docs/GUARDRAILS.md` § Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Middleware Chain                               │
│                                                                      │
│  1-4. (Thread, Upload, Sandbox, Dangling)                            │
│  5. GuardrailMiddleware ◄──── EVALUATES EVERY TOOL CALL             │
│  6-12. (ToolError, Summarization, Title, Memory, Vision, Subagent)   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
           ┌──────────────────────────┐
           │    GuardrailProvider     │  ◄── pluggable: any class
           │    (configured in YAML)  │      with evaluate/aevaluate
           └────────────┬─────────────┘
                        │
              ┌─────────┼──────────────┐
              │         │              │
              ▼         ▼              ▼
         Built-in   OAP Passport    Custom
         Allowlist  Provider        Provider
         (zero dep) (open standard) (your code)
                        │
                  Any implementation
                  (e.g. APort, or
                   your own evaluator)
```

```
Without guardrails:                      With guardrails:

  Agent                                    Agent
    │                                        │
    ▼                                        ▼
  ┌──────────┐                             ┌──────────┐
  │ bash     │──▶ executes immediately     │ bash     │──▶ GuardrailMiddleware
  │ rm -rf / │                             │ rm -rf / │        │
  └──────────┘                             └──────────┘        ▼
                                                         ┌──────────────┐
                                                         │  Provider    │
                                                         │  evaluates   │
                                                         │  against     │
                                                         │  policy      │
                                                         └──────┬───────┘
                                                                │
                                                          ┌─────┴─────┐
                                                          │           │
                                                        ALLOW       DENY
                                                          │           │
                                                          ▼           ▼
                                                      Tool runs   Agent sees:
                                                      normally    "Guardrail denied:
                                                                   rm -rf blocked"
```

---

## 三、深度技术链路验证 (Trace & Log Analysis)

### 3.1 场景还原: Sub-Agent 并行研究任务

**任务描述**: 主 Agent 收到"研究 AI Agent 安全领域的三个子课题"请求，分发至 3 个 Sub-Agent 并行执行。

**还原执行路径** (基于 `SubagentExecutor` 791 行源码):

```
Lead Agent (lead_agent/agent.py)
  │
  ├─ receive_message → apply_prompt_template (注入 Skills + Memory)
  │
  ├─ LLM Call (create_chat_model → PatchedChatDeepSeek)
  │   └─ 返回 3 个 task() tool calls
  │
  ├─ SubagentLimitMiddleware (#16) — 检查 MAX_CONCURRENT_SUBAGENTS=3 ✓
  │
  ├─ GuardrailMiddleware (#6) — 授权 task() 调用 ✓
  │
  ├─ SubagentExecutor.submit()
  │   ├─ _submit_to_isolated_loop_in_context()
  │   │   └─ contextvars.Context.copy_context() — 保留 user_id
  │   │
  │   ├─ _scheduler_pool (3 workers) — 分配至线程池
  │   │
  │   └─ 每个 Sub-Agent:
  │       ├─ 独立 ThreadDataMiddleware — 创建隔离目录
  │       ├─ 独立 SandboxMiddleware — 获取沙箱
  │       ├─ 独立 Skills 注入 (按 custom_agents 白名单过滤)
  │       ├─ 独立 LLM 实例 (可继承主模型，可覆盖)
  │       └─ 独立 Middleware 管道 (不含 ClarificationMiddleware)
  │
  ├─ 轮询 (5s interval, 900s timeout)
  │   └─ SSE events: task_started → task_running → task_completed
  │
  └─ Lead Agent 合成结果 → present_files → 返回最终报告
```

> **Source**: `backend/packages/harness/deerflow/subagents/executor.py` 791 行完整实现

### 3.2 异常恢复路径: Dangling Tool-Call Recovery

**场景**: 用户在推理模型执行 tool-call 循环中发送了新消息（中断），导致部分 tool_calls 缺少响应。

```
1. DanglingToolCallMiddleware (#4) 检测 AIMessage.tool_calls
2. 扫描对话历史，找出缺少 ToolMessage 响应的 tool_call_id
3. 剥离 raw provider tool-call metadata (additional_kwargs["tool_calls"])
4. 注入占位 ToolMessage(content="Tool call was interrupted...")
5. 确保下一次 LLM 调用时 tool_call_id 序列完整
6. 推理模型(如 OpenAI o1)继续正常执行，而非报错 400 "missing tool_call_id"
```

> **Source**: `CLAUDE.md` § Middleware Chain #4 — DanglingToolCallMiddleware

**Harness Engineer 评注**: 这是"吸收 Provider 复杂性"的典型案例。标准做法是让用户避免中断或换用非推理模型。DeerFlow 选择在 Harness 层透明修复，体现了 Harness 的核心职责——让 Agent 开发者不用关心底层模型的怪癖。

### 3.3 资源表现推断

| 指标 | 默认配置值 | 可配置性 |
|------|----------|---------|
| Sub-Agent 超时 | 900s (15min) | 全局 + per-agent 覆盖 |
| 并行度上限 | 3 | 硬编码，SubagentLimitMiddleware 截断 |
| Token 压缩阈值 | 15,564 tokens | `summarization.trigger.value` |
| Skills 保留 | 最近 5 个 / 25k tokens | 完全可配置 |
| Memory debounce | 30s | `memory.debounce_seconds` |
| bash 输出截断 | 20,000 chars | 可配置为 0 (不截断) |
| 文件读取截断 | 50,000 chars | 可配置 |

> **注**: 以上为静态配置分析。实际运行时消耗（内存、CPU、延迟）需在部署环境中进行黑盒压测获取。

---

## 四、开发者与工程化指标 (Engineering Excellence)

### 4.1 Skill 开发便捷度

| 维度 | 评估 | 依据 |
|------|------|------|
| **Skill 定义格式** | Markdown + YAML frontmatter | `skills/types.py` — `Skill` dataclass |
| **Skill 安装方式** | `.skill` ZIP 压缩包通过 Gateway API 上传 | `POST /api/skills/install` |
| **Skill 组织** | 双目录 (`public/` + `custom/`) | Git 跟踪 + .gitignore 隔离 |
| **代码行数** | 一个 Skill 最少仅需一个 `SKILL.md` 文件 | 目录结构分析 |
| **渐进式加载** | 仅任务需要时加载，非全部注入 System Prompt | `README.md` § Skills & Tools |
| **受限程度** | Custom Sub-Agent 可设 Skills 白名单/黑名单 | `config.example.yaml` § custom_agents |

**评分**: ★★★★☆ — Markdown 定义门槛极低，但缺少 Skill 开发调试工具（无 Skill 测试框架）。

### 4.2 测试基础设施

| 维度 | 评估 | 依据 |
|------|------|------|
| **测试框架** | pytest | `backend/tests/conftest.py` |
| **架构约束测试** | `test_harness_boundary.py` — CI 强制执行 | `CLAUDE.md` § Harness/App Split |
| **类型一致性测试** | `TestGatewayConformance` — 验证 Embedded Client vs Gateway 模型一致性 | `CLAUDE.md` § Embedded Client |
| **Sandbox 测试** | `test_docker_sandbox_mode_detection.py`, `test_provisioner_kubeconfig.py` | `CLAUDE.md` § Commands |
| **Memory 测试** | `test_memory_updater.py` — 聚焦回归覆盖 | `CLAUDE.md` § Memory System |
| **集成/E2E 测试** | `Playwright E2E tests with CI workflow` — PR #2279 | 最近合并 |
| **TDD 强制** | "Every new feature or bug fix MUST be accompanied by unit tests" | `CLAUDE.md` § TDD |
| **Q2 计划** | "Sophisticated unit tests / integration tests / e2e tests" 列为 Core Stability 子任务 | Issue #1669 |

**评分**: ★★★★☆ — 有严格的 TDD 纪律和 CI 架构约束，但 Benchmark 仍标记为未完成。

### 4.3 可观测性

| 维度 | 支持情况 | 局限性 |
|------|---------|--------|
| **分布式追踪** | LangSmith + Langfuse (双 Provider 可同时) | 无 OpenTelemetry 原生支持 |
| **Token 监控** | `TokenUsageMiddleware` + Gateway API 端点 | 依赖 Provider 返回 usage 字段 |
| **Run Events** | memory/jsonl/db 三种 backend | 默认 memory (重启丢失) |
| **结构化日志** | Python logging (可配 log_level) | 无内置 Prometheus metrics 端点 |
| **Circuit Breaker** | 配置存在但默认未启用 | 无开箱即用的告警集成 |

**评分**: ★★★☆☆ — 基础能力完备，但离企业级观测性（Grafana/Prometheus/OTel）有距离。

---

## 五、待探索/缺失能力 (Gap Analysis)

### 5.1 社区痛点梳理 (Issue 数据分析)

| 痛点 | Issue 编号 | 优先级信号 | 当前状态 |
|------|----------|----------|---------|
| **多租户分级权限管理** | #2318 | `enhancement` + `help wanted` + 6 comments | Open |
| **并发用户请求支持** | #1669 Roadmap 子任务 | Core Stability 🔥🔥🔥🔥🔥 | 计划中 |
| **Benchmark 体系** | #1669 Roadmap 子任务 | Core Stability 🔥🔥🔥🔥🔥 | 待启动 |
| **Cron 调度器** | #1092 / #1669 Roadmap | Channel Integration 🔥🔥🔥 | 关联 PR 已提交 |
| **统一的持久化层** | #1851 | 关联 PR | 最近合并 |

### 5.1.1 补充核查：多租户/Memory/沙箱生命周期

| 核查项 | 现状 | 源码出处 |
|--------|------|---------|
| 多租户隔离 | `users/{uid}/threads/{tid}/` 三级目录 + `get_effective_user_id()`。当前无 tenant 概念，社区 #2318 追踪中。K3s Pod 模式面向多租户场景但权限管理待开发 | `CLAUDE.md` § ThreadDataMiddleware；Issue #2318 |
| Memory 按 user 隔离 | MemoryMiddleware 过滤 "user + final AI responses"；`users/{uid}/` 路径前缀天然隔离不同 user 的 Memory 存储 | `CLAUDE.md` § Middleware Chain #13 |
| 沙箱生命周期 | **已知产品中最完整**：`acquire(thread_id)` → `get(sandbox_id)` → `release(sandbox_id)` → `shutdown()`；`lazy_init=True` 延迟创建；同 Thread 多轮复用；仅在 app shutdown 时统一清理 | `sandbox/middleware.py` L37-75 |
| 外置沙箱接入 | Docker: 配置已运行 daemon；K8s: 配置 Provisioner + kubeconfig；Apple Container: macOS Virtualization.framework 自动检测 | `CLAUDE.md` § Sandbox System；`APPLE_CONTAINER.md` |
| 评测体系 | Roadmap § Benchmark 体系 🔥🔥🔥🔥🔥（待启动） | Issue #1669 |

### 5.2 代码架构中暴露的缺失

| 缺失项 | 检测来源 | 影响 |
|--------|---------|------|
| **Agent 间直接通信** | Sub-Agent 无 message passing 接口 | Lead Agent 是唯一信息汇聚点 |
| **Network Egress Control** | Sandbox 无网络策略配置 | Docker 容器可自由 curl 外部 |
| **Prompt Versioning** | 无 Prompt 版本管理机制 | 难以追踪 Prompt 变更对行为的影响 |
| **Skill 测试框架** | 无 Skill 级别的测试支持 | 自定义 Skill 质量无法自动化验证 |
| **多语言 SDK** | Client 仅 Python | 限制了非 Python 生态的嵌入使用 |
| **Horizontal Scaling** | 无原生负载均衡/集群管理 | 单实例部署为主 |
| **Memory 智能检索** | 当前仅按 confidence 降序排序注入事实 | 官方 Roadmap 已规划 TF-IDF 相似度检索 (`final_score = similarity*0.6 + confidence*0.4`)，待合并至 main |

### 5.3 未覆盖场景

| 场景 | 原因 |
|------|------|
| 图形化桌面应用操作 | 无 GUI 自动化工具 (如 Playwright/Puppeteer)，仅 Web/CLI |
| 实时音视频处理 | 无音视频流工具集成 |
| 硬件/IoT 设备控制 | 无 GPIO/USB/串口等底层接口 |
| 移动端原生操作 | IM 通道仅为消息收发，不支持设备级控制 |
| 多 Agent 民主协商 | Sub-Agent 架构为树形 (Lead → Workers)，非 P2P 拓扑 |

---

## 六、维度建议 (Dimension Evolution)

> 核心价值: 基于 DeerFlow 2.0 的深度分析，建议在横向评估标准中增加以下新维度。

### 6.1 由 DeerFlow 独有特性推导的新维度

| 建议维度 | 触发特征 | 定义 | 行业前瞻性 |
|---------|---------|------|----------|
| **Harness/App 架构分离度** | `packages/harness/` 与 `app/` 的 CI 级单向依赖约束 | 框架核心是否可独立于应用层发布和复用 | 高 — 这是 Agent 基础设施作为独立产品的必要条件 |
| **Provider 复杂度吸收能力** | `PatchedChatDeepSeek`, `PatchedChatOpenAI` 等透明修复模式 | Harness 是否在框架层修复 Provider 差异，而非将复杂性传递给用户 | 高 — 模型层差异是 Agent 落地的主要障碍 |
| **Context Engineering 成熟度** | Skills 保留策略 + Summarization + Dangling Recovery 三位一体 | 系统是否具备多维度的长程上下文管理能力 | 高 — 长程任务是 Agent 区别于 Chatbot 的核心标志 |
| **Sandbox 抽象层次** | `LocalSandboxProvider` / `AioSandboxProvider` / Apple Container / Provisioner 同一接口 | 执行隔离是否从本地到 K8s 提供统一抽象 | 中 — 安全隔离已是业界共识，但统一抽象仍少见 |
| **Agent vs Agent 平台判定** | 模板化 (Skill/SOUL/YAML) + 隔离化 (四层 Sandbox + Thread 隔离) + "API 调用模板 ID → 生成隔离专业实例" | 系统是"单 Agent 工具"还是"Agent 运行时平台" | 高 — 直接决定选型路径: 快速验证 vs 企业基础设施 |

### 6.2 由 DeerFlow 与行业共同短板推导的维度

| 建议维度 | 理由 |
|---------|------|
| **Agent 对等通信能力** | DeerFlow/CrewAI/AutoGen 均为中心化编排，无 P2P Agent 通信 |
| **Prompt 可观测性** | 所有产品均缺乏 Prompt 版本的 diff/blame/rollback 能力 |
| **多语言 SDK 覆盖** | 除 OpenAI SDK 外，主流 OS Agent 框架均为 Python 单语言 |
| **Agent Evaluation Framework** | 无产品内置标准化的 Agent 任务成功率 Benchmark |

### 6.3 确定性维度: Agent vs Agent 平台 (Harness/Runtime)

> 开源项目洞察和选型时，"Agent" 关注个体能力强弱，"Agent 平台" 关注生产力的规模化与安全性。
> 核心判定标准: **模板化 + 隔离化 = 解耦的生产力配置与标准化的运行环境约束**。

#### 判定定义

| 判定标准 | 定义 | 判断依据 |
|---------|------|---------|
| **模板化 (Templatization)** | 能力的可治理性 — 通过非代码形式 (Markdown/YAML) 声明 Agent 的技能、人设和权限 | 系统是否支持"定义一次角色，多次实例化启动"？是否能通过配置文件热插拔 Agent 功能？ |
| **隔离化 (Isolation)** | 系统的健壮性 — 为每个任务实例提供物理或逻辑上的独立执行空间 | 当 Agent 执行有害代码或处理超长上下文时，是否会影响宿主机或其他 Agent 实例？ |

#### DeerFlow 2.0 的平台判定映射

| 判定标准 | DeerFlow 实现 | 源码证据 |
|---------|-------------|---------|
| **模板化 — Skill 定义** | `SKILL.md` (Markdown + YAML frontmatter) 声明能力 | `skills/types.py` — `Skill` dataclass；`skills/public/*/SKILL.md` |
| **模板化 — Agent 人设** | Custom Agent SOUL/USER.md + `custom_agents` YAML 配置 | `config.example.yaml` § agents_api, subagents.custom_agents |
| **模板化 — 权限声明** | Guardrails Allowlist/OAP Passport (JSON/YAML 声明权限边界) | `GUARDRAILS.md` § OAP Passport JSON 示例 |
| **模板化 — 热插拔** | `extensions_config.json` mtime 检测自动重载；Skills 通过 API 安装/启用 | `CLAUDE.md` § Config Caching, Skills System |
| **模板化 — 一次定义多次实例化** | `custom_agents` 定义 Agent 类型 → `task(type="agent-name")` 动态实例化 | `config.example.yaml` § custom_agents；`subagents/executor.py` |
| **隔离化 — 执行空间** | 四层 Sandbox: Local (线程目录) / Apple Container / Docker / K8s Pod | `APPLE_CONTAINER.md`; `config.example.yaml` § sandbox |
| **隔离化 — 文件系统** | 每 Thread 独立 `{base_dir}/users/{uid}/threads/{tid}/` 目录树 | `CLAUDE.md` § ThreadDataMiddleware |
| **隔离化 — Sub-Agent 上下文** | 每个 Sub-Agent 独立上下文，不可见 Lead Agent 或其他 Sub-Agent | `CLAUDE.md` § Context Engineering — "Isolated Sub-Agent Context" |
| **隔离化 — 资源限制** | Sub-Agent 超时 (900s)、SubagentLimitMiddleware (MAX=3)、输出截断 | `subagents/executor.py`; `CLAUDE.md` § Middleware #16 |

#### 选型对比表: Agent vs Agent 平台

| 维度 | 单 Agent 项目 (如 AutoGPT) | Agent 平台/Harness (如 DeerFlow) | 选型决策影响 |
|------|--------------------------|--------------------------------|-------------|
| **部署形态** | "一处部署，一处使用" | "中心化部署，多租户分发" | 决定维护成本 (O&M) |
| **逻辑定义** | 硬编码在 Python/TS 逻辑中 | 声明式模板 (Skill/SOUL/YAML) | 决定非技术人员的参与度 |
| **任务边界** | 共享本地环境，无物理隔离 | 独立容器沙箱隔离 | **核心选型点**: 决定是否敢让它操作真实系统/文件 |
| **扩展方式** | 修改源代码 | 动态加载 Skill 模板 + MCP/ACP 协议 | 决定业务逻辑更新的灵活性 |
| **协作模式** | 单一 Loop 循环 | 多层级 (Lead-Sub) 编排 | 决定能否处理跨领域的超长任务 |
| **版本管理** | Git 管理代码版本 | 模板 ID + config_version + Skill 版本 | 决定 Prompt/Skill 的可审计性 |
| **安全授权** | 无或简单 if-else | 可插拔 Guardrails (Allowlist/OAP/Custom) | 决定能否通过企业安全审计 |

#### DeerFlow 的"平台级"特征总结

DeerFlow 2.0 满足平台级判定的"一句话标准":

> **"通过 API 调用模板 ID 即可快速生成一个隔离的、具备特定专业技能的对话实例。"**

具体路径:
```
POST /api/langgraph/threads → 创建隔离 Thread
  + configurable.agent_name = "code-reviewer"  → 加载 custom_agents 模板
  + configurable.subagent_enabled = true       → 启用 Lead-Sub 编排
  + SandboxMiddleware                          → 自动获取隔离沙箱
  + GuardrailMiddleware                        → 工具调用前授权检查
```

这种架构解决了两个企业级痛点:
1. **AI 幻觉导致的代码安全** — Sandbox 隔离确保有害代码不影响宿主机
2. **提示词工程难以版本化管理** — Skill/SOUL 模板化使 Prompt 可版本控制、可审计、可热更新



---

## 附录 A: 信息来源索引

| 来源 | 路径/URL | 提取内容 |
|------|---------|---------|
| 项目 README | `README.md` | 官方定义、Quick Start、IM Channels 配置、Skills 结构 |
| 架构文档 | `backend/CLAUDE.md` | 完整架构图、18 层 Middleware、子系统设计、开发规范 |
| 架构详情 | `backend/docs/ARCHITECTURE.md` | 三进程架构 (LangGraph :2024 / Gateway :8001 / Frontend :3000)、Request Flow |
| 后端 README | `backend/README.md` | 架构 ASCII 图、组件列表、Gateway API 路由矩阵 |
| Apple Container | `backend/docs/APPLE_CONTAINER.md` | macOS ARM64 原生容器运行时、自动检测与 Fallback 机制 |
| 流式设计 | `backend/docs/STREAMING.md` | 双路径流式架构、seen_ids/streamed_ids/counted_usage_ids 去重不变式 |
| 依赖声明 | `backend/packages/harness/pyproject.toml` | 30+ 个直接依赖、3 组可选依赖、Python >=3.12 |
| Memory 改进 | `backend/docs/MEMORY_IMPROVEMENTS.md` | TF-IDF 相似度检索 Roadmap、`similarity*0.6+confidence*0.4` 加权策略 |
| 配置 Schema | `config.example.yaml` | config_version:8、models/sandbox/skills/memory/subagents/guardrails 完整配置 |
| 配置指南 | `backend/docs/CONFIGURATION.md` | 各 Provider 详细配置示例、Gemini thought_signature 补丁说明 |
| Guardrails 文档 | `backend/docs/GUARDRAILS.md` | 三种 Provider 架构图、OAP Passport JSON 示例 |
| Lead Agent | `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | make_lead_agent 工厂、_build_middlewares 组装逻辑 |
| ThreadState | `backend/packages/harness/deerflow/agents/thread_state.py` | State Schema + 自定义 Reducer |
| Subagent Executor | `backend/packages/harness/deerflow/subagents/executor.py` | 791 行 — 双线程池、ContextVar 保留、持久化 Event Loop |
| Sandbox Tools | `backend/packages/harness/deerflow/sandbox/tools.py` | 虚拟路径翻译、命令注入防御、输出截断策略 |
| Skills Types | `backend/packages/harness/deerflow/skills/types.py` | Skill Dataclass、SkillCategory 枚举 |
| Test Config | `backend/tests/conftest.py` | pytest fixtures、循环导入 mock、user context auto-set |
| Q2 Roadmap | GitHub Issue #1669 | Core Stability/Performance/Channel Integration 优先级 |
| 多租户 Issue | GitHub Issue #2318 | `enhancement` + `help wanted` — 社区高频需求 |
| 官方站点 | https://deerflow.tech | 在线 Demo 和产品页 |

---

## 附录 B: 术语表

| 术语 | 定义 |
|------|------|
| **Harness** | Agent 执行基础设施/运行时框架 — 不给 Agent 写逻辑，给 Agent 一台电脑 |
| **Lead Agent** | 主调度 Agent，负责任务分解和 Sub-Agent 编排 |
| **Middleware** | 中间件，在 Agent 执行管道中处理横切关注点 (18 层严格有序) |
| **Sandbox** | 隔离执行环境 — Local (线程目录) / Docker (容器) / K8s (Pod) |
| **Skill** | Markdown 驱动的原子能力模块，渐进式加载 |
| **MCP** | Model Context Protocol — 外部工具集成标准协议 |
| **ACP** | Agent Communication Protocol — 跨进程 Agent 通信协议 |
| **OAP** | Open Agent Passport — Agent 身份和权限声明开放标准 |
| **SSE** | Server-Sent Events — 服务端推送流式事件 |
| **Dangling Tool Call** | 缺少对应 ToolMessage 响应的 tool_call — Middleware #4 自动修复 |
| **ContextVar** | Python asyncio 上下文变量 — SubagentExecutor 跨线程保留 |

---

*本报告采用"特征逆向挖掘法"编制: 不预设评价终点，从源码目录结构 (Source Mining)、依赖分析、文档语义抽取和社区 Issue 痛点四个维度反向推导 DeerFlow 2.0 的能力原子图谱。每个结论均标注了官方源码出处，确保可追溯验证。*
