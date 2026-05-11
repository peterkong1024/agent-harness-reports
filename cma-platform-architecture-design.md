# CMA-like Agent 管理平台架构方案

> 架构设计日期: 2026-05-11
> 数据面选型: Hermes Agent (见 `cma-dataplane-solution-selection.md`)
> 控制面基础: kubernetes-sigs/agent-sandbox
> 参考实现: stainlu/openclaw-managed-agents (406★, TypeScript, orchestrator+runtime双容器)

---

## 0. 参考实现分析: stainlu/openclaw-managed-agents

| 组件 | 实现 | 我们在此方案中的对应 |
|------|------|-------------------|
| **Orchestrator** | Docker 容器 — 暴露 CMA REST API + SSE 事件流 | 管理面(K8s Operator) + 控制面(Session Service) |
| **Runtime** | 每个 Agent 一个 OpenClaw 容器 | 每个 Agent 一个 Hermes Pod |
| **HITL 插件** | `confirm-tools-plugin` — OpenClaw plugin 实现工具审批 | 控制面 ApprovalQueue + SS→push |
| **Provider 配置** | `apply-provider-config.mjs` — 动态注入 LLM Provider | 管理面 Agent Registry → per-Agent config |
| **部署** | docker-compose.yml (单机) | K8s + Helm (企业级) |

**借鉴点**: orchestrator/runtime 双组件架构；HITL 通过 plugin 而非核心修改；Provider 配置外部化。

---

## 1. 架构目标

1. **管理面**: 资源定义与生命周期管理 — Agent/Environment/Skill/Vault/File CRUD + 版本管理
2. **控制面**: Session 编排、HITL 审批、事件的发布/持久化、Sandbox 生命周期
3. **数据面**: Hermes Agent Runtime — LLM 推理 + 工具执行 + 上下文管理

---

## 2. 4+1 架构视图

