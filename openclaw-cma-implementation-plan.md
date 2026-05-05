# OpenClaw → CMA 落地方案（修订版）

> 基于垂直能力解构报告, 对标 Anthropic CMA 九大核心概念
> 分析日期: 2026-05-04 | 修订: 2026-05-05 (集成源码工程核查)

---

## 0. 前置说明

OpenClaw 的核心定位是 **Personal AI Assistant Platform**，而非 Agent SDK。其 CMA 对齐分析需考虑以下差异:

| 因素 | 现状 | 影响 |
|------|------|------|
| 管控面 | Gateway daemon (WebSocket + HTTP) 已存在 | ✅ 天然具备部分 CMA 管控能力 |
| Agent 引擎 | `@mariozechner/pi-agent-core` 外部依赖 | ⚠️ 改造受限于上游 |
| 多 Agent | `agents.list[]` 数组已支持 | ✅ 可直接映射 CMA Agent 多实例 |
| 配置热重载 | chokidar hybrid 模式 (hot/restart) | ✅ 配置变更即时生效 |
| 多租户 | 无 tenant 概念 (Session 级隔离) | ⚠️ 企业级 CMA 需补齐 |

---

## 1. CMA 九大核心概念对齐

### 1.1 Agent

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 模板定义 | ✅ **已对齐** | `agents.list[]` 数组定义多 Agent (`src/config/types.agents.ts:137-140`)。每项含 `id` + 独立 `sandbox`/`tools`/`skills` 配置段 |
| 多实例 | ✅ **已对齐** | `agents.defaults` 提供全局默认值，`list[].*` 逐项覆盖。默认 Agent 选定: 首个 `default:true` 或列表首项 (`src/agents/agent-scope-config.ts:82-94`) |
| CRUD | 🟡 需补齐 | 通过 config mutation API (`mutateConfigFile`/`replaceConfigFile`) 间接操作，无独立 Agent Registry API |
| 子资源隔离 | ✅ **已对齐** | per-agent `tools.allow/deny` (`src/config/types.agents.ts:132`)、`sandbox` (`agents.list[].sandbox`)、`skills` (`agents.list[].skills` 替换式，非合并 — `src/config/types.agents.ts:99`) |
| 实例化 | ✅ 已对齐 | Multi-agent routing 按 channel/account/peer 自动创建实例 |
| 版本管理 | 🔴 不存在 | 无 Agent 版本控制 |

**方案**: 在 Gateway 协议中新增 `agent.registry` 方法族 (list/create/update/delete)，封装已有 `mutateConfigFile` 基础设施。Agent 多实例能力已原生支持，CMA 管控面只需提供 Registry CRUD 包装。

### 1.2 Environment (含 Sandbox)

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 环境定义 | ✅ **已对齐** | Sandbox mode(off/non-main/all) + scope(agent/session/shared) + backend(docker/ssh/openshell) 四级配置 (`docs/gateway/sandboxing.md`) |
| 沙箱生命周期 | ✅ **已对齐** | `openclaw sandbox recreate/list/explain` CLI 管理；Docker: `CMD ["sleep","infinity"]` 长运行容器 + `setupCommand` 一次性初始化 |
| 外接沙箱 | ✅ **已对齐** | SSH backend: `agents.list[].sandbox.ssh.target` 指向已有 SSH server (`src/config/types.sandbox.ts:102-125`)；Plugin: `registerSandboxBackend()` 注册第三方 (`src/agents/sandbox.ts:17-19`) |
| 沙箱传递 | ✅ **已对齐** | `resolveSandboxContext` → PI embedded runner → `pi-embedded-runner/sandbox-info.ts` (`src/agents/sandbox/config.ts:221-277`) |
| Sandbox 绑定 | ✅ 已对齐 | `sandbox.mode: "non-main"` 绑定到 session；per-agent `sandbox` 段覆盖 global |
| 资源配额 | 🔴 不存在 | 无 Docker `--memory`/`--cpus` 集成 |
| 网络策略 | 🟡 需补齐 | SSRF policy 存在 (`ssrf-policy`/`ssrf-runtime`)，但非 CMA 式 NetworkPolicy |

