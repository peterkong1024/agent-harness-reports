# 基于 DeerFlow 2.0 复刻 CMA 落地方案 v3

## 管控面(Control Plane) + 数据面(DeerFlow Data Plane) 架构

---

**数据面选型**: DeerFlow 2.0 (bytedance/deer-flow) — Agent 执行引擎  
**管控面**: 独立资源模块 (Agent/Environment/Session/Event/Resource/Vault/File/Skill/MCP)  
**对标目标**: Anthropic Claude Managed Agents (CMA)  
**部署模式**: Kubernetes  
**改动策略**: 数据面最小化改动；管控面模块通过 DeerFlow 现有 API + K8s 基础设施对接  
**分析日期**: 2026-05-03  

---

## 目录

1. [架构总览: 管控面 + 数据面](#一架构总览-管控面--数据面)
2. [管控面 9 模块 × DeerFlow 数据面对接表](#二管控面-9-模块--deerflow-数据面对接表)
3. [DeerFlow 需暴露/增强的接口清单](#三deerflow-需暴露增强的接口清单)
4. [子资源关联关系: 管控面管理 → 数据面执行](#四子资源关联关系-管控面管理--数据面执行)
5. [K8s 基础设施支撑层](#五k8s-基础设施支撑层)
6. [工作项与工作量评估](#六工作项与工作量评估)

---

## 一、架构总览: 管控面 + 数据面

### 1.1 职责边界

```
                        ┌─────────────────────────────────────┐
                        │          用户 / 外部系统              │
                        └──────────────┬──────────────────────┘
                                       │ CMA 兼容 API
                                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          管控面 (Control Plane)                           │
│                                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │  Agent   │ │Environment│ │ Session  │ │  Event   │ │   Resource   │  │
│  │  Module  │ │  Module  │ │  Module  │ │  Module  │ │    Module    │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬───────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │
│  │  Vault   │ │   File   │ │  Skill   │ │   MCP    │                  │
│  │  Module  │ │  Module  │ │  Module  │ │  Module  │                  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                  │
│       │            │            │            │                          │
│       │    管控面管理: 资源 CRUD / 关联关系 / 权限 / 审计                  │
│       │    管控面不执行: Agent 推理 / Tool 调用 / Sandbox 代码执行         │
└───────┼────────────┼────────────┼────────────┼──────────────────────────┘
        │            │            │            │
        │    管控面模块调用数据面 API (DeerFlow Gateway + LangGraph)
        ▼            ▼            ▼            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                       数据面 (Data Plane): DeerFlow                       │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Nginx (:2026) 统一入口                          │   │
│  │  /api/langgraph/*  →  LangGraph Server (:2024)                    │   │
│  │  /api/*            →  Gateway API (:8001)                         │   │
│  │  /*                →  Frontend (:3000)                             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  职责: Agent 运行时 | 沙箱执行 | SSE 流式 | 状态持久化 | 工具编排          │
│  管控面不感知: Middleware 链 | Sub-Agent 调度 | Context 压缩              │
└──────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        K8s 基础设施层                                     │
│  Namespace  │  NetworkPolicy  │  ResourceQuota  │  Secret  │  PVC       │
│  (环境隔离)  │  (网络控制)      │  (资源限制)      │ (密钥)   │  (存储)    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.2 调用链路示例

```
用户调用 CMA API:
  POST /v1/agents/{agent_id}/sessions  →  管控面 Session Module
    │
    │  1. Session Module 验证 Agent 存在 (调 Agent Module)
    │  2. Session Module 检查 Environment 可用 (调 Environment Module)
    │  3. Session Module 注入 Vault Secrets (调 Vault Module)
    │  4. Session Module 调用数据面:
    │     POST /api/langgraph/threads
    │     Header: X-Agent-Name: code-reviewer
    │     Header: X-Agent-Config: {"thinking_enabled": true, ...}
    │
    ▼
  数据面 DeerFlow:
    │  5. LangGraph Server 创建 Thread
    │  6. SandboxMiddleware 调用 Provisioner 创建 K8s Pod
    │  7. Agent 执行 (模型推理 + 工具调用)
    │  8. SSE 流式返回结果
    │
    ▼
  管控面 Session Module 接收结果 → 返回给用户
```

---

## 二、管控面 9 模块 × DeerFlow 数据面对接表

每个管控面模块需要 DeerFlow 数据面提供的接口，以及当前的对齐状态。

### 2.1 Agent Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| Agent CRUD | 注册/查询/更新/删除 Agent 模板 | `custom_agents` YAML 配置 + `agents_api` (默认关闭) | 🟡 | 启用并扩展 `agents_api` 端点 |
| Agent 实例化 | 创建 Agent 运行实例 | `POST /api/langgraph/threads` + `agent_name` 参数 | 🟢 | — |
| Agent 状态查询 | 查询 Agent 实例状态 | `GET /api/langgraph/threads/{id}/state` | 🟢 | — |
| Agent 版本管理 | 模板版本化 | 无 | 🔴 | `agents_api` 增加 version 字段 |
| Agent 权限绑定 | 模板关联 Guardrails Policy | `custom_agents[].guardrails` 配置字段 | 🟡 | CRD 字段或 API 参数 |

**DeerFlow 需提供的接口**:

| 接口 | 现有? | 说明 |
|------|:---:|------|
| `POST /api/agents` | 🆕 | 注册 Agent 模板 (需扩展 `agents_api`) |
| `GET /api/agents` | 🆕 | 列表 Agent 模板 |
| `GET /api/agents/{name}` | 🆕 | 查询 Agent 模板详情 |
| `PUT /api/agents/{name}` | 🆕 | 更新 Agent 模板 |
| `DELETE /api/agents/{name}` | 🆕 | 删除 Agent 模板 |
| `POST /api/langgraph/threads` | ✅ | 已有 — 创建 Agent 实例 (Session) |

### 2.2 Environment Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| Environment CRUD | 创建/销毁执行环境 | 无独立 API；Provisioner 隐式管理 | 🔴 | Provisioner 暴露 Environment 管理 API |
| 环境模板管理 | 环境类型 + 镜像 + 初始化脚本 | `SANDBOX_IMAGE` 环境变量 | 🟡 | Provisioner API 增加模板参数 |
| 环境状态查询 | 查询环境可用 Pod 数/资源使用 | 无 | 🔴 | Provisioner API 增加状态端点 |
| 资源配额 | CPU/Memory/Disk 限制 | K8s ResourceQuota (Provisioner 不感知) | 🟡 | Provisioner 创建 Pod 前检查 Namespace Quota |
| 沙箱生命周期 | acquire/release/shutdown | `SandboxProvider.acquire(thread_id)` + `release(sandbox_id)` + `shutdown()`；`lazy_init=True` 延迟创建；同 Thread 复用 (`sandbox/middleware.py:37-75`) | ✅ | — |
| 外接沙箱 | 已有 Docker/K8s 集群接入 | `resolve_class(config.sandbox.use, SandboxProvider)` 反射加载 Provider (`sandbox/sandbox_provider.py:44-58`)。Docker: AioSandboxProvider；K8s: + provisioner_url；Apple Container: Virtualization.framework | ✅ | — |
| 沙箱传递 | 沙箱传递给 agent loop | `SandboxMiddleware.before_agent()` → `state["sandbox"]` → 子Agent 继承 `SandboxState` (`subagents/executor.py:372-374`) | ✅ | — |

**DeerFlow 需提供的接口**:

| 接口 | 现有? | 说明 |
|------|:---:|------|
| `POST /api/environments` | 🆕 | 注册 Environment → K8s Namespace 绑定 |
| `GET /api/environments` | 🆕 | 列表 Environment |
| `GET /api/environments/{name}` | 🆕 | Environment 详情 (资源使用/状态) |
| `DELETE /api/environments/{name}` | 🆕 | 注销 Environment |
| `POST /api/sandboxes` | ✅ | 已有 — Provisioner 创建 Sandbox Pod |
| `GET /api/sandboxes` | ✅ | 已有 — Provisioner 列表 Sandbox |
| `DELETE /api/sandboxes/{id}` | ✅ | 已有 — Provisioner 销毁 Sandbox |

### 2.3 Session Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| Session 创建 | 创建对话实例 | `POST /api/langgraph/threads` | 🟢 | — |
| Session 列表 | 分页查询 Session | 无 Thread 列表端点 | 🔴 | 新增 `GET /api/threads` |
| Session 状态 | 查询 Active/Idle/Completed | ThreadState 持久化但无显式状态字段 | 🟡 | ThreadState 增加 status |
| Session 删除 | 删除对话及关联资源 | `DELETE /api/langgraph/threads/{id}` + Gateway 清理 | 🟢 | — |

**DeerFlow 需提供的接口**:

| 接口 | 现有? | 说明 |
|------|:---:|------|
| `POST /api/langgraph/threads` | ✅ | 已有 |
| `GET /api/threads` | 🆕 | 列表 Thread (分页/搜索/过滤) |
| `GET /api/threads/{id}` | 🆕 | Thread 详情 (含 status/metadata) |
| `DELETE /api/langgraph/threads/{id}` | ✅ | 已有 |

### 2.4 Event Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| Event 列表 | 按 Session 查询事件 | `GET /api/threads/{id}/runs/{rid}/events` | 🟢 | — |
| Event 过滤 | 按类型/时间过滤 | 无过滤参数 | 🟡 | 增加 `?type=&from=&to=` |
| Event 分页 | 游标分页 | `after_seq`/`before_seq` 已支持 | 🟢 | — |
| Trace 聚合 | 事件→Trace 树 | 无 | 🔴 | 需新增 Trace Aggregator (管控面独立服务) |

**DeerFlow 需提供的接口**:

| 接口 | 现有? | 说明 |
|------|:---:|------|
| `GET /api/threads/{id}/runs/{rid}/events` | ✅ | 已有 — 增加过滤参数 |
| `GET /api/runs/{rid}/messages` | ✅ | 已有 — 分页消息 |

### 2.5 Resource Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| Resource CRUD | 统一资源模型 (文件/产物) | artifacts + uploads 两套 API | 🟡 | 管控面封装统一 Resource API |
| Resource 引用 | resource:// URI | 虚拟路径 `/mnt/user-data/` | 🟡 | 管控面维护 URI → 虚拟路径映射 |
| 文件上传 | Session 级文件上传 | `POST /api/threads/{id}/uploads` | 🟢 | — |
| 产物访问 | 下载生成文件 | `GET /api/threads/{id}/artifacts/{path}` | 🟢 | — |

**DeerFlow 需提供的接口**: 无需新增。管控面 Resource Module 封装现有 API。

### 2.6 Vault Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| 密钥存储 | 加密存储 API Key | K8s Secrets (基础设施层) | 🟢 | — |
| 密钥注入 | Agent 运行时读取密钥 | `$VAR` 环境变量语法 | 🟢 | — |
| 密钥绑定 | Agent 模板关联密钥 | 无显式绑定 | 🟡 | 管控面维护绑定关系 |

**DeerFlow 需提供的接口**: 无需新增。密钥通过 K8s Secret → env var → `$VAR` 语法注入。

### 2.7 File Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| 文件上传 | Session 级上传 | `POST /api/threads/{id}/uploads` | 🟢 | — |
| 文件列表 | 查询已上传文件 | `GET /api/threads/{id}/uploads/list` | 🟢 | — |
| 文件下载 | 下载文件 | `GET /api/threads/{id}/artifacts/{path}` | 🟢 | — |
| 文件搜索 | 按名称/类型搜索 | 无 | 🔴 | 新增搜索端点 |
| 文档转换 | PDF/Office → Markdown | markitdown 自动转换 | 🟢 | — |

**DeerFlow 需提供的接口**:

| 接口 | 现有? | 说明 |
|------|:---:|------|
| `GET /api/threads/{id}/files?search=&type=` | 🆕 | 文件搜索 |

### 2.8 Skill Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| Skill CRUD | 安装/启用/禁用/删除 | `GET/PUT /api/skills` + `POST /api/skills/install` | 🟢 | — |
| Skill 绑定 | Agent 关联 Skill | `custom_agents[].skills` + SubAgent 级 `skills` 白名单 | 🟢 | — |
| Skill 传播 | 管理面更新后传递到 Agent | 后台线程异步缓存 + `refresh_skills_system_prompt_cache_async()` API + Gateway `/skills` 路由 (`lead_agent/prompt.py:137-142`; `gateway/routers/skills.py`)。超时等待 5.0s | ✅ | — |
| Skill 版本 | 版本化管理 | SKILL.md frontmatter 可选 version | 🟡 | 管控面强制校验 version |

**DeerFlow 需提供的接口**: 无需新增。现有 Skills API 已完备。

### 2.8.1 配置热重载

DeerFlow 使用 **mtime 被动检测 + 显式 reload API** 的配置热重载：

| 机制 | 详情 | 出处 |
|------|------|------|
| 自动检测 | `get_app_config()` 对比 mtime，变化则重载并记录日志 | `config/app_config.py:331-386` |
| 强制重载 | `reload_app_config()` API | `config/app_config.py:389` |
| 请求级覆盖 | `_current_app_config` ContextVar，支持单请求临时切换配置 | `config/app_config.py:333` |
| 生效时机 | 下次 API 请求时自动生效。修改 Agent 模板/Sandbox/Skill 配置无需重启 Gateway | — |

### 2.9 MCP Module

| 管控面职责 | 需数据面提供 | DeerFlow 现有接口 | 对齐度 | 需增强 |
|-----------|------------|-----------------|:---:|------|
| MCP Server CRUD | 注册/启用/禁用 | `GET/PUT /api/mcp/config` | 🟢 | — |
| MCP 工具发现 | 查询可用工具 | MCP 协议自动发现 | 🟢 | — |

**DeerFlow 需提供的接口**: 无需新增。MCP 集成已 100% 对齐。

---

## 三、DeerFlow 需暴露/增强的接口清单

### 3.1 接口汇总

```
DeerFlow 现有接口:  ████████████████████  20 个
需新增接口:         ████                   8 个
需增强接口:         ██                     2 个 (增加查询参数)
```

### 3.2 需新增接口

| # | 接口 | 所属模块 | 用途 | 优先级 |
|---|------|---------|------|:---:|
| 1 | `POST /api/agents` | Agent | 注册 Agent 模板 | P0 |
| 2 | `GET /api/agents` | Agent | 列表 Agent 模板 | P0 |
| 3 | `GET /api/agents/{name}` | Agent | 查询 Agent 模板详情 | P0 |
| 4 | `PUT /api/agents/{name}` | Agent | 更新 Agent 模板 | P0 |
| 5 | `DELETE /api/agents/{name}` | Agent | 删除 Agent 模板 | P0 |
| 6 | `GET /api/threads` | Session | 列表 Thread (分页/搜索) | P0 |
| 7 | `GET /api/threads/{id}` | Session | Thread 详情 (status/metadata) | P1 |
| 8 | `GET /api/threads/{id}/files?search=&type=` | File | 文件搜索 | P2 |

### 3.3 需增强接口

| # | 接口 | 增强内容 | 优先级 |
|---|------|---------|:---:|
| 1 | `GET /api/threads/{id}/runs/{rid}/events` | 增加 `?type=tool_call&from=&to=` 过滤参数 | P1 |
| 2 | `GET /api/threads/{id}/runs/{rid}/messages` | 增加 `?type=ai&from=&to=` 过滤参数 | P1 |

### 3.4 现有接口 (无需改动)

| 接口 | 管控面对应模块 |
|------|-------------|
| `POST /api/langgraph/threads` | Session Module (创建 Session) |
| `DELETE /api/langgraph/threads/{id}` | Session Module (删除 Session) |
| `POST /api/threads/{id}/uploads` | File Module (文件上传) |
| `GET /api/threads/{id}/uploads/list` | File Module (文件列表) |
| `DELETE /api/threads/{id}/uploads/{filename}` | File Module (文件删除) |
| `GET /api/threads/{id}/artifacts/{path}` | Resource Module (产物下载) |
| `POST /api/threads/{id}/runs/stream` | Session Module (流式运行) |
| `POST /api/threads/{id}/runs/wait` | Session Module (同步运行) |
| `GET /api/threads/{id}/runs/{rid}/messages` | Event Module (消息分页) |
| `GET /api/skills` | Skill Module (Skill 列表) |
| `GET /api/skills/{name}` | Skill Module (Skill 详情) |
| `PUT /api/skills/{name}` | Skill Module (启用/禁用) |
| `POST /api/skills/install` | Skill Module (安装 Skill) |
| `GET /api/mcp/config` | MCP Module (MCP 配置) |
| `PUT /api/mcp/config` | MCP Module (更新 MCP 配置) |
| `POST /api/sandboxes` | Environment Module (创建 Sandbox) |
| `GET /api/sandboxes` | Environment Module (Sandbox 列表) |
| `GET /api/sandboxes/{id}` | Environment Module (Sandbox 状态) |
| `DELETE /api/sandboxes/{id}` | Environment Module (销毁 Sandbox) |
| `GET /api/models` | Agent Module (可用模型列表) |

---

## 四、子资源关联关系: 管控面管理 → 数据面执行

### 4.1 关联关系管理原则

```
管控面职责 (资源模块内部或模块间调用):
  - 存储关联关系 (数据库)
  - 校验关联完整性 (Agent 引用的 Skill 必须存在)
  - 级联操作 (删除 Agent → 清理关联的 Session)

数据面职责 (DeerFlow):
  - 执行 Agent 推理
  - 管理沙箱生命周期
  - 不感知管控面的关联关系
```

### 4.2 完整父子关系与调用链路

```
Agent (管控面 Agent Module 管理)
├── [子] → Skill      管控面: Agent.skills[] 引用 Skill Module 中的 Skill ID
│                     数据面: custom_agents[].skills 白名单 → DeerFlow 按需加载
│
├── [子] → Guardrails  管控面: Agent.guardrails 引用 Guardrails Policy ID
│                      数据面: GuardrailMiddleware → Policy 评估
│
├── [子] → Environment 管控面: Agent.environment 指定目标 K8s Namespace
│                      数据面: Provisioner 在对应 Namespace 创建 Sandbox Pod
│
├── [子] → Model       管控面: Agent.model 引用 Model Module 中的模型 ID
│                      数据面: create_chat_model(name) → 实例化 LLM
│
└── [子] → Vault Secret 管控面: Agent.secrets[] 引用 Vault Module 中的密钥 ID
                        数据面: K8s Secret → env var → config.yaml $VAR 语法


Environment (管控面 Environment Module 管理)
├── [子] → Sandbox     管控面: 创建 Environment 时创建 K8s Namespace
│                      数据面: Provisioner 在 Namespace 中创建/销毁 Sandbox Pod
│
├── [子] → NetworkPolicy 管控面: Environment 创建时附带 NetworkPolicy
│                        数据面: K8s NetworkPolicy CRD (管控面通过 K8s API 创建)
│
├── [子] → ResourceQuota 管控面: Environment 创建时附带 ResourceQuota
│                        数据面: K8s ResourceQuota CRD (管控面通过 K8s API 创建)
│
└── [子] → InitScript   管控面: Environment 配置 InitContainer spec
                        数据面: Provisioner 创建 Pod 时注入 InitContainer


Session (管控面 Session Module 管理)
├── [子] → Events       管控面: 通过数据面 Events API 拉取
│                       数据面: Run Events (memory/db/jsonl)
│
├── [子] → Resources    管控面: 封装 Artifacts + Uploads 为统一 Resource API
│                       数据面: 不感知 Resource 概念
│
├── [子] → Files        管控面: 通过数据面 Uploads API 管理
│                       数据面: 不感知管控面 File 元数据
│
└── [子] → Sandbox      管控面: 不感知 (数据面 Middleware 自动绑定)
                        数据面: SandboxMiddleware → sandbox_id → ThreadState


Vault (管控面 Vault Module 管理)
└── [子] → Secret       管控面: 存储 Secret → 同步到 K8s Secret
                        数据面: envFrom.secretRef → $VAR 语法读取
```

### 4.3 管控面调用数据面关键场景

```
场景: 创建 Agent 并启动 Session

管控面 Agent Module:
  1. POST /api/agents {name, model, environment, skills[], secrets[]}
  2. 校验: Environment 存在? Skills 存在? Secrets 存在?
  3. 存储 Agent 定义 → 同步到 DeerFlow config.yaml (或通过 agents_api)

管控面 Session Module:
  4. POST /api/langgraph/threads
     Header: X-Agent-Name: code-reviewer
     Header: X-Environment: env-python
  5. 返回 thread_id → 管控面存储 Session 记录

数据面 DeerFlow:
  6. LangGraph Server 创建 Thread
  7. SandboxMiddleware → Provisioner.POST /api/sandboxes (namespace=env-python)
  8. Provisioner 在 K8s Namespace env-python 中创建 Pod
  9. Agent 推理开始
 10. SSE 流式返回结果

管控面 Event Module:
 11. 轮询 GET /api/threads/{id}/runs/{rid}/events
 12. 事件聚合为 Trace → 存储
```

---

## 五、K8s 基础设施支撑层

> 基础设施层由 K8s 原生资源提供，管控面通过 K8s API 管理，数据面从 K8s 资源中读取配置。

### 5.1 基础设施清单

| K8s 资源 | 管控面操作 | 数据面读取方式 | DeerFlow 改动 |
|----------|----------|-------------|:---:|
| **Namespace** | Environment Module 创建 | Provisioner `K8S_NAMESPACE` env var | ❌ |
| **Secret** | Vault Module → K8s API 创建/更新 | `envFrom.secretRef` → `$VAR` 语法 | ❌ |
| **NetworkPolicy** | Environment Module 创建 | K8s 自动执行 | ❌ |
| **ResourceQuota** | Environment Module 创建 | K8s 自动执行 | ❌ |
| **PVC** | 管控面基础设施配置 | Provisioner PVC 挂载 (现有机制) | ❌ |
| **ConfigMap** | Agent Module 同步配置 | Gateway config 热重载 (现有机制) | ❌ |

### 5.2 管控面 → K8s → 数据面 数据流

```
管控面 Vault Module:
  POST /internal/vault/secrets {GITHUB_TOKEN: "ghp_xxx"}
    │
    ▼
K8s API:
  kubectl create secret generic agent-secrets --from-literal=GITHUB_TOKEN=ghp_xxx
    │
    ▼
DeerFlow Pod:
  envFrom.secretRef.name: agent-secrets
    │
    ▼
config.yaml:
  api_key: $GITHUB_TOKEN    ← DeerFlow 现有语法, 无需改动
```

---

## 六、工作项与工作量评估

### 6.1 数据面改动 (DeerFlow)

| # | 改动项 | 改动方式 | 文件 | 工作量 |
|---|--------|---------|------|:---:|
| 1 | `POST/GET/PUT/DELETE /api/agents` | 扩展 `agents_api`，新增 router | `app/gateway/routers/agents.py` (新文件) | 5 人天 |
| 2 | `GET /api/threads` (列表) | 新增 router | `app/gateway/routers/threads.py` | 2 人天 |
| 3 | `GET /api/threads/{id}` (详情) | 新增 router | 同上 | 1 人天 |
| 4 | Events API 过滤参数 | 增加 query params | `app/gateway/routers/runs.py` | 1 人天 |
| 5 | `GET /api/threads/{id}/files?search=` | 新增搜索端点 | `app/gateway/routers/uploads.py` | 1 人天 |
| **数据面合计** | | | | **10 人天** |

### 6.2 管控面对接 (不实现管控面，仅定义接口契约)

| # | 工作项 | 产出 | 工作量 |
|---|--------|------|:---:|
| 1 | 管控面 9 模块接口契约定义 | OpenAPI spec 文档 | 3 人天 |
| 2 | 管控面 → 数据面调用适配层 | 调用封装 (HTTP client wrapper) | 3 人天 |
| 3 | K8s 基础设施 Helm Chart | Namespace/Secret/NetworkPolicy/ResourceQuota 模板 | 3 人天 |
| 4 | Provisioner 增强: 动态 Namespace | 从请求参数读取 Namespace | 2 人天 |
| 5 | Provisioner 增强: ResourceQuota 检查 | 创建 Pod 前检查 Namespace Quota | 1 人天 |
| 6 | Provisioner 增强: InitContainer | 读取 Environment 配置注入 | 1 人天 |
| **管控面对接合计** | | | **13 人天** |

### 6.3 测试与文档

| # | 工作项 | 工作量 |
|---|--------|:---:|
| 1 | 管控面 → 数据面集成测试 | 5 人天 |
| 2 | 管控面模块接口契约文档 | 2 人天 |
| 3 | K8s 部署文档 | 2 人天 |
| **测试文档合计** | | **9 人天** |

### 6.4 总工作量

```
┌─────────────────────────────────────────────────────┐
│  类别                    工作量       改动源码?      │
├─────────────────────────────────────────────────────┤
│  数据面 (DeerFlow 改动)   10 人天     ⚠️ 新增路由     │
│  管控面对接               13 人天     ❌ 不实现管控面  │
│  测试与文档                9 人天     ❌              │
├─────────────────────────────────────────────────────┤
│  总计                    32 人天                    │
│  管控面模块实现           (管控面团队, 不包含在本方案)  │
└─────────────────────────────────────────────────────┘
```

### 6.5 各方案对比

| 指标 | v1 (源码大改) | v2 (K8s 原生) | v3 (管控面+数据面) |
|------|:---:|:---:|:---:|
| 总工作量 | 78 人天 | 40 人天 | **32 人天** |
| DeerFlow 源码改动 | ~2000 行 | 0 行 | **~500 行** (仅路由) |
| 管控面实现 | 嵌入 DeerFlow | 独立服务 | **管控面团队负责** |
| 架构清晰度 | 低 (单体) | 中 (服务化) | **高 (面分离)** |
| 职责边界 | 模糊 | 较清晰 | **完全分离** |

---

## 附录: 管控面模块接口契约速查

| 管控面模块 | CMA 风格 API (管控面暴露) | 调用数据面接口 |
|-----------|------------------------|-------------|
| Agent Module | `POST/GET/PUT/DELETE /v1/agents` | `POST/GET /api/agents` |
| Environment Module | `POST/GET/DELETE /v1/environments` | Provisioner API + K8s API |
| Session Module | `POST /v1/agents/{id}/sessions` | `POST /api/langgraph/threads` |
| Event Module | `GET /v1/sessions/{id}/events` | `GET /api/threads/{id}/runs/{rid}/events` |
| Resource Module | `GET /v1/sessions/{id}/resources` | `GET /api/threads/{id}/artifacts` |
| Vault Module | `POST/GET/DELETE /v1/vault/secrets` | K8s Secrets API |
| File Module | `POST/GET/DELETE /v1/sessions/{id}/files` | `POST/GET /api/threads/{id}/uploads` |
| Skill Module | `POST/GET/PUT/DELETE /v1/skills` | `GET/PUT /api/skills` |
| MCP Module | `POST/GET/PUT/DELETE /v1/mcp/servers` | `GET/PUT /api/mcp/config` |

---

*本方案基于 DeerFlow 2.0 源码和文档编制。管控面模块的实现由管控面团队负责，本方案仅定义对接接口契约和 DeerFlow 数据面需暴露/增强的端点。*
