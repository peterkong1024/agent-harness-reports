# CMA-like Agent 管理平台 — 方案报告

> 构建兼容 Anthropic CMA API 规范的开源 Agent 管理平台的技术方案。

---

## 核心结论

**数据面引擎选型: Hermes Agent 胜出**

基于 11 维度等权打分: Hermes 9.93 vs OpenClaw 5.94 vs DeerFlow 0.00

| 关键差距 | Hermes | OpenClaw |
|---------|:------:|:--------:|
| 空载内存 | **71M** | 829M (11.6×) |
| 空载CPU | **0.05%** | 8.80% (176×) |
| 启动时间 | **6.9s** | 20s (2.9×) |
| Event 覆盖 | **4/4** | 2/4 |
| Agent Loop | 自有 Python | 外部 pi-agent-core |
| 安全 | Tirith 审计 | 风险较高，政企接受度低 |

OpenClaw / DeerFlow 多session 路径因隔离不满足一票否决。

**架构方案: 三面分离**

```
管理面 (定义层)  →  K8s Operator + FastAPI, 9大CMA资源CRUD
控制面 (编排层)  →  Session FSM + SSE事件总线 + HITL审批 + SandboxCTRL
数据面 (执行层)  →  Hermes Runtime × N, per-Agent task_id隔离
```

---

## 交付文档

| 文档 | 内容 |
|------|------|
| `cma-dataplane-solution-selection.md` | 数据面选型: 6方案→3方案等权11维打分→Hermes胜出 |
| `cma-platform-architecture-design.md` | 架构方案: 4+1视图 + 模块拆分 + 数据流 + ADR + 53人天 |
| `cma-platform-architecture.html` | SVG架构图: 三面分层拓扑 |
| `cma-api-resource-model.md` | CMA API: 9大资源模型解析 (anthropic-cma.yaml) |
| `开源agent与cma概念对齐.xlsx` | 实测数据: 3产品CMA概念对齐 + 资源占用 |

---

## 架构图预览

```
CMA Client (HTTPS)
    │
    ▼
Ingress (TLS + x-api-key)
    │
    ├─ /agents, /skills... → Management Plane (K8s Operator + etcd)
    │
    ├─ /sessions, /events/stream → Control Plane (FSM + SSE + Redis)
    │       │
    │       ▼ Hermes Internal (Python同进程, 零RPC)
    │
    └─→ Data Plane: Hermes Agent × N
            Agent-A (task_id=agent_aaa)
            Agent-B (task_id=agent_bbb)
            Agent-C (task_id=agent_ccc)
            │
            └─ Sandbox (per-Agent): Docker / gVisor / SSH
```

---

## 关键指标

| 指标 | 值 |
|------|-----|
| 管理面 API | 兼容 Anthropic CMA OpenAPI 3.1 (9资源) |
| 控制面 Event | SSE 18类事件全覆盖 |
| 数据面部署 | 每Agent 71M 空载, 6.9s 启动 |
| 工作量 | 53 人天 (管理8+控制21+数据11+部署5+测试8) |
| 参考实现 | stainlu/openclaw-managed-agents (406★) |