### 2.1 逻辑视图 — 三面分层

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ┌─── CMA REST API (OpenAPI 3.1) ───┐                    │
│  POST/GET /agents     /environments  /skills  /vaults                        │
│  POST/GET /sessions   /files         /memory_stores                          │
│  POST      /sessions/{id}/events      (用户消息+工具审批+自定义工具结果)       │
│  GET       /sessions/{id}/events/stream  (SSE 事件流)                         │
│─────────────────────────────────────────────────────────────────────────────│
│                                                                              │
│ ┌── 管理面 (Management Plane) — 「定义」层 ──────────────────────────────┐   │
│ │  职责: 资源的 CRUD + 版本 + 归档 + 权限。无运行时状态，无Agent Loop。      │   │
│ │                                                                          │   │
│ │  Agent Registry          Environment Manager      Skill Registry          │   │
│ │  ├─ 多Agent模板定义      ├─ 沙箱规格(网络/包)      ├─ 上传+目录管理        │   │
│ │  ├─ 版本管理(递增)       ├─ Docker/SSH/gVisor      ├─ 版本控制(unixtime)   │   │
│ │  ├─ 工具+权限模板        ├─ 归档/恢复              ├─ anthropic/custom 源   │   │
│ │  └─ 归档+查询            └─ 资源规格定义           └─ 分发+缓存失效         │   │
│ │                                                                          │   │
│ │  Vault Manager           File Manager              Memory Store Manager   │   │
│ │  ├─ 密钥安全存储         ├─ 上传/下载/分享          ├─ KV持久化存储          │   │
│ │  ├─ MCP OAuth凭证        ├─ 作用域(session)         ├─ 版本/审计/redact     │   │
│ │  └─ token自动refresh      └─ 生命周期管理           └─ read_write/read_only │   │
│ └──────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                        │
│                                      ▼ gRPC (内部服务调用)                      │
│                                                                                │
│ ┌── 控制面 (Control Plane) — 「编排」层 ───────────────────────────────────┐  │
│ │  职责: Session 生命周期 + Agent 调度 + HITL + 事件推送。无 LLM 推理。       │  │
│ │                                                                           │  │
│ │  Session Orchestrator          Event Bus                  Sandbox CTRL    │  │
│ │  ┌────────────────────┐  ┌─────────────────────┐  ┌───────────────────┐  │  │
│ │  │ Session FSM:        │  │ SSE Publisher        │  │ SandboxController  │  │  │
│ │  │                     │  │  agent.message       │  │ acquire/release    │  │  │
│ │  │  new                 │  │  agent.tool_use      │  │ 容器寿命管理       │  │  │
│ │  │  ↓ POST /sessions   │  │  agent.tool_result   │  │ NetworkPolicy      │  │  │
│ │  │  running             │  │  agent.thinking      │  │ packages注入       │  │  │
│ │  │  ├─→ requires_action│  │  agent.custom_tool_  │  │ Resource挂载       │  │  │
│ │  │  │   (待审批)       │  │  use/result          │  └───────────────────┘  │  │
│ │  │  ├─→ end_turn       │  │  session.status_*    │                          │  │
│ │  │  │   (用户输入)     │  │  span.model_request_* │  HITL Approval Queue   │  │
│ │  │  ├─→ retries_       │  │  session.error       │  ┌───────────────────┐  │  │
│ │  │  │   exhausted      │  │  session.deleted     │  │ always_ask → 入队  │  │  │
│ │  │  ├─→ terminated     │  │  thread_context_     │  │ → SSE idle(        │  │  │
│ │  │  └─→ deleted       │  │     compacted         │  │   requires_action) │  │  │
│ │  └────────────────────┘  └─────────────────────┘  │ → user.tool_conf.  │  │  │
│ │                                                      │ → PolicyEval → 执  │  │  │
│ │  Agent Dispatcher           Event Store              └───────────────────┘  │  │
│ │  ┌────────────────────┐  ┌─────────────────────┐                           │  │
│ │  │ per-Agent沙箱分配   │  │ JSONL持久化          │  Gateway (扩展Hermes) │  │
│ │  │ 负载均衡+亲和性     │  │ 事件重放+查询         │  ┌───────────────────┐  │  │
│ │  │ task_id分配         │  │ 按session/list过滤    │  │ WebSocket/SSE路由  │  │  │
│ │  └────────────────────┘  └─────────────────────┘  │ x-api-key认证      │  │  │
│ │                                                      └───────────────────┘  │  │
│ └──────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                         │
│                                      ▼ Hermes Internal (Python同进程)           │
│                                                                                │
│ ┌── 数据面 (Data Plane) — 「执行」层 ──────────────────────────────────────┐  │
│ │  职责: LLM 推理 + 工具执行 + 上下文管理。无多租户路由，无API认证。          │  │
│ │                                                                           │  │
│ │  Hermes Agent Runtime × N (per-Agent实例)                                  │  │
│ │  ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐     │  │
│ │  │ Agent-A (agent_aaa)│ │ Agent-B (agent_bbb)│ │ Agent-C (agent_ccc)│     │  │
│ │  │ task_id=agent_aaa  │ │ task_id=agent_bbb  │ │ task_id=agent_ccc  │     │  │
│ │  │                    │ │                    │ │                    │     │  │
│ │  │ Agent Loop (自有)  │ │ Agent Loop (自有)  │ │ Agent Loop (自有)  │     │  │
│ │  │ ├─ Loop检测(硬停)  │ │ ├─ Loop检测(硬停)  │ │ ├─ Loop检测(硬停)  │     │  │
│ │  │ ├─ 上下文压缩      │ │ ├─ 上下文压缩      │ │ ├─ 上下文压缩      │     │  │
│ │  │ ├─ Guardrails      │ │ ├─ Guardrails      │ │ ├─ Guardrails      │     │  │
│ │  │ └─ SubAgent委派    │ │ └─ SubAgent委派    │ │ └─ SubAgent委派    │     │  │
│ │  │                    │ │                    │ │                    │     │  │
│ │  │ Tools (per-Agent)  │ │ Tools (per-Agent)  │ │ Tools (per-Agent)  │     │  │
│ │  │ ├─ bash/read/edit  │ │ ├─ bash (only)     │ │ ├─ web_fetch       │     │  │
│ │  │ ├─ browser         │ │ ├─ read            │ │ ├─ code_execution │     │  │
│ │  │ ├─ MCP client      │ │ └─ 6 terminal后端  │ │ ├─ memory          │     │  │
│ │  │ └─ 6 terminal后端  │ │                    │ │ └─ docker sandbox  │     │  │
│ │  │                    │ │                    │ │                    │     │  │
│ │  │ Memory + Skills    │ │ Memory + Skills    │ │ Memory + Skills    │     │  │
│ │  │ ├─ FTS5会话搜索    │ │ ├─ Honcho后端      │ │ ├─ mem0向量检索    │     │  │
│ │  │ ├─ SKILL.md注入    │ │ ├─ SOUL.md         │ │ ├─ 3个技能注入     │     │  │
│ │  │ └─ SOUL.md         │ │ └─ 无技能           │ │ └─ SOUL.md         │     │  │
│ │  └────────────────────┘ └────────────────────┘ └────────────────────┘     │  │
│ │                                                                           │  │
│ │  Sandbox (per-Agent)                                                       │  │
│ │  ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐     │  │
│ │  │ Docker容器:        │ │ SSH远程:           │ │ gVisor:            │     │  │
│ │  │ python3.11+        │ │ 远程执行环境        │ │ 高性能微VM          │     │  │
│ │  │ NetworkPolicy      │ │ ResourceQuota      │ │ Firecracker备选    │     │  │
│ │  └────────────────────┘ └────────────────────┘ └────────────────────┘     │  │
│ │                                                                           │  │
│ └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**三层职责边界（铁律）**:

