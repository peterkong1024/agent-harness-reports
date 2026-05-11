# Anthropic CMA (Managed Agents) API 资源模型解析

> 基于 anthropic-cma.yaml OpenAPI 3.1.0 规范解析
> 解析日期: 2026-05-11

---

## 0. CMA 概览

Claude Managed Agents (CMA) 是 Anthropic 在标准 Messages API 之上构建的 **Agent-as-a-Service 平台**。核心抽象层次:

```
CMA 管理面 API (v1/?beta=true)
  ├─ Agent 管理 (定义/版本/归档)
  ├─ Session 管理 (创建/事件流/SSE)
  ├─ Environment 管理 (沙箱/包/网络)
  ├─ Resource 管理 (文件/GitHub/Memory Store 挂载)
  ├─ Vault + Credential 管理 (密钥存储)
  ├─ Skill 管理 (上传/版本/分发)
  ├─ File 管理 (上传/下载/作用域)
  ├─ Memory Store 管理 (持久化KV存储/版本/审计)
  └─ User Profile 管理 (信任授权/OAuth注册)
```

## 1. 九大核心资源模型

### 1.1 Agent

**API路径**: `POST/GET /v1/agents?beta=true`, `POST /v1/agents/{agent_id}?beta=true`

```yaml
Agent:
  id: "agent_011CZkYpogX7uDKUyvBTophP"
  version: int (从1开始, update时递增)
  name: str (1-256字符)
  description: str (≤2048, nullable)
  model:
    id: "claude-sonnet-4-6"
    speed: "standard" | "fast"
  system: str (≤100000字符, system prompt)
  tools:                          # 工具配置(最大128)
    - agent_toolset_20260401:     # 内置工具集
        default_config:
          enabled: bool
          permission_policy:
            always_allow | always_ask
        configs:                  # per-tool覆盖
          - name: "bash" | "edit" | "read" | "write" | "glob" | "grep" | "web_fetch" | "web_search"
            enabled: bool
            permission_policy
    - mcp_toolset:               # MCP工具集
        mcp_server_name: str
        default_config
        configs
    - custom:                     # 自定义工具
        name: str
        description: str
        input_schema: JSONSchema
  mcp_servers:                    # MCP服务器(最大20)
    - type: "url"
      name: str
      url: str
  skills:                         # Skills(最大20)
    - type: "anthropic" | "custom"
      skill_id: str
      version: str
  metadata: map[str]str           # 最大16键
  created_at: timestamp
  updated_at: timestamp
  archived_at: timestamp|null
```

**关键设计决策**:
- Agent 有版本控制 (`version` 递增, CRUD含乐观锁)
- 支持内置工具 + MCP工具 + 自定义工具三层结构
- 权限策略 per-tool 粒度 (`always_allow` vs `always_ask`)
- 归档而非删除 (软删除)

### 1.2 Environment

**API路径**: `POST/GET /v1/environments?beta=true`, `POST/GET/DELETE /v1/environments/{id}?beta=true`

```yaml
Environment:
  id: "env_011CZkZ9X2dpNyB7HsEFoRfW"
  name: str (1-256)
  description: str (≤1024)
  config:
    cloud:                        # Anthropic Cloud
      networking:
        unrestricted | limited:
          allowed_hosts: [str]    # 白名单域名
          allow_package_managers: bool  # PyPI/npm等
          allow_mcp_servers: bool       # MCP出站
      packages:                   # 预装依赖
        pip: [str]   # "pandas==2.0.0"
        npm: [str]
        apt: [str]
        cargo: [str]
        gem: [str]
        go: [str]
  metadata: map[str]str
  archived_at: timestamp|null
```

**关键设计决策**:
- Environment = 沙箱规格（网络策略+预装包），非运行时实例
- 网络策略支持 `limited`(白名单) / `unrestricted`(全通)
- 包管理器维度覆盖 6 种语言生态

### 1.3 Session

**API路径**: `POST/GET /v1/sessions?beta=true`, `GET/POST/DELETE /v1/sessions/{id}?beta=true`