**方案**: 已有沙箱能力可直接映射 CMA Environment。补充 Docker resource limit → `--memory`/`--cpus` 映射。SSRF policy 提升为结构化 NetworkPolicy。

### 1.3 Session

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 创建/列表/状态/删除 | ✅ 已对齐 | `sessions_list`/`sessions_history`/`sessions_send`/`sessions_spawn` 提供完整 Session 操作 |
| DM 隔离 | ✅ 已对齐 | `dmScope`: main/per-peer/per-channel-peer/per-account-channel-peer 四级 (`docs/concepts/session.md`) |
| 生命周期 | ✅ 已对齐 | daily reset (4AM) + idle reset + manual /reset + maintenance(pruneAfter/maxEntries) |
| Sandbox 绑定 | ✅ 已对齐 | `non-main` session 自动绑定 Sandbox |
| 状态查询 | 🟡 需补齐 | Session 状态可通过 WS 查询，但缺少标准 REST API |

**方案**: Gateway 协议已覆盖 Session 操作。补充 REST API 便于外部系统集成。

### 1.4 Event

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 事件流 | ✅ 已对齐 | Gateway WS `event:agent`/`event:presence`/`event:tick` 提供实时事件流 |
| 过滤/分页 | 🟡 需补齐 | Events 不重放 ("Events are not replayed")，无服务端过滤 |
| Trace 聚合 | 🟡 需补齐 | `/trace on` 本地追踪，非云聚合 |

**方案**: Gateway 协议已覆盖 Event。可扩展 event storage 支持历史事件查询。

### 1.5 Resource

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 统一资源模型 | 🔴 不存在 | 无 CMA 式统一 Resource URI |
| Artifacts/Uploads | 🟡 需补齐 | Media store 存在 (`media-store`/`media-runtime`)，但非统一 Artifact API |
| URI 方案 | 🔴 不存在 | 无标准化 URI |

**方案**: 利用 Gateway HTTP 服务器 (`/__openclaw__/`) 新增 Resource API。

### 1.6 Vault

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 密钥存储 | 🟡 需补齐 | `secret-ref-runtime`/`secret-file-runtime` 提供密钥引用，但存储方式不透明 |
| 加密 | 🔴 不存在 | 无内置加密 |
| 绑定到 Agent | 🟡 需补齐 | 推断: 通过 channel secret runtime 绑定 |
| 轮换 | 🔴 不存在 | 无轮换机制 |

**方案**: 利用 Node.js `crypto` 模块加密存储 Secret。Vault 对于个人产品优先级 P2。

### 1.7 File

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 上传/下载 | 🟡 需补齐 | Workspace 文件系统 + Media store，但无统一 File API |
| 列表/搜索 | 🟡 需补齐 | `read`/`glob` 工具提供基本文件操作 |
| 转换 | 🔴 不存在 | 无文件格式转换 |
| 上传限制 | 🟡 需补齐 | Sandbox tool policy 提供间接限制 |

**方案**: 封装 Workspace 和 Media store 为统一 File API。

### 1.8 Skill

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 定义格式 | ✅ 已对齐 | SKILL.md (YAML frontmatter + Markdown)，遵循 Anthropic Agent Skills 规范 |
| CRUD | 🟡 需补齐 | 通过 ClawHub 分发 + 本地文件管理，无 CRUD API |
| 绑定 | ✅ **已对齐** | per-agent `skills` 段独立声明 (`agents.list[].skills`)，替换式非合并 |
| 热加载 | ✅ **已对齐** | 版本快照失效: `bumpSkillsSnapshotVersion()` → 会话下一轮检测 → 重建快照 (`src/agents/skills/refresh-state.ts:38-72`)。配置文件变更触发自动重载 (`src/gateway/config-reload.ts:236-238`) |
| 版本 | ✅ 已对齐 | ClawHub 提供版本化技能分发 |
| 渐进式披露 | ✅ 已对齐 | SKILL.md 格式天然支持渐进式 |