| 层 | 可以做 | 绝不做 |
|----|--------|--------|
| **管理面** | Agent/Environment/Skill 定义与版本管理，配置持久化 | 运行时状态跟踪、Agent Loop、工具执行 |
| **控制面** | Session 生命周期、HITL 审批、事件推送、Sandbox 创建/销毁 | LLM 推理、工具实现、Skill 解释 |
| **数据面** | LLM 推理、工具调用、上下文管理、记忆系统、SubAgent | 多租户路由、API 认证、资源配额 |

### 2.2 开发视图 — 模块拆分

**新增模块** 以 `(新建)` 标注，**复用 Hermes 已有模块** 以 `(现有)` 标注。

```
cma-platform/
├── management/                          # 管理面 — CMA REST API (K8s Operator)   [全部新建]
│   ├── api/                             
│   │   ├── agents.py                    #  POST/GET /agents + versions      (新建)
│   │   │   ├─ AgentRegistry            #  多Agent模板CRUD + 版本递增
│   │   │   └─ AgentSchema              #  OpenAPI JSON Schema 验证
│   │   ├── environments.py             #  POST/GET /environments            (新建)
│   │   │   ├─ EnvironmentManager       #  沙箱规格 + 网络 + 包管理
│   │   │   └─ SandboxSpec              #  Docker/gVisor/SSH 规格
│   │   ├── skills.py                   #  POST/GET /skills + versions       (新建)
│   │   │   └─ SkillRegistry            #  上传 + 目录 + 版本管理
│   │   ├── vaults.py                   #  POST/GET /vaults + /credentials   (新建)
│   │   │   ├─ VaultManager             #  K8s Secret 封装
│   │   │   └─ CredentialRotator        #  OAuth refresh token 自动轮换
│   │   ├── files.py                    #  POST/GET /files + content          (新建)
│   │   │   └─ FileManager              #  上传/下载/作用域(session scope)
│   │   └── memory_stores.py            #  POST/GET /memory_stores           (新建)
│   │       └─ MemoryStoreManager       #  FTS5 + KV 兼容 CMA MemoryStore API
│   ├── crd/                             # K8s CRD 定义
│   │   ├── agent.crd.yaml              #  Agent 定义 CRD
│   │   ├── environment.crd.yaml        #  Environment (sandbox spec) CRD
│   │   ├── skill.crd.yaml              #  Skill 定义 + 版本 CRD
│   │   ├── session.crd.yaml            #  Session 运行时状态 CRD
│   │   ├── vault.crd.yaml              #  Vault 引用 CRD
│   │   └── files.crd.yaml              #  File 元数据 CRD
│   ├── operator/                        # K8s Operator Controller
│   │   ├── agent_controller.py         #  Agent reconcile loop               (新建)
│   │   │   ├─ 监听 Agent CRD 变更
│   │   │   ├─ 写入 etcd (控制面读取)
│   │   │   └─ 版本管理 (version递增)
│   │   └── sandbox_controller.py       #  Sandbox reconcile                  (新建)
│   │       ├─ 创建/销毁 Sandbox Pod
│   │       ├─ NetworkPolicy 应用
│   │       └─ ResourceQuota 管理
│   └── config/                          # 配置管理
│       ├── helm/                        # Helm Charts
│       └── auth/                        # x-api-key 验证 + RBAC
│
├── control/                             # 控制面 — Session编排 + 事件总线     [全部新建]
│   ├── orchestrator/
│   │   ├── session_lifecycle.py         #  Session FSM: idle→running→terminated
│   │   │   ├─ SessionState             #  Redis HSET session:{id}
│   │   │   └─ agent_lookup             #  etcd→agent_def 读取
│   │   ├── agent_dispatcher.py          #  Agent 分发 + per-Agent 沙箱分配
│   │   │   ├─ DispatchStrategy         #  轮询/亲和性/最少连接
│   │   │   └─ task_id_allocator        #  agent_{id}_{uuid} 生成
│   │   └── hitl_approval.py             #  HITL 审批
│   │       ├─ ApprovalQueue            #  Redis Stream key=tool_use_id
│   │       └─ PolicyEvaluator          #  always_allow/always_ask 匹配
│   ├── event_bus/
│   │   ├── sse_publisher.py             #  SSE 推送 → GET /sessions/{id}/events/stream
│   │   ├── event_store.py              #  JSONL 持久化 (兼容 CMA)
│   │   └── event_types.py             #  CMA 事件类型枚举
│   └── sandbox/
│       └── k8s_sandbox.py              #  kubernetes-sigs/agent-sandbox 适配
│           ├─ SandboxCRD              #  qbec/v1beta1 映射
│           ├─ NetworkPolicy           #  limited(allowed_hosts) / unrestricted
│           └─ PackageInstaller        #  pip/npm/apt 注入
│
├── data/                                # 数据面 — Hermes Agent 扩展
│   ├── multi_agent/                     # Agent多实例                          (新建)
│   │   ├── config_loader.py            #  从环境变量加载 per-Agent 配置       (新建)
│   │   │   ├─ inject: agent_id, task_id, tools_allowlist
│   │   │   ├─ inject: skills_list, system_prompt
│   │   │   └─ inject: sandbox_backend, memory_backend
│   │   └── hermes_entry.py             #  Hermes 启动适配 (替换 config.yaml)   (新建)
│   ├── run_agent.py                     #  AIAgent — 核心 Agent Loop           (现有)
│   ├── model_tools.py                   #  工具编排 + handle_function_call      (现有)
│   ├── tools/registry.py               #  内置工具注册表                       (现有)
│   ├── agent/                           #  Agent 内部组件                      (现有)
│   │   ├── provider_adapter.py         #  LLM Provider 适配
│   │   ├── memory.py                   #  FTS5 + Honcho + mem0 后端
│   │   ├── compression.py              #  上下文压缩
│   │   └── guardrails.py               #  Tirith 威胁检测
│   └── plugins/memory/                  #  Memory 插件                         (现有)
│       ├── honcho_plugin.py
│       └── mem0_plugin.py
│
└── deployment/                          # K8s 部署
    ├── helm/
    │   ├── cma-operator/               # 管理面 Operator + CRD                (新建)
    │   ├── cma-control/                # 控制面 Session + Event + HITL       (新建)
    │   └── cma-data/                   # 数据面 Hermes x N Pods              (新建)
    └── docker/
        ├── Dockerfile.management        #  管理面 FastAPI 容器                 (新建)
        ├── Dockerfile.control           #  控制面 gRPC + SSE 容器              (新建)
        └── Dockerfile.data              #  数据面 Hermes Agent Runtime         (现有, 扩展)
```