```yaml
CreateSessionParams:
  agent: str | AgentParams        # Agent引用
  environment_id: str             # 环境ID (必填)
  title: str (≤500, nullable)
  metadata: map[str]str
  resources:                      # 挂载资源
    - file: {file_id, mount_path}
    - github_repository: {url, authorization_token, mount_path, checkout}
    - memory_store: {memory_store_id, access, instructions}
  vault_ids: [str]                # Vault引用

Session:
  id: "sesn_011CZkZAtmR3yMPDzynEDxu7"
  status: "rescheduling" | "running" | "idle" | "terminated"
  agent: SessionAgent (快照)
  environment_id: str
  resources: [SessionResource]
  vault_ids: [str]
  usage:                          # 累计Token
    input_tokens: int
    output_tokens: int
    cache_read_input_tokens: int
    cache_creation: {ephemeral_5m/1h}
  stats:                          # 计时
    duration_seconds: float
    active_seconds: float
  archived_at: timestamp|null
```

**Session States & Events** (核心事件模型):

| 状态 | 含义 | 事件流 |
|------|------|--------|
| running | Agent 执行中 | agent.message → agent.tool_use → agent.tool_result → agent.thinking |
| idle | 等待用户输入 | session.status_idle + stop_reason |
| rescheduling | 自动重试恢复 | session.status_rescheduled |
| terminated | 终止 | session.status_terminated |

> **Event 流**: `GET /v1/sessions/{id}/events/stream?beta=true` → SSE (Server-Sent Events)

**Key Events**:
```
user.message          → 用户消息
user.interrupt        → 中断
user.tool_confirmation → 工具审批(allow/deny)
user.custom_tool_result → 自定义工具结果

agent.message         → Agent文本响应
agent.thinking        → 思考进度信号
agent.tool_use        → 内置工具调用(含evaluated_permission)
agent.tool_result     → 工具执行结果
agent.mcp_tool_use    → MCP工具调用(含permission)
agent.mcp_tool_result → MCP工具结果
agent.custom_tool_use → 自定义工具调用(需客户端提供结果)
agent.thread_context_compacted → 上下文压缩事件
session.error         → 错误(含retry_status)
session.status_idle   → stop_reason: end_turn | requires_action | retries_exhausted
session.deleted       → 会话删除通知
span.model_request_start/end → Span追踪(含model_usage)
```

### 1.4 Resource

**API路径**: `GET/POST/DELETE /v1/sessions/{id}/resources?beta=true`

```yaml
# 三类资源
Resource:
  file:
    id: str
    file_id: str
    mount_path: str
  github_repository:
    id: str
    url: str
    mount_path: str
    checkout: {branch | commit}
    authorization_token: str (可rotate)
  memory_store:
    id: str
    memory_store_id: str
    access: "read_write" | "read_only"
    mount_path: str (derived)
    instructions: str (≤4096, per-attachment guidance)
```

### 1.5 Vault + Credential

**API路径**: `POST/GET /v1/vaults?beta=true`, `POST/GET/DELETE /v1/vaults/{id}/credentials?beta=true`

```yaml
Vault:
  id: "vlt_..."
  display_name: str
  metadata: map[str]str

Credential:
  id: "vcrd_..."
  vault_id: str
  display_name: str
  auth:
    static_bearer:               # 静态Bearer Token
      mcp_server_url: str
      token: str (≤8192, 写入时提供, 读取时隐藏)
    mcp_oauth:                   # OAuth 2.0 (支持refresh)
      mcp_server_url: str
      access_token: str
      expires_at: timestamp
      refresh:
        refresh_token: str
        token_endpoint: str
        client_id: str
        token_endpoint_auth:
          none | client_secret_basic | client_secret_post
```

**关键设计**: Credential 绑定到 MCP server URL 而非 Agent，实现凭证复用。

### 1.6 Skill

**API路径**: `POST/GET /v1/skills?beta=true`, `DELETE /v1/skills/{id}?beta=true`

```yaml
Skill:
  id: "skill_..."
  display_title: str
  source: "custom" | "anthropic"
  latest_version: str (Unix epoch timestamp)
  created_at: timestamp
  updated_at: timestamp

SkillVersion:
  id: "skillver_..."
  skill_id: str
  version: str (Unix epoch timestamp)
  name: str
  description: str
  directory: str
  created_at: timestamp
```

**上传方式**: multipart/form-data，其中 `files` 数组包含 SKILL.md + 所有资源文件。版本号是时间戳。

### 1.7 File

**API路径**: `POST/GET /v1/files?beta=true`, `GET/DELETE /v1/files/{id}?beta=true`, `GET /v1/files/{id}/content?beta=true`

