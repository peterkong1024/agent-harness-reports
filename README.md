# Agent Harness 产品横向对比报告

> 面向「类 CMA (Claude Managed Agents) 企业 Agent 管理系统」选型的技术尽职调查

## 概述

本仓库包含 5 款主流开源 Agent Harness 产品的深度技术分析报告，覆盖从垂直能力解构、横向对比矩阵到 CMA 落地方案的全流程。所有声明均标注源码出处，遵循"特征逆向挖掘法"——不预设评价维度，从源码/文档/社区反馈反向推导产品能力。

## 分析对象

| 产品 | GitHub | Stars | 语言 | 定位 |
|------|--------|------:|------|------|
| **OpenClaw** | [openclaw/openclaw](https://github.com/openclaw/openclaw) | 368k | TypeScript | Personal AI Assistant Platform |
| **Hermes Agent** | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) | 132k | Python | Self-improving AI Agent |
| **DeerFlow 2.0** | [bytedance/deer-flow](https://github.com/bytedance/deer-flow) | 64k | Python | Super Agent Harness |
| **Cursor SDK** | [cursor/cursor](https://github.com/cursor/cursor) | 33k | TypeScript | IDE-native Agent SDK (闭源) |
| **Deep Agents** | [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents) | 22k | Python | Batteries-included Agent Harness SDK |

## 方法论

### 特征逆向挖掘法 (Feature Reverse Mining)

```
数据来源权威性层级:
  1. 源码 (.py/.ts 实现代码 — 最权威)
  2. 源码内配置与开发文档 (config.yaml, CLAUDE.md, docs/)
  3. 官方文档 (README.md, 官方站点)
  4. 推断 (标注 "推断: ...")
```

不预设评价维度，通过扫描源码目录结构、依赖声明、配置文件 Schema、CI/CD pipeline 和 GitHub Issues，自底向上提取 360+ 条原子特征，再通过语义聚类归一化为 6 个核心评价维度。

### 三阶段执行流程

| 阶段 | 名称 | 核心目标 | 产出 |
|------|------|---------|------|
| **Phase 1** | 垂直解析 | 单品深度解构 | 5 份垂直能力报告 + 校准备忘录 |
| **Phase 2** | 聚类建模 | 基准维度确立 | 6 维度评价框架 + 四级成熟度标准 |
| **Phase 3** | 横向映射 | 选型矩阵输出 | 横向对比报告 + 3 场景加权评分 |

### 六级评价维度

| 维度 | 覆盖范围 |
|------|---------|
| **D1. 通信广度** | IM 渠道数、协议抽象、流式输出、Voice/A2UI |
| **D2. 执行深度** | Shell/Filesystem/Browser、Sandbox 后端、沙箱生命周期 |
| **D3. 任务编排** | SubAgent 委派、并行执行、Plan/Todo、Loop 检测、上下文压缩 |
| **D4. 安全隔离** | DM 配对、Sandbox 隔离、工具白名单、SSRF 防护、多租户 |
| **D5. 记忆系统** | Session 持久化、Memory 系统、Session Search、自我完善 |
| **D6. 扩展生态** | Skills/Plugin SDK、MCP/ACP、Provider 生态、社区 Hub |

## 文件清单与阅读顺序

### 📊 横向对比报告 (入口)

| 文件 | 说明 |
|------|------|
| `harness-agent-cross-comparison-report.md` | **核心报告**。含 6 维度 × 5 产品对比矩阵、3 场景加权评分、CMA 工程核查、Benchmark 摸底 |

### 📋 垂直能力解构报告 (单品深度)

| 文件 | 产品 | Stars |
|------|------|------:|
| `openclaw-vertical-capability-report.md` | OpenClaw | 368k |
| `hermes-vertical-capability-report.md` | Hermes Agent | 132k |
| `deerflow-2.0-vertical-capability-report.md` | DeerFlow 2.0 | 64k |
| `cursor-sdk-vertical-capability-report.md` | Cursor SDK | 33k |
| `deepagents-vertical-capability-report.md` | Deep Agents | 22k |

每份垂直报告包含：
- **产品画像**: 定位、技术栈、数据来源层级
- **架构图提取**: 优先使用官方 ASCII 架构图
- **原子化特征提取表**: 8 维度 × 5 列 (特征/方案/效果/独特性/来源)
- **技术链路验证**: 正常路径 + 异常路径执行流
- **Gap Analysis**: 三源交叉验证 (Issues + 源码 + 产品对比)
- **多租户/Memory/沙箱补充核查**
- **评测体系摸底**

### ✅ 校准备忘录

| 文件 | 产品 |
|------|------|
| `openclaw-calibration-memo.md` | OpenClaw |
| `deepagents-calibration-memo.md` | Deep Agents |
| `cursor-sdk-calibration-memo.md` | Cursor SDK |

每份备忘录包含：逐项事实核查表、修正建议、总体准确性评分。

### 🏗️ CMA 落地方案

| 文件 | 产品 | 类型 |
|------|------|------|
| `deerflow-cma-control-plane-plan.md` | DeerFlow 2.0 | 管控面+数据面架构 |
| `openclaw-cma-implementation-plan.md` | OpenClaw | 修订版 (集成工程核查) |
| `deepagents-cma-implementation-plan.md` | Deep Agents | 修订版 (集成工程核查) |
| `hermes-cma-implementation-plan.md` | Hermes Agent | 修订版 (集成工程核查) |

每份 CMA 方案包含：
- CMA 九大概念对齐 (Agent/Environment/Session/Event/Resource/Vault/File/Skill/MCP)
- 沙箱并入 Environment 概念 (生命周期/外接/传递)
- 配置更新机制 (热重载/API/文件)
- CMA 工程可行性核查 (Agent 多实例、子资源隔离、沙箱外接、Skill 传播)
- 子资源关联关系分析
- 架构模式判定 (A/B 模式)
- 工作项与工作量评估
- 实施路线图

## 核心结论

### 场景加权排名

| 排名 | 个人 AI 助手 | 企业 Agent 平台 | 开发者 SDK |
|:----:|:-----------:|:------------:|:----------:|
| 1 | Hermes Agent 3.60 | Hermes Agent 3.40 | Hermes Agent 3.80 |
| 2 | OpenClaw 3.05 | DeerFlow 2.0 3.10 | OpenClaw 3.40 |
| 3 | DeerFlow 2.0 2.95 | OpenClaw 3.00 | DeerFlow 2.0 3.25 |

### 选型推荐

| 场景 | 推荐 | 理由 |
|------|------|------|
| 企业 Agent 管理系统 (类 CMA) | Hermes Agent / DeerFlow 2.0 | 编排+记忆全优，二者均需补多租户 |
| 个人 AI 助手 | OpenClaw / Hermes Agent | OpenClaw 渠道最广，Hermes 能力最全 |
| 开发者 Agent SDK | Deep Agents / Hermes Agent | Deep 编排最强，Hermes 通用最全 |

### 共性短板

- **所有产品均无真正的多租户**（租户级资源配额 + 配置分离 + 数据隔离）
- **所有产品均无公开的横向对比 benchmark 得分**

## 修订记录

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-05-04 | v1 | Phase 1 垂直报告 + 初步 CMA 方案 |
| 2026-05-05 | v2 | 补充多租户/Memory/沙箱生命周期核查 + OpenClaw 官方文档纳入 |
| 2026-05-05 | v3 | CMA 方案修订 (沙箱并入 Environment + 5 工程问题集成) + Benchmark 摸底 |
| 2026-05-05 | v4 | OpenClaw D3 评分修正 (2→4) — 源码 CHANGELOG 深度核查 |

## 许可

本仓库中的分析报告和方法论仅供技术选型参考。报告中引用的源码、文档版权归各产品所属组织所有。