### 2.3 进程视图 — 运行时拓扑与通信

```
                    ┌─────────────┐
                    │  CMA Client │  (Python SDK / curl / Claude Code)
                    └──────┬──────┘
                           │ HTTPS + SSE
       ┌───────────────────┼───────────────────┐
       │                   ▼                    │
       │        ┌─────────────────────┐         │  SSE connect
       │        │ Ingress Controller  │         │
       │        │ (nginx / Traefik)   │         │
       │        └─────────┬───────────┘         │
       │    ┌─────────────┼─────────────┐       │
       │    ▼             ▼             ▼       │
       │ ┌──────┐   ┌──────────┐  ┌─────────┐  │
       │ │Mgmt  │   │Control   │  │SSE Proxy│  │
       │ │API   │   │Service   │  │(WS→SSE) │  │
       │ │2x rep│   │3x rep    │  │2x rep   │  │
       │ └──┬───┘   └────┬─────┘  └────┬────┘  │
       │    │            │             │        │
       │    │ gRPC       │ Redis       │        │
       │    ▼            ▼             │        │
       │ ┌─────┐   ┌──────────┐        │        │
       │ │etcd │   │  Redis   │        │        │
       │ │Agent│   │ Session  │        │        │
       │ │Def  │   │ State +  │        │        │
       │ │     │   │ HITL Q   │        │        │
       │ └─────┘   └──────────┘        │        │
       │                               │        │
       │ gRPC (管理面 ▶ 控制面)         │        │
       │        等控制面 AgentLookup        │        │
       └───────────────────────────────┘        │
                                                │
 ┌─────────── Hermes Data Plane ──────────┐    │
 │                                         │    │
 │  ┌──────────┐ ┌──────────┐ ┌──────────┐│    │
 │  │ Hermes A │ │ Hermes B │ │ Hermes C ││◄───┘ SSE events pushed to client
 │  │ 9k LOC   │ │ 同步引擎  │ │ 同步引擎  ││    via Control Service SSE Publisher
 │  │ 同步引擎  │ │          │ │          ││
 │  │          │ │ Sandbox: │ │ Sandbox: ││
 │  │ Sandbox: │ │ Docker-B │ │ gVisor-C ││
 │  │ Docker-A │ │          │ │          ││
 │  └──────────┘ └──────────┘ └──────────┘│
 │                                         │
 │  per-Agent Pod = Hermes + Sandbox       │
 │  task_id = agent_{id}_{uuid}            │
 │  tools = per-Agent allowlist            │
 │  skills = per-Agent 过滤               │
 │  memory = per-Agent user_id             │
 └─────────────────────────────────────────┘
```

