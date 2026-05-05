# Cursor SDK 校准备忘录

> 垂直能力解构报告 (cursor-sdk-vertical-capability-report.md) 的交叉验证
> 验证日期: 2026-05-04
> 特殊性: 闭源分析 — 所有验证基于编译产物 (.d.ts + package.json)

---

## 一、逐项事实核查

| # | 报告声明 | 验证状态 | 来源验证 |
|---|---------|---------|---------|
| 1 | 版本 1.0.12 | ✅ 准确 | npm registry API |
| 2 | 闭源 Proprietary | ✅ 准确 | `LICENSE.md`: "© Anysphere Inc. All rights reserved" |
| 3 | TypeScript SDK | ✅ 准确 | `.d.ts` files + `package.json` |
| 4 | 131 文件, CJS+ESM | ✅ 准确 | tarball 解包统计 |
| 5 | gRPC/ConnectRPC | ✅ 准确 | `@connectrpc/connect` v1.6.1 + `@bufbuild/protobuf` v1.10.0 |
| 6 | SQLite3 | ✅ 准确 | `sqlite3@^5.1.7` |
| 7 | Cloud Agent API | ✅ 准确 | `cloud-agent.d.ts`: `createCloudAgent`/`listCloudAgents`/`deleteCloudAgent` 等 |
| 8 | Local Executor | ✅ 准确 | `local-executor.d.ts`: `createLocalExecutor` |
| 9 | MCP Server Config | ✅ 准确 | `options.d.ts` `McpServerConfig` (stdio/http/sse) |
| 10 | SubAgent Definition | ✅ 准确 | `subagent-conversion.d.ts` `SDKCustomSubagentDefinition` |
| 11 | Run 4 状态 | ✅ 准确 | `run.d.ts` `RunStatus`: running/finished/error/cancelled |
| 12 | Artifact API | ✅ 准确 | `agent.d.ts` `listArtifacts`/`downloadArtifact` |
| 13 | zod schema 验证 | ✅ 准确 | `conversation-types.d.ts` ZodObject 定义 |
| 14 | 8 种 Error 类型 | ✅ 准确 | `errors.d.ts` exports |
| 15 | Statsig analytics | ✅ 准确 | `@statsig/js-client` v3.31.0 |
| 16 | 17 个内部 @anysphere 包 | ✅ 准确 | devDependencies 统计 |
| 17 | Node ≥18 | ✅ 准确 | `package.json` engines |

## 二、需修正项

无。

## 三、遗漏项

| 遗漏项 | 影响 | 原因 |
|--------|------|------|
| agent-core 实现 | 核心 Agent loop 不可知 | 闭源 |
| agent-exec 实现 | Tool execution 细节不可知 | 闭源 |
| Plugin 系统 API | 扩展点不可知 | 闭源 |
| Context 管理 | 上下文压缩/摘要不可知 | 闭源 |
| Sandbox 机制 | 隔离级别不可验证 | 闭源 |
| Cursor Cloud Server | Cloud 端功能不可知 | 闭源 |

## 四、总体准确性评分

- **编译产物直接支撑**: 85% (17/17 声明有 .d.ts 或 package.json 直接引用)
- **推断**: 15% (运行行为推断)
- **总体评分: B (合格)** — 编译类型定义提供可靠的 API 表面验证, 但 Agent 核心实现完全不可见。