```yaml
FileMetadata:
  id: "file_..."
  filename: str (≤500)
  mime_type: str
  size_bytes: int
  downloadable: bool
  scope:                         # Beta-only: Session作用域
    type: "session"
    id: str (session_id)
  created_at: timestamp
```

### 1.8 Memory Store

**API路径**: `POST/GET /v1/memory_stores?beta=true`, `POST/GET/DELETE /v1/memory_stores/{id}/memories?beta=true`

```yaml
MemoryStore:
  id: "memstore_..."
  name: str (1-255)
  description: str (≤1024)
  metadata: map[str]str
  archived_at: timestamp|null

Memory:
  id: str
  memory_store_id: str
  path: str (2-1024字符)
  content: str | null
  content_size_bytes: int
  content_sha256: str
  memory_version_id: str (最新版本ID)
  created_at/updated_at: timestamp

MemoryVersion:
  id: str
  memory_store_id: str
  memory_id: str
  operation: "created" | "modified" | "deleted"
  created_by: Actor (session_actor | api_actor | user_actor)
  redacted_at/by: timestamp|null
  content_sha256: str (用于乐观锁precondition)
```

**关键特性**:
- 文件系统式路径 (树状结构), 支持 `path_prefix` 查询
- 乐观并发: `DELETE` 可带 `expected_content_sha256` precondition
- 完整审计链: MemoryVersion 记录每次变更的 Actor + SHA256
- 支持 `redact` 操作 (GDPR合规)
- `memory_store` 可挂载到 session 资源中

### 1.9 User Profile

**API路径**: `POST/GET /v1/user_profiles?beta=true`, `POST /v1/user_profiles/{id}/enrollment_url?beta=true`

```yaml
UserProfile:
  id: "uprof_..."
  external_id: str (≤255, nullable)
  trust_grants:                   # Trust Grant状态
    cyber:
      status: "active" | "pending" | "rejected"
  metadata: map[str]str
  created_at/updated_at: timestamp

EnrollmentUrl:
  url: str
  expires_at: timestamp
```

**用途**: 平台用户注册 + Trust Grant (如 Cyber安全合规审查) 流程。

---

## 2. CMA API 路由拓扑

```
/v1/
├─ messages                          # 标准 Messages API
│   ├─ POST /messages                # Create Message
│   ├─ POST /complete                # [Legacy] Text Completion
│   ├─ POST /messages/count_tokens   # Token计数
│   └─ /batches                      # 批量(创建/列表/取消/结果)
├─ models                            # 模型列表
│   ├─ GET /models
│   └─ GET /models/{model_id}
├─ files                             # 文件管理
│   ├─ POST/GET /files
│   └─ GET/DELETE /files/{id}
│       └─ GET content
├─ skills                            # Skill管理
│   ├─ POST/GET /skills
│   ├─ GET/DELETE /skills/{id}
│   └─ POST/GET/DELETE /skills/{id}/versions/{version}
│
└─ ?beta=true (Beta CMA前缀)
    ├─ /environments                 # Environment CRUD
    │   ├─ POST/GET
    │   ├─ GET/POST/DELETE /{id}
    │   └─ POST /{id}/archive
    ├─ /agents                       # Agent CRUD
    │   ├─ POST/GET
    │   ├─ GET/POST /{id}
    │   ├─ POST /{id}/archive
    │   └─ GET /{id}/versions
    ├─ /sessions                     # Session + Event Stream
    │   ├─ POST/GET
    │   ├─ GET/POST/DELETE /{id}
    │   ├─ POST /{id}/archive
    │   ├─ GET /{id}/events          # 事件列表
    │   ├─ POST /{id}/events         # 发送事件
    │   ├─ GET /{id}/events/stream   # SSE事件流
    │   └─ GET/POST/DELETE /{id}/resources
    │       └─ GET/POST/DELETE /{id}/resources/{resource_id}
    ├─ /vaults                       # Vault + Credential
    │   ├─ POST/GET
    │   ├─ GET/POST/DELETE /{id}
    │   ├─ POST /{id}/archive
    │   └─ GET/POST/DELETE /{id}/credentials/{credential_id}
    │       └─ POST /{id}/credentials/{credential_id}/archive
    ├─ /memory_stores                # Memory Store
    │   ├─ POST/GET
    │   ├─ GET/POST/DELETE /{id}
    │   ├─ POST /{id}/archive
    │   ├─ GET/POST/DELETE /{id}/memories/{memory_id}
    │   ├─ GET /{id}/memory_versions
    │   └─ GET /{id}/memory_versions/{id}  + POST redact
    └─ /user_profiles                # User Profile
        ├─ POST/GET
        ├─ GET/POST /{id}
        └─ POST /{id}/enrollment_url
```