**进程间通信协议**:

| 通信路径 | 协议 | 说明 |
|---------|------|------|
| Client → Management | HTTPS/REST | CMA API，x-api-key 认证 |
| Management → etcd | gRPC | Agent/Environment/Skill 定义持久化 |
| Management → Control | gRPC | `CreateSession(agent_id, env_id)` |
| Client → Control (events) | SSE (HTTP) | `/sessions/{id}/events/stream` |
| Control → Data (Hermes) | Hermes Internal | Python 同进程函数调用，零额外延迟 |
| Control → Sandbox | k8s CRD | qbec/v1beta1 Sandbox CRD |
| Data (Hermes) → Sandbox | Docker API / SSH / k8s exec | 工具执行环境 |
| Control ↔ HITL Queue | Redis Streams | tool_use_id: {status, policy} |

**CMA Event 类型 → Hermes 内部事件映射**:

| CMA Event Type | 方向 | Hermes 触发点 | 所属 |
|---------------|------|--------------|------|
| `user.message` | Client→Agent | `POST /sessions/{id}/events` 转发 | 控制面 |
| `user.interrupt` | Client→Agent | 中断 Agent Loop (`_interrupt_requested=True`) | 控制面 |
| `user.tool_confirmation` | Client→Agent | 审批队列消费 → `run_conversation` 恢复 | 控制面 |
| `user.custom_tool_result` | Client→Agent | 自定义工具回调注入 | 控制面 |
| `agent.message` | Agent→Client | `AIAgent.run_conversation` 文本输出 | 数据面 |
| `agent.tool_use` | Agent→Client | `handle_function_call` 调用前 (需审批) | 数据面 |
| `agent.tool_result` | Agent→Client | 工具执行完毕，结果回传 | 数据面 |
| `agent.thinking` | Agent→Client | 推理过程 (extended thinking) | 数据面 |
| `agent.mcp_tool_use/result` | Agent→Client | MCP Client 调用/结果 | 数据面 |
| `agent.custom_tool_use` | Agent→Client | Custom tool 调用 | 数据面 |
| `agent.thread_context_compacted` | Agent→Client | 上下文压缩触发 | 数据面 |
| `session.status_rescheduled` | System→Client | 错误恢复重新调度 | 控制面 |
| `session.status_running` | System→Client | 状态机 running 状态 | 控制面 |
| `session.status_idle` | System→Client | 结束/挂起 (end_turn/requires_action/retries_exhausted) | 控制面 |
| `session.status_terminated` | System→Client | 致命错误终止 | 控制面 |
| `session.error` | System→Client | 错误 (model_overloaded/rate_limited/billing) | 控制面 |
| `span.model_request_start/end` | System→Client | LLM 请求跨度 (Token使用量) | 数据面 |
| `session.deleted` | System→Client | Session 彻底删除 | 控制面 |

### 2.4 部署视图 — K8s 物理拓扑

```
                   ┌─── External ───┐
                         │
                   ┌─────▼─────┐
                   │  Ingress  │  TLS termination + API routing
                   │  nginx    │
                   └─────┬─────┘
              ┌──────────┼──────────┐
         /agents/*    /sessions*    /sessions/*/events/stream
              ▼          ▼          ▼
    ┌────────────┐ ┌────────┐ ┌────────────┐
    │ Management │ │Control │ │ SSE Proxy  │
    │ Service    │ │Service │ │ (WS→SSE)   │
    │ 2 replicas │ │ 3 repl │ │ 2 replicas │
    └─────┬──────┘ └───┬────┘ └─────┬──────┘
          │            │            │
          ├────────────┼────────────┘
          ▼            ▼
    ┌──────────┐ ┌──────────┐
    │  etcd    │ │  Redis   │
    │ Agent    │ │ Session  │
    │ Config   │ │ State    │
    └──────────┘ └──────────┘

┌──────────── Hermes Data Plane ─────────────────┐
│                                                 │
│  ┌─────────────┐ ┌─────────────┐ ┌───────────┐│
│  │ Agent Pod A │ │ Agent Pod B │ │ Agent Pod ││
│  │             │ │             │ │     C     ││
│  │ Hermes RT   │ │ Hermes RT   │ │ Hermes RT ││
│  │ + Sandbox   │ │ + Sandbox   │ │ + Sandbox ││
│  │ Container   │ │ Container   │ │ Container ││
│  └─────────────┘ └─────────────┘ └───────────┘│
│                                                 │
│  Sandbox Controller (k8s-sigs/agent-sandbox)    │
│  acquir→e/start/stop/release per session        │
└─────────────────────────────────────────────────┘

┌────── Sandbox Infrastructure ──────┐
│  ┌────────┐ ┌────────┐ ┌─────────┐│
│  │ Docker │ │ gVisor │ │ Fire-   ││
│  │ Runtime│ │ Runtime│ │ cracker ││
│  └────────┘ └────────┘ └─────────┘│
│                                     │
│  NetworkPolicy: per-Pod egress      │
│  ResourceQuota: CPU/Mem/Disk        │
└─────────────────────────────────────┘
```

