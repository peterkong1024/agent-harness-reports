# Cursor SDK → CMA 落地方案

> 基于垂直能力解构报告, 对标 Anthropic CMA 九大核心概念
> 分析日期: 2026-05-04
> 特殊性: 闭源分析 — 源码不可访问, CMA 对齐基于类型定义推断

---

## 0. 前置说明

Cursor SDK 是**闭源商业产品**, CMA 对齐分析受到根本性限制:
- Agent 核心实现 (`@anysphere/agent-core`) 不可见
- Cloud 组件 (Cursor Server) 不可自托管
- 源码不可修改

此分析基于 npm 分发的编译类型定义 (.d.ts) 进行最大努力推断。

---

## 1. CMA 九大核心概念对齐

### 1.1 Agent

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 模板定义 | 🟡 需补齐 | `AgentOptions.agents: Record<string, AgentDefinition>` 提供声明式子Agent, 但无独立模板 DSL |
| CRUD | 🟡 需补齐 | Cloud Agent: `create/list/get/delete` 已有, 但 Local Agent 无注册表 |
| 实例化 | ✅ 已对齐 | `createCloudAgent`/`createLocalExecutor` |
| 版本管理 | 🔴 不存在 | 无 Agent 版本概念 |
| 权限绑定 | 🔴 不透明 | Sandbox Options 存在但细节未知 |

### 1.2 Environment

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 环境定义 | 🟡 需补齐 | `sandboxOptions` 参数存在 |
| Sandbox 绑定 | ⚠️ 不透明 | sandbox-helper artifacts 表明存在, 但闭源不可验证 |

### 1.3 Session

| 属性 | 对齐状态 | 详情 |
|------|---------|------|
| 创建/状态 | 🟡 需补齐 | `Run` 管理提供基本 session 抽象 |
| Sandbox 绑定 | ⚠️ 不透明 | 未知 |

### 1.4-1.9 (由于闭源限制, 仅作推断对齐)

| CMA 概念 | 对齐状态 | 详情 |
|---------|---------|------|
| Event | 🟡 | `RunEventTailer` + `RunEventStore` 提供事件管理 |
| Resource | 🔴 | 无 Resource URI |
| Vault | 🔴 | API Key 通过代码传递, 无 Vault |
| File | 🟡 | Artifact API (`listArtifacts`/`downloadArtifact`) 覆盖基本文件操作 |
| Skill | 🔴 | 无 Skills 系统 |
| MCP | ✅ | `McpServerConfig` + `cloud-mcp-utils` + `mcp-agent-exec` |

---

## 2. 架构模式判定

**无法判定** (闭源)。Cursor SDK 依赖 Cursor Cloud Server, 无法独立运行完整 CMA 栈。

---

## 3. 核心限制

| 限制 | 影响 |
|------|------|
| 源码闭源 | 安全审计不可行 |
| Cloud 依赖 | 无法自托管 |
| 许可证限制 | 仅 Cursor IDE 用户可用 |
| 无插件 SDK | 社区扩展受限 |

**结论**: Cursor SDK 不适合作为独立 CMA 数据面。更适合作为 IDE 内 Agent 集成点。