**方案**: Skills 系统对齐度最高。仅需补充 CRUD API 封装已有的版本快照机制。

### 1.9 MCP

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| Server 管理 | 🟡 需补齐 | `@modelcontextprotocol/sdk` 集成，推断: 配置文件管理 MCP servers |
| 传输协议 | ✅ 已对齐 | stdio/HTTP 支持 |
| OAuth | 🟡 需补齐 | 状态不确定 |
| 工具过滤 | 🟡 需补齐 | 推断: 通过 Plugin SDK 的 tool policy |

**方案**: MCP 集成已通过 SDK 实现。补充 OAuth 和工具过滤。

---

## 2. 配置更新机制

OpenClaw 使用 **chokidar 文件监听 + hybrid 模式** 的配置热重载:

| 机制 | 详情 | 出处 |
|------|------|------|
| 监听 | chokidar, `awaitWriteFinish` 200ms stability | `src/gateway/config-reload.ts:368-372` |
| 模式 | off / restart / hot / hybrid (默认) | `src/gateway/config-reload-settings.ts:9-12` |
| Agent 变更 | `agents.list` → hot reload + heartbeat restart | `src/gateway/config-reload-plan.ts:96-99` |
| Skills 变更 | `skills.*` → version snapshot invalidation | `src/gateway/config-reload.ts:39-53` |
| Gateway 变更 | `gateway.*` → restart | `src/gateway/config-reload-plan.ts:125` |

**结论**: CMA 管控面修改 Agent 配置后，无需重启 Gateway。修改 `agents.list` 热重载，修改 `skills.*` 触发快照版本递增。满足 CMA 管理面的配置下发需求。

---

## 3. 子资源关联关系分析

```
CMA Agent
  ├─ ✅ 多实例: agents.list[] 数组 — 已原生支持
  ├─ ✅ Channels: 24+ Channel plugins → 直接映射为 Agent 入口
  ├─ ✅ Sandbox: per-agent sandbox 段 + mode/scope/backend 四级
  ├─ ✅ Tools: per-agent tools.allow/deny — 已隔离
  ├─ ✅ Skills: per-agent skills 段 (替换式) + 版本快照热加载
  ├─ ✅ DM Policy: pairing/allowlist → per-agent
  ├─ 🟡 Agent Registry: 需 Registry CRUD API 封装已有 config mutation
  ├─ 🟡 Memory: AGENTS.md → 已有, 需多源管理
  └─ 🔴 Agent Versioning: 不存在

CMA Environment
  ├─ ✅ Sandbox mode/scope/backend: 四级配置
  ├─ ✅ 沙箱生命周期: sandbox recreate/list/explain CLI
  ├─ ✅ 外接沙箱: SSH backend + registerSandboxBackend()
  ├─ ✅ 沙箱传递: resolveSandboxContext → PI Agent
  ├─ 🟡 SSRF policy: 已有, 需提升为 NetworkPolicy
  └─ 🔴 Resource quota: 不存在

CMA Session
  ├─ ✅ sessions_* tools: 完整的 Session 管理
  ├─ ✅ dmScope: 四级 DM 隔离
  ├─ ✅ 生命周期: daily/idle/manual reset
  └─ ✅ Sandbox binding: non-main session → sandbox

CMA Skill
  ├─ ✅ SKILL.md 格式
  ├─ ✅ per-agent skills 段
  ├─ ✅ 版本快照热加载 (chokidar → bumpSkillsSnapshotVersion)
  └─ 🟡 CRUD API
```

### 统计