**K8s CRD 映射**:

| CMA 概念 | K8s CRD | 存储 |
|---------|---------|------|
| Agent | `Agent` CRD (版本链) | etcd |
| Environment | `Environment` CRD (SandboxSpec) | etcd |
| Session | `Session` CRD (运行时状态) | Redis (热) + SQLite (冷) |
| Skill | `SkillDefinition` ConfigMap | etcd |
| Vault | `VaultSecret` K8s Secret | Kubernetes Secret |
| File | `File` CRD (元数据) + PV (内容) | MinIO/S3 + etcd |
| Memory Store | `MemoryStore` CRD | SQLite FTS5 (Hermes原生) |

### 2.5 场景视图 — 关键流程

**Scenario A: 完整对话流程 (含 HITL)**

```
Client         Management API           Control Plane            Hermes Data
  │                  │                       │                      │
  │ POST /agents     │                       │                      │
  │ ────────────────>│                       │                      │
  │                  │ etcd: write Agent     │                      │
  │ <─ agent_xxx v1 ─│                       │                      │
  │                  │                       │                      │
  │ POST /sessions   │                       │                      │
  │ ────────────────>│                       │                      │
  │                  │ gRPC: CreateSession──>│                      │
  │                  │                       │ etcd: read Agent    │
  │                  │                       │ alloc task_id       │
  │                  │                       │ acquire Sandbox────>│
  │                  │                       │ dispatch Agent ────>│
  │                  │                       │                      │ start Hermes RT
  │                  │                       │                      │ load config
  │                  │                       │ <── ready ──────────│
  │ <─ sesn_xxx ─────│ <── SessionCreated ───│                      │
  │  GET .../stream  │                       │                      │
  │ ────────────────────────────────────────>│ SSE connect          │
  │                  │                       │                      │
  │ POST .../events  │                       │                      │
  │ user.message     │                       │                      │
  │ ────────────────────────────────────────>│                      │
  │                  │                       │ dispatch event ─────>│
  │ <── SSE: status=running ─────────────────│ <── running ────────│
  │ <── SSE: agent.thinking ─────────────────│ <── thinking ───────│
  │                  │                       │                      │
  │                  │                       │ agent.tool_use("bash")
  │                  │                       │ permission=always_ask│
  │                  │                       │ → ApprovalQueue      │
  │ <── SSE: status=idle ────────────────────│ stop_reason=         │
  │       (requires_action)                  │   requires_action    │
  │                  │                       │                      │
  │ POST .../events  │                       │                      │
  │ user.tool_confirmation("allow")          │                      │
  │ ────────────────────────────────────────>│                      │
  │                  │                       │ dequeue → allow ────>│
  │                  │                       │                      │ execute bash
  │ <── SSE: agent.tool_result ──────────────│ <── tool_result ────│
  │ <── SSE: agent.message ──────────────────│ <── final text ─────│
  │ <── SSE: status=idle ────────────────────│ stop_reason=end_turn │
```

**Scenario B: CMA Session 状态转换**

```
                     POST /sessions
                          │
                          ▼
                    ┌──────────┐
              ┌────>│ running  │
              │     └────┬─────┘
              │          │
    事件注入后   │  ┌───────┼───────┐
    自动恢复     │  │       │       │
              │  ▼       ▼       ▼
              │ end_turn requires retries_
              │          action   exhausted
              │  │       │       │
              │  ▼       ▼       ▼
              │ ┌──────────────┐
              └─│    idle      │
                │ (等待用户输入) │
                └──────┬───────┘
                       │
                POST /sessions/
                {id}/events
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
       user.message  user.     user.custom_
                    tool_conf  tool_result
                    │
                    ▼
              (恢复 running)

特殊情况:
  running ──(fatal error)──> terminated ──(不可恢复)──> 终态
  任一状态 ──(DELETE /sessions)──> deleted (SSE session.deleted)
```

