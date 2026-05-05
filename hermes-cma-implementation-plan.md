# Hermes Agent → CMA 落地方案（修订版）

> 基于垂直能力解构报告, 对标 Anthropic CMA 九大核心概念
> 分析日期: 2026-05-04 | 修订: 2026-05-05 (集成源码工程核查)

---

## 0. 前置说明

Hermes Agent 在已知分析产品中 CMA 对齐度最高。本次源码工程核查发现需修正以下关键认知:

| 原认定 | 修正 | 影响 |
|--------|------|------|
| Agent 多配置 (模板化 ✅) | 🔴 单 config.yaml，无多 Agent 定义 | CMA Agent 多实例需新建 |
| SubAgent 子资源隔离 | ⚠️ 仅 toolset 级(平台维度)，无 per-Agent 隔离 | 不同 Agent 无法独立配置 tools/skills/sandbox |
| 沙箱隔离 | ⚠️ task_id="default" 全局共享 | 所有用户共用同一容器，多租户隐患 |
| Skill 热加载 ✅ | ⚠️ 内容实时读但命令列表缓存，无文件监控 | 新增 Skill 需 Gateway `/reload-skills` 或重扫 |

**保留的优势**: Skills 自动创建/改进闭环、FTS5 Session Search、双向 MCP、6 种 Terminal Backend。

---

## 1. CMA 九大核心概念对齐

### 1.1 Agent

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 模板定义 | ⚠️ **单 Agent** | `config.yaml` (430行) 定义单个 Agent。**不支持多 Agent 配置** — 无 `agents.list[]` 或 `agents/<name>/` 机制 |
| 多实例 | 🔴 **不存在** | 仅通过 `platform_toolsets` 按平台区分 toolset (`hermes_cli/tools_config.py:52-94`)，非独立 Agent 身份 |
| CRUD | 🟡 需补齐 | `hermes config set/get` 提供配置管理, 但无 Agent Registry API |
| 实例化 | ✅ 已对齐 | `run_agent.py AIAgent()` 从 config 创建 Agent 实例 |
| 子资源隔离 | ⚠️ **toolset 级** | `TOOLSETS` dict + `enabled_toolsets`/`disabled_toolsets` (`toolsets.py:73-501`; `model_tools.py:335-484`)。Skills 全局共享，Sandbox `task_id="default"` 全局共享 |
| 版本管理 | 🟡 需补齐 | Git checkpoint 可做版本回溯, 但非结构化的 Agent 版本 |
| 权限绑定 | ✅ 已对齐 | `command_allowlist` + `approvals` + `FilesystemPermission` |

**方案**: CMA Agent 多实例需**新建** Agent Registry + per-Agent config 隔离机制。当前单 config 模型不足以支撑 CMA 的多 Agent 管理。

### 1.2 Environment (含 Sandbox)

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 环境定义 | ✅ 已对齐 | `terminal.backend` (6种: local/docker/ssh/daytona/modal/singularity) + `container_*` 配置 |
| 沙箱生命周期 | ✅ 已对齐 | Docker: UUID 容器名 + `sleep infinity` + 安全加固(`--cap-drop=ALL`)；SSH: ControlMaster 多路复用；Daytona: stop/delete 双模式 |
| 外接沙箱 | ✅ 已对齐 | `TERMINAL_ENV` 环境变量选后端。Docker: docker_image + docker_volumes；SSH: host/user/port/key；Daytona: DAYTONA_API_KEY |
| 沙箱传递 | 🟡 **隐式** | 不显式传入 `run_conversation()`，`terminal_tool` 执行时从环境变量读配置 (`tools/terminal_tool.py:1101-1208`) |
| 沙箱共享问题 | 🔴 **全局共享** | `task_id="default"` → 所有用户共用同一 Docker/SSH/Daytona 实例 (`tools/terminal_tool.py:964`) |
| 资源配额 | 🟡 需补齐 | `container_cpu: 1` / `container_memory: 5120` / `container_disk: 51200` 已定义但无 CMA 式绑定 |
| 网络策略 | 🟡 需补齐 | `website_blocklist` + `network.force_ipv4` 提供基本控制 |

**方案**: 沙箱能力深度足够，但 **`task_id="default"` 共享问题是企业 CMA 的关键短板**。需实现 per-Agent 沙箱分配。

