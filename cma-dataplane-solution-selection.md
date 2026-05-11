# CMA 数据面引擎方案选型

> 选型原则: 多session → 任一隔离不满足即否决。单session → 基于 xlsx 实测数据等权打分。不改造开源产品。

---

## 关键方案选型

### 方案设计

| 序号 | 目标场景 | 方案思路概述 | 是否已预研 | 是否详细选型 |
|:---:|---------|------------|:--------:|:----------:|
| 1 | 多session | OpenClaw: tools审批+ mcp key 不满足实例隔离→否决 | 是 | 否 |
| 2 | 多session | Hermes: 5项全全局共享→否决 | 是 | 否 |
| 3 | 多session | DeerFlow: mcp仅全局配置→否决 | 是 | 否 |
| 4 | **单session** | **Hermes** | 是 | **是** |
| 5 | **单session** | **OpenClaw** | 是 | **是** |
| 6 | **单session** | **DeerFlow** | 是 | **是** |

---

## 特性技术方案选型

### CMA 数据面引擎的方案选型

#### 业务场景及技术特征

**业务场景**: 构建兼容 Anthropic CMA API 规范的 Agent 管理平台数据面。

**技术约束**:

1. **多Session 不可行** — 三个产品多session 路径全部否决: OpenClaw 不支持实例级 tools 审批和 mcp key 隔离; Hermes system_prompt / skill / mcp / tools / model 五项配置全实例共享; DeerFlow mcp 仅支持全局配置。
2. **只能走单session路径** — 每Agent=一个独立进程实例。

#### 业务规格要求

| 约束 | 要求 |
|------|------|
| 开源 | 必须开源 |
| 架构 | 单session (多session全否决) |
| 空载内存 | < 1G |
| 镜像 | < 10G |

#### 方案1: Hermes 单实例单session

全栈 Python 自有 Agent loop。

| 维度 | 实测值 |
|------|--------|
| 镜像大小 | 7.28G |
| 空载CPU | 0.05% |
| 空载内存 | 71.69M |
| 启动时间 | 6.9s |
| Event user.message | 支持 |
| Event user.interrupt | 支持 |
| Event user.tool_confirmation | 支持 |
| Event user.custom_tool_result | 支持 |
| tools审批 | 部分支持 (仅shell命令审批) |
| sandbox外接 | local / docker / ssh / singularity / modal / daytona / vercel |
| mcp凭证配置 | 支持 (~/.hermes/.env, 需重启或/reload-mcp) |

#### 方案2: OpenClaw 单实例单session

TypeScript + pi-agent-core 外部 Agent 引擎。

| 维度 | 实测值 |
|------|--------|
| 镜像大小 | 3.13G |
| 空载CPU | 8.80% |
| 空载内存 | 828.67M |
| 启动时间 | 20s |
| Event user.message | 支持 |
| Event user.interrupt | 不支持 |
| Event user.tool_confirmation | 支持 |
| Event user.custom_tool_result | 不支持 |
| tools审批 | 部分支持 (自定义plugin拦截pre_tool_call, gateway需重启) |
| sandbox外接 | 支持 (sandbox隔离部署方案) |
| mcp凭证配置 | 支持 (openclaw.json.mcp, 立即生效) |
| 安全 | 存在较大安全风险，政企单位接受度低 |

> **安全备注**: OpenClaw 在安全审计中暴露出较多风险点，面向政企客户时需重点评估。

#### 方案3: DeerFlow 单实例单session

Python + LangGraph 框架。

| 维度 | 实测值 |
|------|--------|
| 镜像大小 | — |
| 空载CPU | — |
| 空载内存 | — |
| 启动时间 | — |
| Event user.message | — |
| Event user.interrupt | — |
| Event user.tool_confirmation | — |
| Event user.custom_tool_result | — |
| tools审批 | 仅支持全局级别 |
| sandbox外接 | — |
| mcp凭证配置 | — |
| 社区活跃度 | 差距大 (64k ★ vs HM 132k / OC 368k) |