---

## 3. 核心数据流

### 3.1 Session 创建全链路 (7步)

| Step | 组件 | 动作 | 输出 |
|:----:|------|------|------|
| 1 | Ingress | x-api-key 验证 + RBAC + 限流 | 通过/拒绝 |
| 2 | 管理面 | etcd: `GET agent:{id}` + `GET env:{id}` | AgentDef + SandboxSpec |
| 3 | 控制面 | 生成 `task_id = agent_{id}_{uuid}` | per-Agent唯一标识 |
| 4 | 控制面 | SandboxController.acquire(env) → K8s | Sandbox CRD → Pod |
| 5 | 控制面 | 注入 packages + NetworkPolicy + Resource挂载 | 就绪容器 |
| 6 | 控制面 | AgentDispatcher.create_agent_instance(config) → Hermes | 启动Hermes RT |
| 7 | 控制面 | Redis HSET session:{id} status=running | SSE publish running |

### 3.2 Agent Loop 与事件映射

```
Hermes Agent Loop (run_agent.py, 同步引擎)

  1. UserMessage 注入
       │
       ▼
  2. LLM API Call (provider_adapter → OpenAI/Anthropic格式)
       │
       ├── 非 tool_call → final text
       │   → control/event_bus/sse_publisher: SSE agent.message
       │   → control/event_bus/event_store: JSONL persist
       │   → Session FSM: stop_reason=end_turn → idle
       │
       ├── tool_call (内置工具: bash/read/edit/...)
       │   → model_tools.handle_function_call()
       │   → Guardrails check (Tirith 威胁检测)
       │   → if permission=always_ask:
       │       → control/hitl_approval: publish agent.tool_use + push queue
       │       → SSE: status=idle (requires_action)
       │       → wait for user.tool_confirmation
       │   → if permission=always_allow:
       │       → execute tool (Docker/gVisor/SSH)
       │       → SSE: agent.tool_result
       │       → continue loop
       │
       ├── mcp_tool_call
       │   → MCP Client execute (config注入的mcp_servers)
       │   → SSE: agent.mcp_tool_use + agent.mcp_tool_result
       │   → continue loop
       │
       └── custom_tool_call
           → SSE: agent.custom_tool_use
           → SSE: status=idle (requires_action)
           → wait for user.custom_tool_result

  3. 每轮迭代后:
       ├── Loop检测: max_iterations=90 or iteration_budget exhausted
       ├── Compaction: token budget 50% → agent/compression.py
       │   → SSE: agent.thread_context_compacted
       └── Span: span.model_request_start → span.model_request_end
```

---

## 4. 接口设计

### 4.1 CMA 管理面 REST API

> 完整规范见 `cma-api-resource-model.md`。以下列出核心端点。

| 端点 | 方法 | 用途 |
|------|------|------|
| `/v1/agents` | POST/GET | 创建/列出 Agent 模板 |
| `/v1/agents/{id}` | GET/POST/DELETE | 读取/更新/归档 Agent |
| `/v1/agents/{id}/versions` | GET | 列出 Agent 版本历史 |
| `/v1/environments` | POST/GET | 创建/列出 Environment (沙箱规格) |
| `/v1/sessions` | POST/GET | 创建/列出 Session |
| `/v1/sessions/{id}` | GET/POST/DELETE | 读取/更新/删除 Session |
| `/v1/sessions/{id}/events` | GET/POST | 列出/发送事件 |
| `/v1/sessions/{id}/events/stream` | GET | SSE 事件流 |
| `/v1/sessions/{id}/resources` | GET/POST | 列出/添加 Session 资源 |
| `/v1/skills` | POST/GET | 创建/列出 Skill |
| `/v1/skills/{id}/versions` | POST/GET | 创建/列出 Skill 版本 |
| `/v1/vaults` | POST/GET | 创建/列出 Vault |
| `/v1/vaults/{id}/credentials` | POST/GET | 创建/列出凭证 |
| `/v1/files` | POST/GET | 上传/列出文件 |
| `/v1/files/{id}/content` | GET | 下载文件 |
| `/v1/memory_stores` | POST/GET | 创建/列出 Memory Store |
| `/v1/memory_stores/{id}/memories` | POST/GET | 创建/列出记忆条目 |

### 4.2 控制面 → 数据面接口