| 状态 | 数量 | 说明 |
|------|------|------|
| ✅ 已对齐 (zero work) | 19 | 多 Agent 实例、per-agent 隔离、沙箱四级配置、版本快照热加载等 |
| 🟡 需补齐 (work needed) | 9 | Agent Registry API, File API, Vault, Event history, Resource URI 等 |
| 🔴 不存在 (must build) | 4 | Agent versioning, Resource quota, Encryption, File conversion |

---

## 4. 架构模式判定

**选择模式 B: 有管控面 + 数据面**

OpenClaw 天然具备管控面 (Gateway daemon)，数据面 (Agent Engine / PI Agent)。方案: 扩展现有 Gateway 协议。

```
┌──────────────────────────────────────────────────────────────┐
│              OpenClaw Control Plane (Gateway — 已有)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Agent    │  │ Session  │  │ Event    │  │ Config       │ │
│  │ Registry │  │ Manager  │  │ Stream   │  │ Mutation     │ │
│  │ (新增)   │  │ (已有)   │  │ (已有)   │  │ (已有)       │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
│                                                               │
│  WebSocket Protocol (TypeBox → JSON Schema → Swift)           │
│  + HTTP (/__openclaw__/...)                                    │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│              OpenClaw Data Plane (PI Agent — 已有)            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  PI Agent Core (@mariozechner/pi-agent-core)          │    │
│  │  + PI AI + PI Coding Agent + PI TUI                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Sandbox Manager                                      │    │
│  │  Docker │ SSH │ OpenShell                             │    │
│  │  + registerSandboxBackend() plugin 注册               │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. 工作项与工作量评估 (修订)

```
┌─────────────────────────────────────────────────────────────────┐
│  类别                    工作量 (人天)  改动 PI Agent?          │
├─────────────────────────────────────────────────────────────────┤
│  管控面 - Agent Registry (WS methods)   4       ❌ 零改动        │
│  管控面 - File API Gateway             3       ❌ 零改动        │
│  管控面 - Event History Storage        3       ❌ 零改动        │
│  管控面 - Vault/Secret 加密            4       ❌ 零改动        │
│  管控面 - Resource URI 方案            2       ❌ 零改动        │
│  管控面 - MCP OAuth + 工具过滤         2       ❌ 零改动        │
│  数据面 - Docker Resource Limit        2       ❌ 零改动        │
│  数据面 - SSRF → NetworkPolicy 提升    2       ❌ 零改动        │
│  测试与文档                           5       ❌               │
├─────────────────────────────────────────────────────────────────┤
│  总计                                27                          │
│  其中: 零 PI Agent 改动              27 人天 (100%)              │
└─────────────────────────────────────────────────────────────────┘
```

> ⚠️ **修订说明**: 原方案中 Skill 动态绑定/热加载 (6 人天) 和 Sandbox 统一抽象 (3 人天) 已移除 — 源码核查确认 OpenClaw 已原生支持 per-agent skills 隔离 + 版本快照热加载 + registerSandboxBackend()。全部工作项变为零 PI Agent 改动。

---

## 6. 实施路线图 (修订)

### Phase 1: Agent Registry (Week 1-2) [P0]

- [ ] 设计 Agent Registry WS protocol methods (list/create/update/delete)
- [ ] Agent 状态检查通过 `agents.list[]` 数组 — 已原生支持多实例
- [ ] Agent → Channel routing 绑定管理
- [ ] Agent → Environment 绑定管理
- [ ] Config persistence via mutation API

### Phase 2: 资源与安全 (Week 3-4) [P1]

- [ ] File API (upload/download/list) via HTTP
- [ ] Resource URI 标准化
- [ ] 文件操作 size/type 限制
- [ ] Secret 加密存储
- [ ] Docker Resource Limit (`--memory`/`--cpus`)
- [ ] SSRF policy → NetworkPolicy 提升

### Phase 3: 完善 (Week 5-6) [P2]

- [ ] Event history 存储与查询
- [ ] MCP OAuth + 工具过滤
- [ ] 端到端测试
- [ ] 文档: CMA 概念映射指南