### 1.3 Session

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 创建/列表/状态/删除 | ✅ 已对齐 | `session_search` + FTS5 索引 + SQLite state.db |
| Session 隔离 | ✅ 已对齐 | `build_session_key()`: platform+chat_id+user_id 三元组 |
| Sandbox 绑定 | ⚠️ 需补齐 | task_id="default" 共享，非 per-session 隔离 |
| 状态查询 | ✅ 已对齐 | `/usage` + `/insights` + `hermes_state.py` |

### 1.4 Event

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 事件流 | ✅ 已对齐 | Gateway WS events + 结构化日志 |
| Trace 聚合 | 🟡 需补齐 | `trajectory_compressor` 提供轨迹记录, 但非云聚合 |

### 1.5 Resource

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 统一资源模型 | 🔴 不存在 | 无 CMA 式统一 Resource URI |
| Artifacts | 🟡 需补齐 | `data/` 目录管理文件, 但无 API |

### 1.6 Vault

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 密钥存储 | 🟡 需补齐 | `.env` + `auth.json` 已存在 |
| 加密 | 🟡 需补齐 | `security.redact_secrets` 提供脱敏 |
| 绑定 | 🟡 需补齐 | API key 通过环境变量注入 |
| 轮换 | 🔴 不存在 | — |

### 1.7 File

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 上传/下载 | ✅ 已对齐 | `read_file`/`write_file`/`patch`/`search_files` |
| 列表/搜索 | ✅ 已对齐 | `search_files(target='files')` |
| 转换 | 🔴 不存在 | — |
| 上传限制 | 🟡 需补齐 | `file_read_max_chars: 100000` + `tool_output.max_bytes: 50000` |

### 1.8 Skill

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 定义格式 | ✅ 已对齐 | SKILL.md (YAML + Markdown), 兼容 agentskills.io |
| CRUD | ✅ 已对齐 | `skill_manage` (action=create/patch/edit/delete) |
| 绑定 | ⚠️ **全局** | `skills.external_dirs` + 内置 skills。**无 per-Agent 过滤** |
| 热加载 | ⚠️ **被动** | Skill 内容每次 `/skill-name` 调用实时读文件 (`agent/skill_commands.py:53-96`)。命令列表模块级缓存，平台变化触发重载。Gateway `/reload-skills` 主动触发。**无文件监控** |
| 版本 | ✅ 已对齐 | Curator: backup keep=5, archive |
| 渐进式披露 | ✅ 已对齐 | 先加载 name+description, 使用时加载完整 SKILL.md |
| **自动创建** | ✅ 已对齐 (独有) | Agent 从经验中自动创建 Skills |
| **自动改进** | ✅ 已对齐 (独有) | Skills 使用中持续自我改进 |

### 1.9 MCP

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| Server 管理 | ✅ 已对齐 | `mcp_serve.py` (MCP Server) + `native-mcp` skill (MCP Client) |
| 传输协议 | ✅ 已对齐 | stdio/HTTP |
| OAuth | 🟡 需补齐 | `auxiliary.mcp.provider` 配置存在, OAuth 状态不明 |
| 工具过滤 | 🟡 需补齐 | Toolset 系统可过滤 |

---

## 2. 配置更新机制

| 机制 | 详情 | 出处 |
|------|------|------|
| 检测 | mtime + size 三元组缓存 key，每次 `load_config()` 调用时检查 | `hermes_cli/config.py:34-44` |
| Tool 缓存 | `cfg_fp = (st_mtime_ns, st_size)` 纳入 schema key，config 变更后自动失效 | `model_tools.py:297-320` |
| 生效时机 | 下次 Agent API 调用 | — |
| 主动重载 | Gateway `/reload-skills` 命令 | — |
| 限制 | **无文件监控** — 纯被动检测，非 OpenClaw 式 chokidar watcher | — |

---

## 3. 子资源关联关系分析

```
CMA Agent
  ├─ ⚠️ 模板: config.yaml 单 Agent — 需多实例机制
  ├─ ⚠️ Tools: Toolset 级 — 需 per-Agent 隔离
  ├─ ⚠️ Skills: 全局共享 — 需 per-Agent 过滤
  ├─ ✅ Memory: memory tool + Honcho user_id 隔离
  ├─ ✅ Permissions: command_allowlist + approvals
  ├─ ✅ Environment: terminal.backend 6 种
  ├─ ✅ MCP: 双向 MCP
  └─ 🔴 Agent Registry: 不存在

CMA Environment
  ├─ ✅ Backend: 6 种 + container_* 配置
  ├─ ✅ 外接: TERMINAL_ENV 环境变量
  ├─ ✅ 生命周期: UUID 容器 + daemon 空闲清理
  ├─ 🔴 共享问题: task_id="default" 全局共享
  └─ 🟡 资源配额: 已定义但未 CMA 绑定

CMA Skill
  ├─ ✅ SKILL.md + skill_manage CRUD
  ├─ ✅ Curator 自动老化/归档
  ├─ ⚠️ 热加载: 被动重扫, 无文件监控
  └─ ⚠️ 绑定: 全局, 无 per-Agent 过滤
```