```python
class AgentDispatcher:
    """
    控制面 → 数据面调度器。
    运行在控制面进程中，通过 Hermes Internal API 与数据面通信。
    """

    async def create_agent_instance(
        agent_id: str,
        agent_def: AgentDefinition,       # etcd → AgentRegistry.read()
        environment: SandboxSpec,         # etcd → EnvironmentManager.read()
        session_id: str,
        task_id: str,                     # agent_{id}_{uuid}
        vault_credentials: dict[str,str],  # VaultManager.resolve(vault_ids)
        resources: list[ResourceMount],    # files/github/memory_stores
    ) -> AgentHandle:
        """
        1. 分配 per-Agent task_id (替换全局 task_id="default")
        2. 创建 Sandbox (k8s-sigs/agent-sandbox CRD)
        3. 注入 per-Agent:
           - tools.allow = agent_def.tools 过滤
           - skills.list = agent_def.skills 过滤
           - memory backend = agent_id → user_id
           - system_prompt = agent_def.system
        4. 启动 Hermes Agent Runtime: hermes run --config env
        5. 等待 ready signal
        """

    async def send_message(h: AgentHandle, content: list[ContentBlock]) -> None: ...
    async def send_tool_confirmation(h: AgentHandle, tool_use_id: str, result: str) -> None: ...
    async def send_custom_tool_result(h: AgentHandle, id: str, content: list) -> None: ...
    async def interrupt(h: AgentHandle) -> None: ...
    async def destroy(h: AgentHandle) -> None: ...
```

---

## 5. 架构决策记录 (ADR)

| # | 决策 | 理由 | 备选 |
|---|------|------|------|
| ADR-1 | 数据面用 Hermes | 自有Loop可控 + Event全覆盖 + 极轻量71M | OpenClaw (pi外部依赖) |
| ADR-2 | 管理面自研 REST API | CMA Agent/Environment 多实例管理需新建 | 直接改Hermes config.yaml |
| ADR-3 | 控制面 Sandbox 用 k8s-sigs/agent-sandbox | K8s原生，社区标准化 CRD | 自研 SandboxCTRL |
| ADR-4 | 控制面▶ 数据面 Python 同进程 | Agent Loop 关键路径，零RPC开销 | gRPC subprocess |
| ADR-5 | Session State → Redis | 低延迟KV，Streams 支持 HITL 队列 | SQLite (无分布式) |
| ADR-6 | Agent Config → etcd | 版本管理 + watch 机制 | SQLite (无watch) |
| ADR-7 | Event Stream → SSE | CMA API 原生SSE，浏览器/curl可用 | WebSocket (双向但更重) |
| ADR-8 | per-Agent隔离通过 task_id | 利用Hermes现有机制，最小侵入 | 重写Hermes多实例层 |

---

## 6. 工作量评估

| 模块 | 工作内容 | 人天 | 风险 | 新建/复用 |
|------|---------|:---:|------|:-------:|
| 管理面 API | Agent/Environment/Skill CRUD + 版本管理 | 8 | 低 | 新建 |
| 控制面 Session编排 | 状态机 + Agent分发 + 事件持久化 | 10 | 中 | 新建 |
| 控制面 HITL审批 | 审批队列 + Policy评估 + SSE回调 | 5 | 低 | 新建 |
| 控制面 SandboxCTRL | k8s-sigs集成 + 生命周期管理 | 6 | 中 | 新建 |
| 数据面 Agent多实例 | per-Agent config/task_id隔离 | 8 | 高 | 新建 |
| 数据面 文件热加载 | watchdog Skills/Config 变更检测 | 3 | 低 | 新建 |
| K8s 部署 | CRD + Operator + Helm | 5 | 中 | 新建 |
| 测试 + 文档 | 集成测试 + CMA API对齐测试 | 8 | 低 | 新建 |
| **总计** | | **53** | | |

> 注: 选型文档中数据面工作量(45天)仅计算 Hermes 内部改造。此架构文档额外计入控制面(21天)+管理面(8天)+部署(5天)+测试(8天)，完整平台总计 53 人天。

---

## 7. 参考实现

| 参考 | 用途 |
|------|------|
| `stainlu/openclaw-managed-agents` (406★) | CMA Session 管理参考 — orchestrator+runtime 双容器架构 |
| `kubernetes-sigs/agent-sandbox` | 控制面 Sandbox 生命周期 (CRD-based) |
| `anthropic-cma.yaml` | CMA API OpenAPI 3.1 规范 (9大资源模型) |
| `hermes-agent` (132k★) | 数据面 Agent Runtime |
| `cma-api-resource-model.md` | CMA 资源模型深度解析 |
| `cma-dataplane-solution-selection.md` | 数据面选型 (Hermes 4.40 胜出) |