---

## 3. CMA 与开源产品的概念映射 (补全)

| CMA 概念 | API 路径 | 作用 | Hermes Agent 对齐 | OpenClaw 对齐 |
|----------|---------|------|-------------------|---------------|
| **Agent** | `/agents` | 定义+版本+权限 | ⚠️ 单config,无多Agent | ✅ agents.list[] |
| **Environment** | `/environments` | 沙箱规格(网络+包) | ✅ 6种Backend+容器配置 | ✅ mode/scope/backend |
| **Session** | `/sessions` | 运行时实例+事件流 | ✅ session_search+state.db | ✅ sessions_* 系列 |
| **Event** | `/events/stream` | SSE实时事件推送 | 🟡 Gateway WS | ✅ transcript-events |
| **Resource** | `/resources` | 文件/GitHub/MemStore挂载 | 🔴 无统一模型 | 🟡 Artifacts/skills |
| **Vault** | `/vaults` | 密钥存储+OAuth | 🟡 .env+auth.json | ✅ mcp credentials |
| **Credential** | `/credentials` | MCP凭证绑定 | ✅ mcp_oauth.py | ✅ per-server配置 |
| **Skill** | `/skills` | 上传+版本+分发 | ✅ Curator+Hub | ✅ Skill Workshop |
| **File** | `/files` | 文件上传+Session作用域 | 🟡 data/目录 | ✅ Files API |
| **Memory Store** | `/memory_stores` | 持久化KV+版本+审计 | ✅ Honcho/mem0+SOUL.md | ✅ AGENTS.md+sqlite-vec |
| **User Profile** | `/user_profiles` | 平台用户+Trust Grant | 🔴 无显式模型 | 🔴 无显式模型 |

---

## 4. CMA 架构模式总结

```
┌──────────────────────────────────────────────────────┐
│  管理面 (Management Plane)                           │
│  /agents  /environments  /skills  /files             │
│  /vaults  /memory_stores  /user_profiles            │
│  → 资源定义、CRUD、版本管理、归档策略                  │
├──────────────────────────────────────────────────────┤
│  控制面 (Control Plane)                              │
│  /sessions  (创建/状态/事件)                          │
│  /events/stream  (SSE实时推送)                       │
│  → Session生命周期、权限审批(HITL)、状态机编排         │
├──────────────────────────────────────────────────────┤
│  数据面 (Data Plane)                                 │
│  /messages  (标准Messages API)                       │
│  /batches   (批量推理)                               │
│  → LLM推理、工具执行、结果返回                        │
└──────────────────────────────────────────────────────┘
```

**管理面**：资源定义和生命周期管理（CRUD + 版本 + 归档）
**控制面**：将管理面的定义实例化为运行时 Session，编排 Agent Loop，处理 HITL 审批和事件推送
**数据面**：执行 LLM 推理、工具调用（内置/MCP/自定义）和结果收集

---

## 5. 开源实现对照 (从横向对比报告提取)

基于 v3 横向对比报告数据，四个候选产品的 CMA 对齐度：

| CMA 概念 | Hermes Agent | OpenClaw | Deep Agents | DeerFlow 2.0 |
|----------|:-----------:|:--------:|:-----------:|:------------:|
| Agent 多实例 | 🔴 无 | ✅ 原生 | ⚠️ SubGraph | ⚠️ custom_agents |
| Environment | ✅ 6后端 | ✅ 4级 | ✅ 5后端 | ✅ 4层 |
| Session | ✅ FTS5 | ✅ sessions_* | ✅ checkpointer | ✅ SQLite/PG |
| Event Stream | 🟡 WS | ✅ transcript | ⚠️ checkpoint | ✅ run_events |
| Resource 模型 | 🔴 无 | 🟡 artifacts | 🔴 无 | 🔴 无 |
| Vault/Credential | 🟡 env+auth | ✅ mcp creds | 🟡 env | ✅ McpOAuth |
| Skill 管理 | ✅ Curator | ✅ Workshop | ✅ Discovery | ✅ SKILL.md |
| File API | 🟡 data/ | ✅ Files | ❌ | ❌ |
| Memory Store | ✅ 双后端 | ✅ sqlite-vec | ❌ | ⚠️ LLM驱动 |