### 统计 (修订)

| 状态 | 数量 | 说明 |
|------|------|------|
| ✅ 已对齐 | 23 | Skills CRUD/Curator, MCP 双向, Session FTS5, File, 6 Backend |
| 🟡 需补齐 | 9 | Agent Registry, per-Agent 隔离, Vault 轮换, Event API |
| 🔴 不存在 | 4 | Agent 多实例, Skill per-Agent, Resource URI, Sandbox 隔离 |

---

## 4. 架构模式判定

**选择模式 B: 有管控面 (Gateway) + 数据面 (AIAgent)**

```
┌──────────────────────────────────────────────────────────────┐
│              Hermes Control Plane (Gateway — 已有，需扩展)    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Agent    │  │ Session  │  │ Event    │  │ Skill        │ │
│  │ Registry │  │ Manager  │  │ Stream   │  │ Manager      │ │
│  │ (新增)   │  │ (已有)   │  │ (已有)   │  │ (已有,需扩展) │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
│                                                               │
│  WebSocket + HTTP Gateway                                      │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│              Hermes Data Plane (AIAgent — 需扩展)             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  AIAgent (run_agent.py) — 需支持多实例                │    │
│  │  + Terminal Backend (6 种) — 需 per-Agent 分配       │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. 工作项与工作量评估 (修订)

```
┌─────────────────────────────────────────────────────────────────┐
│  类别                    工作量 (人天)  改动 AIAgent?           │
├─────────────────────────────────────────────────────────────────┤
│  管控面 - Agent Registry API         5        ❌ 零改动         │
│  管控面 - Event API 封装             2        ❌ 零改动         │
│  管控面 - Resource URI 方案          2        ❌ 零改动         │
│  管控面 - Vault 轮换机制             2        ❌ 零改动         │
│  数据面 - Agent 多实例机制           8        ⚠️ 核心扩展       │
│  数据面 - per-Agent 子资源隔离       5        ⚠️ 核心扩展       │
│  数据面 - per-Agent 沙箱分配         4        ⚠️ task_id 分配   │
│  数据面 - Skill 文件监控/热加载       3        ⚠️ 新增 watcher   │
│  数据面 - 结构化 Agent Version       3        ⚠️ 新增 features  │
│  K8s 基础设施 - CRD + ConfigMap      3        ❌ 零改动         │
│  K8s 基础设施 - Secret/ESO           2        ❌ 零改动         │
│  测试与文档                          6        ❌               │
├─────────────────────────────────────────────────────────────────┤
│  总计                               45                           │
│  其中: 零 AIAgent 改动              18 人天 (40%)               │
│        需 AIAgent 核心扩展           27 人天 (60%)               │
└─────────────────────────────────────────────────────────────────┘
```

> **修订说明**: 原方案仅 22 人天 (82% 零改动)，基于"Agent 模板化已对齐"的错误认知。源码核查确认 Hermes **不支持多 Agent**，Agent 多实例 + per-Agent 隔离 + Sandbox per-Agent 分配 + Skill 文件监控是最核心的补缺项，工作量翻倍。

---

## 6. 实施路线图 (修订)

### Phase 1: Agent 多实例基础 (Week 1-3) [P0]

- [ ] Agent 多实例机制: config.yaml 从单 Agent 扩展为 `agents: [{...}]` 数组
- [ ] Agent Registry API (`hermes agent list/create/delete`)
- [ ] per-Agent toolset/skills/sandbox 隔离
- [ ] per-Agent Sandbox 分配 (替换 task_id="default" 共享)
- [ ] Resource URI 标准化

### Phase 2: 管控面扩展 (Week 4-5) [P1]

- [ ] 结构化 Agent Version schema
- [ ] Event API (轨迹查询/流式事件)
- [ ] Skill 文件监控 (chokidar/watchdog 替代被动重扫)
- [ ] File API 暴露

### Phase 3: 安全加固 (Week 6) [P2]

- [ ] Vault 密钥轮换
- [ ] OAuth-based MCP
- [ ] K8s CRD + Secret 集成