"—" 表示实测数据未采集，按 0 计。

#### 方案选型决策 — 11 维度等权打分

评分规则:
- **资源维度** (镜像/CPU/内存/启动): 取最佳为满分，按比例折算。`—` = 0。
- **Event维度** (4项): 支持 = 1，不支持 = 0，`—` = 0。
- **能力维度** (tools审批/sandbox外接/mcp凭证): 支持 = 1，不支持 = 0，部分支持 = 0.5，`—` = 0。
- **权重**: 11 维度全部相等 (1/11)。

| 维度 | Hermes | OpenClaw | DeerFlow |
|------|:------:|:--------:|:--------:|
| 镜像大小 | 0.43 | **1.00** | 0 |
| 空载CPU | **1.00** | 0.01 | 0 |
| 空载内存 | **1.00** | 0.09 | 0 |
| 启动时间 | **1.00** | 0.35 | 0 |
| Event user.message | 1 | 1 | 0 |
| Event user.interrupt | **1** | 0 | 0 |
| Event user.tool_confirmation | 1 | 1 | 0 |
| Event user.custom_tool_result | **1** | 0 | 0 |
| tools审批 | 0.50 | 0.50 | 0 |
| sandbox外接 | **1** | 1 | 0 |
| mcp凭证配置 | **1** | 1 | 0 |
| **等权总分 (max=11)** | **9.93** | **5.94** | **0.00** |

```
Hermes   = (0.43 + 1 + 1 + 1) + (1 + 1 + 1 + 1) + (0.5 + 1 + 1) = 3.43 + 4 + 2.5 = 9.93
OpenClaw = (1 + 0.01 + 0.09 + 0.35) + (1 + 0 + 1 + 0) + (0.5 + 1 + 1) = 1.44 + 2 + 2.5 = 5.94
DeerFlow = (0 + 0 + 0 + 0) + (0 + 0 + 0 + 0) + (0 + 0 + 0) = 0
```

差距集中在两组维度:

**资源效率** (HM 3.43 vs OC 1.44):
- 空载内存: 71M vs 828M (11.6× 差距)
- 空载CPU: 0.05% vs 8.80% (176× 差距)
- 启动时间: 6.9s vs 20s (2.9× 差距)
- 镜像: 7.28G vs 3.13G (OC 胜出)

**Event 覆盖** (HM 4/4 vs OC 2/4):
- Hermes 唯一支持 user.interrupt + user.custom_tool_result 的候选产品

#### 选型决策

| 备选方案 | 优点 | 风险/缺点 | 选择 |
|---------|------|----------|:--:|
| **Hermes** | 资源效率远超(CPU 176× 内存 11.6×); Event 4/4 全覆盖; 自有Python Loop | 镜像 7.28G 偏大; 每Agent独立进程 | **✅** |
| OpenClaw | 镜像 3.13G 最小; mcp 凭证立即生效 | 空载 828M 较重; Event 缺 2 项; pi-agent-core 外部依赖; 安全风险高，政企接受度低 | ❌ |
| DeerFlow | — | 11 维度全部数据未采集; 社区活跃度差距大 (64k ★) | ❌ |

#### 方案对业务的约束

| 约束项 | 影响 | 缓解 |
|--------|------|------|
| 每Agent=独立进程 | 需进程编排+健康检查 | N×71M 空载可控 |
| 镜像 7.28G | 分发下载耗时 | Docker multi-stage 分层构建 |
| mcp 热加载需重启 | 变更凭证需/reload-mcp | 纳入控制面管理流程 |

---

## 多session 否决记录

| 方案 | 否决原因 |
|------|---------|
| OpenClaw | tools审批不支持实例级隔离; mcp key不支持实例级隔离 |
| Hermes | system_prompt / skill / mcp / tools / model 五项全全局共享 |
| DeerFlow | mcp 仅支持全局配置，不支持 Agent 级别隔离 |
