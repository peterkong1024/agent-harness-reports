# Deep Agents → CMA 落地方案（修订版）

> 基于垂直能力解构报告, 对标 Anthropic CMA 九大核心概念
> 分析日期: 2026-05-04 | 修订: 2026-05-05 (集成源码工程核查)

---

## 0. 前置条件确认

| 决策项 | 默认假设 | 影响 |
|--------|---------|------|
| 数据面选型 | deepagents SDK 作为数据面 | — |
| 管控面 | 不存在，需新建 | 🔴 全部管控面从零构建 |
| 部署模式 | K8s | ✅ 可利用 K8s 原生资源 |
| 源码改动约束 | 零改动优先 | ⚠️ 部分能力需暴露 API |
| 配置更新 | SDK 模式，无运行时热重载 | 🔴 CMA 管控面下发配置需通过新建 Agent 实例实现 |

> ⚠️ 实际落地前请向用户确认以上决策。

---

## 1. CMA 九大核心概念对齐

### 1.1 Agent

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 模板定义 | 🔴 不存在 | Agent 通过 `create_deep_agent(model, tools, system_prompt, middleware, subagents, ...)` Python API 即时构建，无声明式模板 |
| 多实例 (SubAgent) | 🟡 需补齐 | SubAgent TypedDict 支持声明式定义 (`middleware/subagents.py:25-134`)，含 name/description/tools/model/permissions。三种形态: 声明式/预编译/异步远程 (`graph.py:209`) |
| 子资源隔离 | 🟡 需补齐 | per-SubAgent `tools`(键存在则覆盖 — `graph.py:552`)、`permissions`(独立列表 — `graph.py:501`)、`skills`(独立 SkillsMiddleware — `graph.py:514-516`) |
| CRUD | 🔴 不存在 | 无 Agent 注册/发现/列表/删除 API |
| 实例化 | 🟡 需补齐 | `create_deep_agent()` 已有完整创建能力，但缺少实例管理 |
| 版本管理 | 🔴 不存在 | 无 Agent 版本概念 |

**方案**: 新建 `AgentRegistry` CRD (K8s Custom Resource)。运行时通过 CRD → `create_deep_agent()` 参数映射创建实例。SubAgent 的声明式能力已存在，管控面只需封装 Registry 层。

### 1.2 Environment (含 Sandbox)

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 环境定义 | 🔴 不存在 | 无 Environment 概念 |
| 沙箱类型 | 🟡 需补齐 | 5 种 Sandbox Provider (AgentCore/Daytona/Modal/Runloop/LangSmith)，通过 `SandboxBackendProtocol` + `BackendProtocol` 统一接口接入 |
| 外接沙箱 | ✅ **已对齐** | LangSmith Sandbox 包装 `langsmith.sandbox.Sandbox` 对象 (`backends/langsmith.py:48-57`)；Daytona/Modal/Runloop 通过 LangSmith API 间接接入 |
| 沙箱传递 | ✅ **已对齐** | `backend` 参数 → `FilesystemMiddleware` + `SummarizationMiddleware` + SubAgent 继承 (`graph.py:213,506-509,577-579`) |
| 沙箱生命周期 | ⚠️ 不透明 | Provider 决定创建/销毁策略，无统一生命周期 API |
| 资源配额 | 🔴 不存在 | 无 ResourceQuota 集成 |
| 网络策略 | 🔴 不存在 | 无 NetworkPolicy 集成 |

**方案**: 新建 `AgentEnvironment` CRD，映射到 Sandbox 后端选择 + K8s ResourceQuota/NetworkPolicy。外接沙箱通过 `SandboxBackendProtocol` 实现已有基础。

### 1.3 Session

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 创建/列表/状态/删除 | 🟡 需补齐 | LangGraph Thread 天然提供 session 概念 (`thread_id`)，通过 `checkpointer` 持久化 |
| Sandbox 绑定 | 🟡 需补齐 | Sandbox 在 Agent 创建时绑定，非 session 级 |
| 状态查询 | 🟡 需补齐 | `state_query` 可用但社区反馈有截断问题 (Issue #573) |

**方案**: 利用 LangGraph Thread API 暴露 Session CRUD；通过管控面维护 Session→Sandbox 映射。

### 1.4 Event

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 事件流 | 🟡 需补齐 | LangSmith 提供完整 trace，但需管控面封装为标准 Event API |
| 过滤/分页 | 🟡 需补齐 | 依赖 LangSmith API |
| Trace 聚合 | ✅ 已对齐 | LangSmith trace 已自动聚合，`ls_integration: "deepagents"` 标记 |

**方案**: 管控面封装 LangSmith API 为统一 Event API；或使用 LangGraph 的 streaming events。

### 1.5 Resource

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 统一资源模型 | 🔴 不存在 | 无 CMA 式的统一 Resource URI |
| Artifacts/Uploads | 🟡 需补齐 | Backend 支持文件存储，但无统一 Artifact API |
| URI 方案 | 🔴 不存在 | 无标准化 URI |

**方案**: 管控面统一 Resource URI 规范；Backend 已具备基础文件能力。

### 1.6 Vault

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 密钥存储 | 🔴 不存在 | 完全依赖环境变量 |
| 加密 | 🔴 不存在 | 无内置加密 |
| 绑定到 Agent | 🔴 不存在 | Agent 通过变量名引用密钥 |
| 轮换 | 🔴 不存在 | 无轮换机制 |

**方案**: K8s Secret + External Secrets Operator (ESO) 实现零代码改动的 Vault。Agent 通过环境变量注入读取 Secret。

### 1.7 File

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 上传/下载 | 🟡 需补齐 | Backend 支持文件读写，`FileDownloadResponse`/`FileUploadResponse` 协议已定义 (`protocol.py`) |
| 列表/搜索 | ✅ 已对齐 | `ls`/`glob`/`grep` 工具提供完整文件操作 |
| 转换 | 🔴 不存在 | 无文件格式转换 |
| 上传限制 | 🟡 需补齐 | `FilesystemPermission` 可限制路径，但无大小/类型限制 |

**方案**: Backend 已有 `FileDownloadResponse`/`FileUploadResponse` 协议，管控面封装为统一 File API。

### 1.8 Skill

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 定义格式 | ✅ 已对齐 | YAML frontmatter + Markdown (Anthropic Agent Skills 规范) |
| CRUD | 🟡 需补齐 | Skills 通过 Backend 文件系统管理，无独立 CRUD API |
| 绑定 | 🟡 需补齐 | 通过 `skills` 参数绑定到 Agent；SubAgent 独立 `SkillsMiddleware` (`graph.py:514-516`) |
| 热加载 | 🔴 **不存在** | SkillsMiddleware 首次加载后缓存 `skills_metadata` 到 agent state，已存在则跳过 (`middleware/skills.py:1016-1018`) — **无文件监听，无版本管理**。更新 Skill 需新建 session 或手动清 state |
| 版本 | 🔴 不存在 | 无 Skill 版本概念 |
| 渐进式披露 | ✅ 已对齐 | 先加载列表 (name+description)，使用时加载完整内容 |

**方案**: 原方案标注 Skill 热加载"已对齐"是错误的。Skills 使用 per-session 硬缓存，CMA 管理面更新 Skill 后无法传递到运行中的 Agent。**需新增 Skill 失效机制** — 建议通过管控面发信号清 `skills_metadata` 或移植 OpenClaw 的版本快照模式。

### 1.9 MCP

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| Server 管理 | 🟡 需补齐 | `langchain-mcp-adapters` 集成，但 Server 管理在 CLI 层 |
| 传输协议 | ✅ 已对齐 | 通过 langchain-mcp-adapters 支持 stdio/HTTP |
| OAuth | 🟡 需补齐 | CLI 代码中有 `test_mcp_auth` 测试，但功能状态不确定 |
| 工具过滤 | 🟡 需补齐 | MCP 工具注入为 Agent 工具，过滤通过 `_ToolExclusionMiddleware` |

**方案**: 管控面封装 MCP Server 注册/发现；deepagents SDK 已具备集成基础。

---

## 2. 配置更新机制

🔴 **Deep Agents 是纯 SDK，无独立运行时服务**。配置通过 `create_deep_agent()` 函数参数传入，每次调用构建新 `CompiledStateGraph`。无运行时配置监听或热交换。

| 限制 | 影响 | 对策 |
|------|------|------|
| 无热重载 | CMA 管控面无法"下发配置"到运行中的 Agent | 管控面更新 CRD → 触发新建 Agent 实例 → 旧实例逐步下线 |
| Harness profiles | 提供可复用预设 (`profiles/__init__.py`)，但仍是代码级 | 管控面 CRD 存储模板，Controller 映射到 profile |
| Skill 硬缓存 | 管理面更新 Skill 无法传递 | **需新增 Skill 失效机制** (P0) |

---

## 3. 子资源关联关系分析

```
CMA Agent
  ├─ 🟡 多实例: SubAgent TypedDict 已支持，需管控面 Registry 封装
  ├─ 🟡 SubAgent 隔离: tools/permissions/skills 独立 — 已存在
  ├─ 🟡 Skills: SkillsMiddleware + per-SubAgent — 但热加载缺失
  ├─ 🟡 Memory: MemoryMiddleware — 已有, 需管理 API
  ├─ 🟡 Permissions: FilesystemPermission — 已有, 需统一管理
  ├─ 🔴 Vault Bindings: 不存在 → 需 K8s Secret + ESO
  ├─ 🟡 Environment: Sandbox 绑定 → 需 CRD
  └─ 🟡 Model Config: model 参数 → 已有, 需模板化

CMA Environment
  ├─ 🟡 Sandbox Backend: 5 种 + SandboxBackendProtocol — 已支持
  ├─ 🟡 外接沙箱: LangSmith/AgentCore/Daytona/Modal/Runloop — 已支持
  ├─ ⚠️ 沙箱生命周期: Provider 决定，无统一 API
  ├─ 🔴 ResourceQuota: 不存在 → 使用 K8s ResourceQuota
  └─ 🔴 NetworkPolicy: 不存在 → 使用 K8s NetworkPolicy

CMA Session
  ├─ ✅ Thread: LangGraph thread_id → 天然映射
  ├─ 🟡 Sandbox Binding: 不存在 → 需管控面维护
  └─ ✅ Checkpointer: LangGraph checkpointer → 天然映射

CMA Skill
  ├─ ✅ SKILL.md 格式 + 渐进式披露
  ├─ 🟡 per-SubAgent SkillsMiddleware
  └─ 🔴 热加载: per-session 硬缓存 → 需新增失效机制
```

### 统计

| 状态 | 数量 | 说明 |
|------|------|------|
| ✅ 已对齐 (zero work) | 5 | Thread, Checkpointer, Skill Format, MCP Transport, 渐进式披露 |
| 🟡 需补齐 (work needed) | 18 | Agent Registry, Session CRUD, Event API, File API, Skills CRUD 等 |
| 🔴 不存在 (must build) | 9 | Agent 模板, Vault, Environment CRD, ResourceQuota, Skill 热加载 等 |
| 🆕 K8s 原生新增 | 4 | Secret+ESO, ResourceQuota, NetworkPolicy, CRD |

> **修订**: Skill 热加载从 ✅ 修正为 🔴，需新增工作项。

---

## 4. 架构模式判定

**选择模式 A: 无管控面 (纯数据面)**

deepagents 本身是纯 SDK/数据面，无独立管控面服务。方案如下:

```
┌──────────────────────────────────────────────────────────────┐
│                     CMA Control Plane (新建)                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Agent    │  │ Session  │  │ Vault    │  │ Trace        │ │
│  │ Registry │  │ Manager  │  │ Adapter  │  │ Aggregator   │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘ │
│       │             │             │                │          │
│       ▼             ▼             ▼                ▼          │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              K8s Native Resources                     │    │
│  │  Agent CRD  │  Secret+ESO  │  ResourceQuota          │    │
│  │  NetworkPolicy  │  ConfigMap  │  PVC                 │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│               Deep Agents Data Plane (零改动)                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  create_deep_agent(model, tools, middleware, ...)      │    │
│  │  + Backend (StateBackend / FilesystemBackend / ...)   │    │
│  │  + Sandbox (5 providers, SandboxBackendProtocol)      │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. K8s 原生资源映射

| CMA 需求 | K8s 原生资源 | 说明 |
|---------|------------|------|
| Agent 模板 | CRD `AgentTemplate` | 存储 Agent 定义 (model, system_prompt, tools, middleware, subagents) |
| 密钥管理 | Secret + External Secrets Operator | API Keys 等敏感信息通过 ESO 同步到 K8s Secret |
| 网络隔离 | NetworkPolicy | 限制 Agent Pod 的网络访问范围 |
| 资源配额 | ResourceQuota + LimitRange | 限制 Agent 的 CPU/Memory 使用 |
| 存储 | PVC (ReadWriteMany/ReadOnlyMany) | Skills/Memory 文件的持久化存储 |
| 配置管理 | ConfigMap | Agent 配置文件注入 |
| Environment | CRD `AgentEnvironment` | Sandbox 类型 + 资源配额 + 网络策略绑定 |
| Session 状态 | 利用 LangGraph Checkpointer (SQLite/PG) | 通过 PVC 持久化 |

---

## 6. 工作项与工作量评估 (修订)

```
┌─────────────────────────────────────────────────────────────────┐
│  类别                    工作量 (人天)  改动 deepagents 源码?    │
├─────────────────────────────────────────────────────────────────┤
│  管控面 - Agent Registry CRD      5       ❌ 零改动              │
│  管控面 - Environment CRD         3       ❌ 零改动              │
│  管控面 - Session Manager         4       ⚠️ 暴露 thread API      │
│  管控面 - Vault Adapter (ESO)     3       ❌ 零改动              │
│  管控面 - Trace Aggregator        3       ❌ 零改动 (LangSmith)  │
│  管控面 - Skills CRUD API         3       ❌ 零改动 (Backend)    │
│  管控面 - File API Gateway        2       ❌ 零改动 (Backend)    │
│  数据面 - Skill 热加载失效机制     5       ⚠️ 新增失效机制         │
│  数据面 - 暴露 thread CRUD        2       ⚠️ 新增路由             │
│  数据面 - 暴露 session 查询       1       ⚠️ 新增路由             │
│  K8s 基础设施 - CRD + Operator    5       ❌ 零改动              │
│  K8s 基础设施 - Secret/ESO        2       ❌ 零改动              │
│  K8s 基础设施 - NetworkPolicy     2       ❌ 零改动              │
│  K8s 基础设施 - PVC/ConfigMap     2       ❌ 零改动              │
│  测试与文档                       8       ❌                     │
├─────────────────────────────────────────────────────────────────┤
│  总计                            48                              │
│  其中: 零 deepagents 源码改动      38 人天 (79%)                  │
│        需 deepagents 改动          10 人天 (21%)                  │
└─────────────────────────────────────────────────────────────────┘
```

> **修订**: Skills 热加载从"已对齐"修正为"需新增"(5 人天)，总量 +5。

---

## 7. 实施路线图 (修订)

### Phase 1: 基础管控面 (Week 1-3) [P0]

- [ ] 定义 AgentTemplate CRD (model, tools, system_prompt, middleware, subagents)
- [ ] AgentTemplate Controller 实现 create_deep_agent() 参数映射
- [ ] K8s Secret + ESO 集成 (Provider API Keys)
- [ ] 基础 Agent 实例化流程 (CRD → Pod with deepagents)
- [ ] Sandbox 后端选择逻辑 (ConfigMap)

### Phase 2: 隔离与安全 (Week 4-5) [P0]

- [ ] Environment CRD 定义
- [ ] ResourceQuota + LimitRange 集成
- [ ] NetworkPolicy 自动生成 (基于 Environment 定义)
- [ ] FilesystemPermission 策略从管控面下发
- [ ] **Skill 热加载失效机制** (per-session 缓存清除/版本快照)  ← 新增

### Phase 3: 高级管控 (Week 6-7) [P1]

- [ ] Session Manager (CRUD on LangGraph threads)
- [ ] Event/Trace API (LangSmith 封装)
- [ ] Skills CRUD API (Backend-based)
- [ ] File API Gateway (Backend-based)
- [ ] SubAgent 注册表

### Phase 4: 完善与测试 (Week 8-9) [P2]

- [ ] 端到端集成测试
- [ ] 文档: CMA 概念映射指南
- [ ] 性能基线测试
- [ ] 安全审计
