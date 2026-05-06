# Agent Harness 产品横向对比报告 v2

## Phase 2: 聚类建模 + Phase 3: 横向映射

> 产出日期: 2026-05-06
> 分析对象: OpenClaw / Deep Agents / DeerFlow 2.0 / Hermes Agent / Cursor SDK / Claude Code / Codex / OpenCode
> 数据基础: 8 份源码级垂直能力解构报告 (Phase 1 产出)
> 方法论: 特征归一化 + 亲和聚类 + 四级成熟度评分
> v2 变更: 精简结构 (移除 B.3-B.7)，B.2.2 每项支持特征标注实现方法与效果
> v2.1 变更: 新增 B.2.4 特征举证索引 — 每条特征标注垂直报告行号 (79% 覆盖率), 待收集清单
> v2.2 变更: 源码级补充 — Codex并行执行✅(tokio::spawn), OpenCode上下文压缩✅(compaction.ts), DeepAgents压缩量化配置
> v2.3 变更: 批量源码挖掘(7产品×7子代理) — 大面积补全待收集特征
> v2.4 变更: B.2.2 增强 — D1-D3 每格增至 实现+配置+量化效果 三段式，源码行号引用 + 配置示例 + 量化指标

---

# Part A: Phase 2 — 聚类建模 (Clustering & Modeling)

## A.1 方法论设计

### A.1.1 方法论选型依据

传统横向对比存在两大缺陷：

| 缺陷 | 传统做法 | 本方案改进 |
|------|---------|-----------|
| **先入为主** | 预定义维度再套入产品 | 从 8 份报告的 500+ 原子特征中自底向上聚类 |
| **布尔化** | "有/无"二元判断 | 四级成熟度评分 (0-4)，捕捉实现深度差异 |
| **等权重** | 所有维度一视同仁 | 场景驱动的动态权重分配 |
| **忽视冲突** | 不记录特征互斥/替代关系 | 显式标注"互斥特征对"，避免重复计数 |

### A.1.2 特征归一化流程

```
8 份垂直报告 × 8 维度 × 平均 8 个原子特征 ≈ 500 条原始特征
       │
       ▼
Step 1: 去重合并 — 合并跨产品的同名/同义特征
       产物: 标准化特征池 (~120 个唯一特征)
       │
       ▼
Step 2: 语义聚类 — 将相关特征聚合为候选维度
       │
       ▼
Step 3: 维度精简 — 合并强相关维度，保证 MECE
       产物: 6 个核心维度
       │
       ▼
Step 4: 成熟度定义 — 为每个维度定义 0-4 级评分标准
```

## A.2 六大核心评价维度

基于 500+ 条原始特征的归一化和语义聚类，得出以下 6 个互斥完备的维度：

### Dimension 1: 通信广度与渠道归一化 (Channel Breadth)

| 定义 | 评估 Agent 产品的消息渠道覆盖数量和归一化抽象质量 |
|------|--------------------------------------------------|
| 来源 | 原维度四「通信与适配」+ 原维度一中的"语音/A2UI"特征 |
| 核心关注 | IM 渠道数、协议抽象、流式输出质量、Voice/A2UI 支持 |

成熟度评分标准：

| 级别 | 标准 |
|:----:|------|
| 0 | 无独立通信层，仅 API/CLI 输入 |
| 1 | 1-3 个渠道，无统一抽象 |
| 2 | 3-10 个渠道，有基础的 channel plugin 架构 |
| 3 | 10+ 渠道，统一 inbound/outbound envelope 归一化，含流式卡片 |
| 4 | 20+ 渠道 + Voice + A2UI Canvas + Device Nodes，全渠道归一化 |

### Dimension 2: 工具深度与执行环境 (Execution Depth)

| 定义 | 评估 Agent 可执行的操作范围和执行环境的隔离/灵活程度 |
|------|-----------------------------------------------------|
| 来源 | 原维度二「操作与控制」+ 原维度五「安全与隔离」中的 Sandbox 部分 |
| 核心关注 | Shell/FS/Browser/Git 工具完备度、Sandbox 后端多样性、跨应用调用 |

成熟度评分标准：

| 级别 | 标准 |
|:----:|------|
| 0 | 仅 Shell 或 API-only |
| 1 | Shell + 基础文件读写，无 Sandbox |
| 2 | Shell + FS + Browser/Git，单种 Sandbox 后端 |
| 3 | 完整工具集 + 2+ Sandbox 后端 + ACP 跨 Agent 调用 |
| 4 | 完整工具集 + 3+ Sandbox 后端 + 细粒度工具白名单 + 虚拟路径翻译 |

### Dimension 3: 智能决策与任务编排 (Orchestration)

| 定义 | 评估 Agent 的任务分解、并行执行、规划质量和异常恢复能力 |
|------|-------------------------------------------------------|
| 来源 | 原维度三「规划与决策」 |
| 核心关注 | SubAgent 委派、并行执行、Plan/Todo、Loop 检测、压缩/总结 |

成熟度评分标准：

| 级别 | 标准 |
|:----:|------|
| 0 | 单轮工具调用，无分解能力 |
| 1 | 基础 SubAgent/Task 委派（串行），无 Plan Mode |
| 2 | SubAgent + Todo/Plan + 并行执行（2-3 并发） |
| 3 | SubAgent(3+) + Todo + Loop 检测 + 上下文自动压缩 |
| 4 | 全功能 SubAgent(含远程) + 复杂 Middleware Stack(10+) + 多模型自适应 |

### Dimension 4: 安全与多租户隔离 (Security & Tenancy)

| 定义 | 评估 Agent 的安全边界、隔离机制和多租户管理能力 |
|------|-----------------------------------------------|
| 来源 | 原维度五「安全与隔离」(除 Sandbox 后端外) + 平台判定标准「隔离化」 |
| 核心关注 | DM 配对、工具白名单、SSRF 防护、Vault/Secret 管理、容器隔离 |

成熟度评分标准：

| 级别 | 标准 |
|:----:|------|
| 0 | 无安全机制，host 直连 |
| 1 | 基础 API Key 管理 + 环境变量，无 DM 配对 |
| 2 | DM 配对 + 基础沙箱 + 工具白名单 |
| 3 | DM 配对 + 多沙箱 + 工具白/黑名单 + SSRF 防护 + 威胁检测 |
| 4 | 全功能安全 + Vault/Secret 轮换 + 审计日志 + 多租户 RBAC |

### Dimension 5: 持久化与记忆系统 (Memory & State)

| 定义 | 评估 Agent 的状态持久化、跨会话记忆和学习能力 |
|------|---------------------------------------------|
| 来源 | 原维度六「持久化与状态管理」+ 平台判定标准「模板化」 |
| 核心关注 | Session 持久化、Memory 系统、Skills 生命周期、配置声明式 |

成熟度评分标准：

| 级别 | 标准 |
|:----:|------|
| 0 | 无状态，每次对话独立 |
| 1 | 基础 session 持久化 (文件/SQLite)，无搜索 |
| 2 | Session + Vector Memory + 配置文件 |
| 3 | Session + Memory + Agent 记忆文件 (AGENTS.md) + 自动压缩 |
| 4 | 全功能 + Session Search (FTS5) + 自动 Skill 创建/管理 + 自我完善闭环 |

### Dimension 6: 扩展性与生态开放性 (Extensibility)

| 定义 | 评估 Agent 的扩展机制、插件生态和社区活跃度 |
|------|-------------------------------------------|
| 来源 | 原维度八「扩展性与生态」+ 原维度七「工程化与可观测」中的 CI/测试社区部分 |
| 核心关注 | Skills/Plugin SDK、MCP/ACP 支持、Provider 生态、社区 Skills Hub |

成熟度评分标准：

| 级别 | 标准 |
|:----:|------|
| 0 | 无扩展机制 |
| 1 | 基础自定义工具/函数注册 |
| 2 | Skills 系统 + MCP Client + 5-10 Provider |
| 3 | Skills/Plugin SDK + MCP 双向 + ACP + 20+ Provider + 社区 Hub |
| 4 | 80+ Plugin 导出 + MCP/ACP 双向 + Skills Hub + Plugin Registry + 渠道扩展 API |

---

# Part B: Phase 3 — 横向映射 (Horizontal Mapping)

## B.1 评分方法论

### B.1.1 评分流程

```
六大维度 × 成熟度标准 (0-4)
       │
       ▼
逐产品评分 — 基于垂直报告的原子特征表 + Gap分析 + 校准备忘录
       │
       ▼
交叉验证 — 消除主观偏差
       │
       ▼
最终排序
```

### B.1.2 评分制约因素

| 制约因素 | 影响 | 适用产品 |
|---------|------|---------|
| **闭源** | 部分维度评分降 0.5 级 (安全可审计性、扩展深度无法验证) | Cursor SDK |
| **依赖外部 Agent 框架** | 核心 Agent loop 不可控，降 0.5 级 | OpenClaw (依赖 PI Agent) |
| **纯 SDK / 无独立运行时** | Channel Breadth 自动 0 分 | Deep Agents、Cursor SDK |

## B.2 横向对比矩阵

### B.2.1 总览矩阵

| 维度 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|:--------:|:-----------:|:------------:|:------------:|:----------:|:-----------:|:-----:|:-------:|
| **D1. 通信广度** | ████ 4 | ██ 1 | ███ 3 | ████ 4 | █ 0 | ██ 2 | █ 1 | ███ 3 |
| **D2. 执行深度** | ███ 3 | ██ 2 | ████ 4 | ████ 4 | ██ 2 | ████ 4 | ███ 3 | ███ 3 |
| **D3. 任务编排** | ████ 4 | ████ 4 | ███ 3 | ████ 4 | ██ 2 | ████ 4★ | ███ 3 | ███ 3 |
| **D4. 安全隔离** | ██ 2 | ██ 2 | ███ 3 | ██ 2 | █ 1 | ████ 4 | ████ 4 | ███ 3 |
| **D5. 记忆系统** | ██ 2 | ███ 3 | ██ 2 | ████ 4 | █ 1 | ████ 4 | ███ 3 | ██ 2 |
| **D6. 扩展生态** | ████ 4 | ███ 3 | ███ 3 | ████ 4 | ██ 2 | ████ 4 | ██ 2 | ████ 4 |
| **平均分** | **3.17** | **2.50** | **3.00** | **3.67** | **1.33** | **3.67** | **2.67** | **3.00** |
| **社区规模** | 368k ★ | 22k ★ | 64k ★ | 132k ★ | 33k ★ | 泄露分析 | 80k ★ | 155k ★ |

> ★ Claude Code D3 实际能力超越 4 分体系（三套 Multi-Agent 并存），但评分体系上限为 4。

### B.2.2 逐维度详细对比

> 标记规则: **实现** 描述技术方案和源码出处；**配置** 说明如何启用/配置该特征；**效果** 描述可达成的用户可感知结果（含量化指标）。
> `-` = 不支持；`❓` = 推断（未完整读取源码）；`⟳` = 已源码核实确实不支持。

#### D1: 通信广度与渠道归一化

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| IM 渠道数 | Agent 触达用户的平台范围 | ✅ 实现: 24+ 原生 Channel Plugin (Telegram/Discord/Slack/WhatsApp/Matrix/Signal/Line/微信/iMessage)。配置: `openclaw.json`→`channels` 数组声明，每 channel 独立 TS 模块。效果: 任意常用 IM 中 @Agent 即可交互 | - 纯 CLI only | ✅ 实现: 飞书/钉钉/企业微信/Slack/Telegram/Discord bridge。配置: `config.yaml`→`channels[]`。效果: 国内+国际企业IM全覆盖，飞书流式卡片+钉钉AI Card | ✅ 实现: Gateway统一协议层(WebSocket/HTTP/MQTT/ACP)。配置: `config.yaml`→`gateway.channels`。效果: 新增渠道仅需实现adapter，18+平台 | - IDE内嵌 | - CLI/TUI only (Ink React) | - CLI only (Rust TUI) | ✅ 实现: CLI(23命令)+TUI+Web(SolidJS)+Desktop(Electron)+Slack Bot。各自独立package。效果: 四端覆盖 |
| 流式输出 | 打字机效果实时反馈 | ✅ 实现: draft-stream-loop.ts节流草稿循环，按throttleMs间隔sendOrEditStreamMessage。配置: `streaming.mode: partial/chunk`。效果: 编辑消息模式，非逐token推送 | ✅ 实现: TUI终端流式 | ✅ 实现: 飞书流式卡片/钉钉AI Card(server push)。效果: 原生富交互卡片，支持中断+重试 | ✅ 实现: KawaiiSpinner+SSE。效果: 带动画终端流式 | ✅ 实现: SSE | ✅ 实现: Ink TUI流式重渲染 | ✅ 实现: sse.rs→stream-parser→turn.rs→TUI五层管道。效果: 逐delta实时，Plan模式分离 | ✅ 实现: TUI+WebSocket Desktop |
| Voice | 语音唤醒+对话 | ✅ 实现: VoiceWake(唤醒词)+Talk(TTS)+Realtime Relay(Gemini Live)。配置: `voicewake.json`触发词(默认["openclaw","claude","computer"])+`voicewake-routing.json`路由(最多32条)。效果: 唤醒词激活+路由到指定Agent+实时语音 | - | - | ✅ 实现: TTS 5 providers+faster-whisper STT。配置: `ctrl+b`录音(120s,静音检测)。效果: 本地STT不依赖云端 | - | - | - | - |
| A2UI Canvas | Agent动态生成交互Web UI | ✅ 实现: Canvas组件+Widget渲染。效果: 图表/表单/仪表盘富交互 | - | - | - | - | - | - | - |
| Device Nodes | Camera/Screen/Location | ✅ 实现: Camera/Screen/Location三个节点。效果: Agent感知物理世界 | - | - | - | - | - | - | - |
| 多入口/远程 | 非CLI入口(SDK/Bridge/Remote) | ✅ 实现: WebChat+Desktop(macOS/iOS/Android)+`hermes connect`。效果: 消费级入口 | - | ✅ 实现: Web UI+HTTP API+ACP Server。效果: 浏览器+API双入口 | ✅ 实现: Gateway+MCP Server+ACP。效果: 可作为MCP Server被外部Agent调用 | ✅ 实现: Cloud Agent gRPC | ✅ 实现: SDK/MCP/Bridge/Remote六种入口。效果: `/resume`远程会话恢复 | - | ✅ 实现: Client/Server+`opencode serve --port 4096`+`opencode attach <url>`。效果: 远程TUI attach |

| **得分** | | **4** | **1** | **3** | **4** | **0** | **2** | **1** | **3** |
|---------|------|-------|-------|--------|--------|---------|---------|-------|--------|

#### D2: 工具深度与执行环境

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Shell 执行 | Agent 执行系统命令能力 | ✅ 实现: child_process + sandbox隔离(Docker/SSH/OpenShell)。效果: 沙箱内任意命令 | ✅ 实现: `execute` tool, SandboxBackendProtocol, 默认120s timeout。配置: 5种Backend实现。效果: `background=True`后台执行 | ✅ 实现: bash tool+虚拟路径翻译+LocalSandboxProvider默认禁用host bash。配置: 四层隔离控制可达范围，输出中间截断(20,000 chars)。效果: 路径自动映射+防Token爆炸 | ✅ 实现: 6种终端后端(local/docker/ssh/daytona/modal/singularity)。配置: `config.yaml`→`tools.terminal.backend`。效果: PTY交互式支持 | ✅ 实现: IDE Shell | ✅ 实现: Shell.ts (bwrap封装)。效果: Bubblewrap容器内执行，文件+网络隔离 | ✅ 实现: Rust std::process+Landlock LSM (`spawn_command_under_linux_sandbox`, `core/src/lib.rs:47-48`)。效果: Linux 5.13+内核级LSM沙箱 | ✅ 实现: node-pty伪终端。效果: 支持vim/npm init等交互式CLI |
| 文件操作 | 编辑器级文件读写 | ✅ 实现: fs promises+原子写入 | ✅ 实现: 6种Backend(State/FS/Shell/Store/Composite)虚拟文件系统 | ✅ 实现: read_file(偏移+截断)/write_file(自动mkdir)/str_replace(per-sandbox锁)。配置: 按(sandbox.id,path)序列化防并发。效果: 编辑器级精确替换 | ✅ 实现: read_file(行号)/search_files(ripgrep)/write_file(自动mkdir)。效果: 带行号+regex搜索 | ✅ 实现: IDE文件API | ✅ 实现: 文件操作tool | ✅ 实现: Rust std::fs原生IO | ✅ 实现: AppFileSystem(Effect-TS)类型安全。效果: acquire/release管理句柄生命周期 |
| Browser | Agent操控真实浏览器 | ✅ 实现: Playwright集成。效果: 点击/截图/表单填写 | - | - | ✅ 实现: 12个 browser_* tools(Playwright/CDP)。效果: 导航/点击/输入/截图/对话框/iframe+coordinate click | - | - | - | ✅ 实现: agent-browser CLI。效果: `snapshot -i`获取交互元素ref |
| Sandbox 后端 | Agent执行隔离级别多样性 | ✅ 实现: Docker/SSH/OpenShell(3种)。配置: `openclaw.json`→`sandbox.mode`+`sandbox.scope` | ✅ 实现: 5种Backend(State/FS/Shell/Store/Composite)。效果: 执行虚拟化非容器 | ✅ 实现: 四层隔离(Session/Request/Resource/Network)。效果: 纵深防御，每层独立配置 | ✅ 实现: Docker/SSH/Daytona(3种)+daemon管理+persistent开关。效果: daemon自动空闲清理，容器可持久化 | ❓ 闭源 | ✅ 实现: bwrap+Seatbelt+macOS runtime(3层)+SandboxDoctor诊断。效果: 跨平台双重隔离+健康检查 | ✅ 实现: Landlock(Linux 5.13+ LSM内核级)+bwrap容器。效果: 已知产品唯一内核级LSM沙箱 | ✅ 实现: containers/ package(Docker+预构建bun-node/rust/tauri-linux镜像) |
| ACP 跨Agent | 调用外部Agent作为子执行器 | ✅ 实现: ACP client+subagent ACP transport。效果: 可调用任何ACP Agent | ✅ 实现: ACP transport+远程SubAgent | ✅ 实现: `invoke_acp_agent`+workspace挂载`/mnt/acp-workspace`。效果: Lead Agent委派Claude Code/Codex CLI | ✅ 实现: ACP server+delegate_task ACP | - | ✅ 实现: Swarm/Remote模式。效果: 多Agent跨进程协作 | ✅ 实现: codex_delegate.rs(846行)SubAgentSource+agent-graph-store图拓扑 | ✅ 实现: @agentclientprotocol/sdk 0.16.1+acp/(1982行) |
| Git 集成 | Agent自动git commit/checkpoint | - | - | - | ✅ 实现: Checkpoint tool(git commit+push自动)。效果: 每步自动checkpoint可回滚 | ✅ 实现: git-core集成 | ❓ 推断 | ❓ 推断 | ✅ 实现: src/git/(352行)。效果: git状态/commit/diff |
| 虚拟路径翻译 | 沙箱路径与宿主自动映射 | - | - | ✅ 实现: 虚拟路径自动翻译。效果: 沙箱内操作不知宿主真实路径 | - | - | ❓ 推断 | ❓ 推断 | ❓ |
| 沙箱生命周期 | 创建/复用/销毁策略 | ✅ 实现: mode(off/non-main/all)+scope(agent/session/shared)+CLI(recreate/list/explain)。效果: 四级配置+CLI管理 | ✅ 实现: 5 Provider协议化+acquire/release/shutdown | ✅ 实现: acquire/release/shutdown+lazy_init+Thread复用+shutdown()。效果: 显式生命周期API | ✅ 实现: 6 Backend+daemon空闲清理+persistent开关。注意:`task_id="default"`全局共享 | ❓ 闭源 | ✅ 实现: cleanupAfterCommand+SandboxManager | ❓ 推断: Landlock进程级 | ❓ 推断: Docker生命周期 |
| 外置sandbox空闲管理 | session idle销毁/active重建 | ✅ 实现: scope(agent/session/shared)自动管理。效果: session结束scope触发清理，active时重建 | ⚠️ Provider决定生命周期，无统一idle管理 | ✅ 实现: lazy_init+Thread复用+shutdown()。效果: idle时释放sandbox，active时lazy重建 | ✅ 实现: `_cleanup_inactive_envs`+daemon空闲清理+`container_persistent`开关。配置: `container_persistent: true`(默认)。效果: daemon自动清理空闲容器，persistent=false时idle销毁 | 🔒 闭源 | ✅ 实现: cleanupAfterCommand+SandboxManager。效果: 命令结束后自动清理，下次执行重建 | ⚠️ Landlock进程级: 命令结束进程销毁，无idle/active概念 | ⚠️ Docker容器生命周期由docker-compose管理，无内置idle检测 |

| **得分** | | **3** | **2** | **4** | **4** | **2** | **4** | **3** | **3** |
|---------|------|-------|-------|--------|--------|---------|---------|-------|--------|

#### D3: 智能决策与任务编排

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| SubAgent 委派 | 复杂任务拆解为独立子Agent | ✅ 实现: sessions_spawn+subagent registry。配置: `agents.list[]`每Agent独立。效果: 可spawn多个subagent session，独立workspace | ✅ 实现: SubGraphAgent三种形态(声明式/预编译/远程)。配置: `SubGraphAgent`声明或`create_subagent`编程。效果: 最灵活定义，支持预编译graph | ✅ 实现: 双线程池(3+3)。配置: `max_concurrency=3`。效果: 主线程+副线程池并发 | ✅ 实现: delegate_task tool(max=3并行)。配置: `delegate_task(goal=...,context=...)`。效果: leaf/orchestrator角色，独立工具集 | ✅ 实现: customSubagents配置。效果: IDE内自定义 | ✅ 实现: AgentTool 6 built-in+SubAgent/Coordinator/Swarm三套并行。配置: `.claude/agents/*.md`声明式定义。效果: 已知最丰富Multi-Agent体系 | ✅ 实现: codex_delegate.rs(846行)+agent-graph-store图拓扑。配置: `CodexSpawnArgs{config,auth,model,env,skills,mcp}`。效果: 图状非树状SubAgent关系 | ✅ 实现: agent mode: subagent/primary/all。配置: `Agent.Info.mode` Effect Schema类型。效果: 三级角色+per-Agent permission |
| 并行执行 | 互不依赖子任务同时执行 | ✅ 实现: parallel subagent+grouped results。效果: 总耗时=max(子任务) | ✅ 实现: AsyncSubAgent+LangGraph并行节点。效果: 图式并行任意拓扑 | ✅ 实现: MAX=3并发限制。效果: 最多3并行 | ✅ 实现: delegate_task max=3。效果: 最多3并行 | ❓ | ✅ 实现: Swarm teammates并行。效果: Swarm模式全并行 | ✅ 实现: tokio::spawn并发event/op通道(codex_delegate.rs:130,144)。效果: 多子Agent异步并发 | - ⟳ |
| Plan/Todo | 自动创建Todo跟踪进度 | ✅ 实现: update_plan tool(experimental,opt-in)。配置: `agents.defaults.tools.experimental.planTool=true`。效果: 需显式开启 | ✅ 实现: TodoListMiddleware(MW栈第6层)。效果: 自动Todo生成更新 | ✅ 实现: TodoListMiddleware(MW栈第5层)。效果: 自动Todo | - | - | ✅ 实现: TaskCreate/TaskStop工具。效果: Agent自主创建/停止任务 | ✅ 实现: 三层体系: Goal(持久化+token预算)+PlanMode(协作planning阶段)+TodoList(update_plan回合清单)。配置: `create_goal`/`update_goal`/`get_goal`三工具+`UpdatePlanArgs{plan: PlanItemArg[]}`。效果: 持久化目标+回合清单+Plan模式三轨并行 | - |
| Loop 检测 | 防止Agent陷入死循环 | ✅ 实现: detectors(3种)+circuitBreaker+postCompactionGuard。效果: 三级检测+熔断+压缩后二次检查 | ✅ 实现: PatchToolCalls。效果: 工具调用修复即破坏循环 | ✅ 实现: LoopDetectionMW独立中间件。效果: 硬中断循环 | ✅ 实现: tool_loop_guardrails。效果: Guardrails内置检测 | ❓ 闭源 | ✅ 实现: autoCompact.ts circuit breaker(MAX_CONSECUTIVE_FAILURES=3)+blocking_limit(contextWindow-3000)。配置: `DISABLE_AUTO_COMPACT`环境变量。效果: 3次连续失败跳闸，每天节省~250K API调用 | ❓ | ✅ 实现: DOOM_LOOP_THRESHOLD=3(连续3次同名工具相同输入触发)+continue_loop_on_deny+maxSteps。配置: `agent.steps`+`experimental.continue_loop_on_deny`。效果: 5层防护(maxSteps+doomLoop+MCP审批循环+自然退出+deny策略) |
| 上下文压缩 | Token超限自动压缩 | ✅ 实现: auto-compaction+midTurnPrecheck+retry+notifier。效果: 自动触发+中途检查+重试+通知 | ✅ 实现: SummarizationMW(MW栈第11层)。配置: trigger=('fraction',0.85), keep=('fraction',0.10), model='gpt-4o-mini'。效果: 85%阈值触发，保留10%最近消息 | ✅ 实现: DeerFlowSummarizationMiddleware。配置: `summarization.enabled=false`(默认), trigger支持fraction/tokens/messages三种, trim_tokens_to_summarize=4000。效果: 启用后token超限摘要，skill文件保护 | ✅ 实现: compressor+自动触发(50%阈值)。效果: 半触发 | ❓ 闭源 | ✅ 实现: compact+hooks回调。效果: 可编程压缩，autoCompactThreshold=contextWindow-13000 tokens | ✅ 实现: compact_remote.rs(354行)+compact_remote_v2.rs。效果: 远程执行压缩，v1+v2双策略 | ✅ 实现: session/compaction.ts(652行)—isOverflow检测+SUMMARY_TEMPLATE(Constraints/Preferences/Critical Context)+prune(PRUNE_PROTECT=40000)。效果: tail_turns(默认2)保留原文，旧消息摘要 |
| 异常重启恢复 | crash后自动/手动恢复任务 | ❌ 无checkpoint/interrupt-on恢复 | ✅ 实现: LangGraph Checkpointer(SqliteSaver)。配置: `checkpointer=SqliteSaver.from_conn_string("checkpoints.db")`。效果: graph状态检查点，断点续跑 | ⚠️ 依赖LangGraph Checkpointer(SQLite/PostgreSQL)，无显式crash recovery命令 | ✅ 实现: `batch_runner.py:_load_checkpoint`+SQLite FTS5 session持久化。配置: `--resume`参数。效果: batch模式checkpoint恢复+session自动持久化 | 🔒 闭源(Cloud Agent支持resume) | ✅ 实现: `/resume`命令(Fuse.js模糊)+`--continue`参数+remote session URL恢复。配置: `CLAUDE_CODE_USE_CCR_V2=1`+`/resume`。效果: 模糊匹配+远程恢复+CLI续跑 | ⚠️ TurnState状态机+agent-graph-store持久化。配置: TUI Session Picker(分页25条)+Ctrl+R。效果: 无显式/resume，通过Session Picker间接恢复 | ⚠️ `session/retry.ts`+`session/revert.ts`+`session/run-state.ts`三层恢复。配置: `opencode attach <url>`远程TUI。效果: retry/revert/run-state会话级恢复 |
| 工具调用恢复 | 失败后patch/重试/降级 | ✅ 实现: orphan recovery+compaction retry+tool-result guard | ✅ 实现: PatchToolCalls自动修复参数 | ✅ 实现: LoopDetection hard stop即恢复 | ✅ 实现: Guardrails hard_stop | ❓ 闭源 | ✅ 推断: StreamingToolExecutor含重试 | ❓ | - |

| **得分** | | **4** | **4** | **3** | **4** | **2** | **4★** | **3** | **3** |
|---------|------|-------|-------|--------|--------|---------|---------|-------|--------|

#### D4: 安全与多租户隔离

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| DM 配对 | 未知用户首次 DM 需审批 | ✅ 实现: pairing默认启用+配对码机制。配置: `openclaw.json`→`channels`→`pairing`。效果: 首次DM必须配对码确认，防止未授权访问 | - | - | ✅ 实现: Gateway DM pairing。配置: `config.yaml`→`gateway.dm_pairing`。效果: 首次DM生成配对码 | - | - | - | - |
| Sandbox 隔离 | 执行环境与宿主物理/逻辑隔离 | ✅ 实现: Docker/SSH/OpenShell(3种)。配置: `sandbox.mode`(off/non-main/all)+`sandbox.scope`(agent/session/shared)。效果: 不同场景选不同隔离级别，CLI管理(recreate/list/explain) | ✅ 实现: 5种Backend(State/FS/Shell/Store/Composite)虚拟化执行。效果: 执行环境抽象非容器 | ✅ 实现: 四层隔离(Session/Request/Resource/Network)。配置: 每层独立开关。效果: 纵深防御，LocalSandboxProvider默认禁用host bash | ✅ 实现: Docker/SSH/Daytona(3种)+daemon管理+persistent开关。配置: `config.yaml`→`tools.terminal.backend`。效果: daemon自动空闲清理，持久化可选。注意:`task_id="default"`全局共享 | ❓ 闭源 | ✅ 实现: bwrap+Seatbelt+macOS runtime(3层)+SandboxDoctor诊断。配置: 四层sandbox自动检测平台。效果: 跨平台内核级+用户态双重隔离+健康检查 | ✅ 实现: Landlock(Linux 5.13+ LSM内核级)+bwrap容器(`core/src/lib.rs:47-48`)。配置: `CODEX_SANDBOX_NETWORK_DISABLED`环境变量。效果: 已知产品唯一内核级LSM沙箱，文件系统+网络双重控制 | ✅ 实现: containers/ package(Docker+预构建bun-node/rust/tauri-linux镜像)。效果: 多语言预构建容器 |
| 工具白名单 | 限制Agent可使用的工具 | ✅ 实现: Allow/Deny按Agent配置。配置: `agents.list[].tools` per-agent。效果: 每Agent独立工具白名单 | ⚠️ 实现: 文件权限检查(wcmatch.glob)。效果: 仅文件操作受控，非全工具白名单 | ✅ 实现: 四层权限控制(Session/Resource/Network级)。配置: guardrails AllowlistProvider。效果: 工具级allowlist/denylist | ✅ 实现: Guardrails工具拦截+tool_loop_guardrails。配置: `config.yaml`→`guardrails`。效果: 运行时工具调用拦截，循环检测 | ❓ 闭源 | ✅ 实现: bashPermissions+Tool Permission层。效果: 双层权限，命令级+工具级 | ✅ 实现: Guardian审批系统(`codex_delegate.rs:34-37`)。配置: MCP_TOOL_APPROVAL_ACCEPT/ACCEPT_FOR_SESSION/DECLINE_SYNTHETIC三级。效果: 工具调用需Guardian审批，会话级信任 | ✅ 实现: allow/deny/ask三级+DB持久化+pattern matching(`permission/index.ts:21`)。配置: `PermissionRule{permission,pattern,action}`。效果: 精确命令匹配(e.g. `bash:git push*`)+持久化选择 |
| SSRF 防护 | 防止Agent访问内网 | ✅ 实现: ssrf-policy中间件。配置: `openclaw.json`→`security.ssrfPolicy`。效果: 阻断内网IP访问(169.254.169.254等) | - | - | ✅ 实现: website_blocklist。配置: `config.yaml`→`security.website_blocklist`。效果: 黑名单URL拦截 | ❓ 闭源 | ✅ 实现: bwrap net namespace网络隔离。效果: bwrap层全网络隔离 | ✅ 实现: `CODEX_SANDBOX_NETWORK_DISABLED`环境变量。配置: 环境变量开关。效果: 全局网络禁用 | ✅ 实现: FenceMiddleware(`server/server.ts:14`)。效果: SSRF请求拦截，服务端中间件 |
| 工具审批 | 工具调用前人工审批(HITL) | ⚠️ DM配对(非受信用户需审批)，但工具审批不如Codex/OpenCode精细。配置: `dmPolicy: "pairing"` | ✅ 实现: HumanInTheLoopMiddleware: per-tool `interrupt_on`配置。配置: `interrupt_on={"tool_name": True}`。效果: 每工具独立HITL开关 | ✅ 实现: GuardrailMiddleware(AllowlistProvider/OAP/Custom三种Provider)。配置: `guardrails.provider: allowlist/oap/custom`。效果: 模式匹配审批 | ✅ 实现: `approval.py`多级审批: permanent approval+session级YOLO+gateway异步队列+Smart Approvals guardian(模仿Codex)。配置: `approvals.mode: manual/yolo/deny`+`command_allowlist`。效果: 四级审批粒度 | 🔒 闭源 | ✅ 实现: bashPermissions(命令级)+PermissionRule(工具级)+交互式弹窗审批(`SandboxPermissionRequest.tsx`)。效果: 双重审批+UI弹窗 | ✅ 实现: Guardian审批子系统(`codex_delegate.rs:34-37`): GuardianApprovalRequest→routes_approval_to_guardian→spawn_approval_request_review。效果: 独立审批子系统，approve/reject | ✅ 实现: allow/deny/ask三级+pattern matching+DB持久化(`permission/index.ts:21`)。配置: `PermissionRule{permission, pattern, action}`。效果: 精确命令匹配(`bash:git push*`)+持久化 |
| MCP审批 | MCP工具独立审批流程 | ❌ 无独立MCP审批 | ❌ MCP通过HITL middleware间接覆盖 | ❌ MCP通过langchain-mcp-adapters桥接无独立审批 | ❌ MCP工具直接注册无独立审批，command-level审批仅覆盖bash | 🔒 闭源 | ⚠️ MCP工具作为native tool包装，通过bashPermissions统一审批，无独立MCP通道 | ✅ 实现: 三级MCP审批(`codex_delegate.rs:38-43`): `MCP_TOOL_APPROVAL_ACCEPT`/`ACCEPT_FOR_SESSION`/`DECLINE_SYNTHETIC`。效果: 单次/会话/拒绝三级，**已知产品唯一独立MCP审批** | ⚠️ MCP使用统一Permission系统(非独立)，含MCP无限审批循环防护(`continue_loop_on_deny`)。配置: `experimental.continue_loop_on_deny` |
| 自定义工具 | 用户创建自定义工具 | ✅ 实现: 80+ plugin-sdk运行时导出(channel/auth/reply/sandbox/TTS)。效果: 几乎无不可扩展子系统 | ✅ 实现: Middleware API 13层可插拔+`tools`参数支持Callable/dict。配置: `create_deep_agent(tools=[my_tool])` | ⚠️ 反射式Provider加载(`reflection/__init__.py`)，无独立Plugin SDK。配置: `config.yaml`声明provider路径 | ✅ 实现: Skill定义工具(`required_credential_files`+`required_environment_variables`)+toolset组合。配置: SKILL.md YAML frontmatter+`toolsets.includes` | 🔒 闭源(Plugin存在但API不公开) | ✅ 实现: 184工具文件+Plugin架构。效果: 工具文件可扩展 | ⚠️ `core-plugins`+`core-skills`独立crate暗示插件化，细节待确认 | ✅ 实现: Plugin SDK(Hooks系统,2767行)+多认证插件。配置: `PluginInput`+`Hooks`接口。效果: `plugin/loader.ts`动态加载 |
| MCP凭证调用位置 | MCP凭证在sandbox还是Agent loop使用 | ✅ Sandbox内: mode(non-main)在Docker/SSH/OpenShell中执行 | ⚠️ Agent loop: FilesystemMiddleware权限，Sandbox后端仅执行抽象 | ✅ Sandbox内: 四层Sandbox隔离+thread_data_mounts挂载凭证 | ✅ Sandbox内: `credential_files.py`挂载到remote sandbox(`get_credential_file_mounts()`)+`env_passthrough.py`注册sandbox白名单(provider credential blocklist防护)。效果: **已知产品最明确的sandbox凭证传递机制** | 🔒 闭源 | ✅ Agent loop: bashPermissions在Agent loop内检查，bwrap提供隔离执行但凭证在Agent侧 | ✅ Sandbox内: Landlock LSM内核级沙箱(`core/src/lib.rs:47-48`)，凭证在沙箱进程内使用，受Landlock文件系统+网络双重控制 | ✅ Agent loop: Effect-TS fiber上下文，Server端FenceMiddleware做SSRF防护 |
| Agent隔离 | 同实例多Agent配置+资源per-Agent隔离 | ✅ 实现: `agents.list[]`数组: 每Agent独立skills/tools/model/thinking。配置: per-agent `skills` allowlist(替换非合并)+`skillsLimits`。效果: **full per-agent资源隔离** | ✅ 实现: SubGraphAgent三种形态(声明式/预编译/远程)+per-subagent middleware继承。效果: 三种形态均支持独立配置 | ✅ 实现: `subagents.custom_agents`独立system_prompt/tools/model+per-agent timeout/max_turns。效果: per-agent资源隔离 | ⚠️ delegate_task支持子Agent+独立工具集+budget。**但无agents.list[]多Agent配置语法** | 🔒 闭源(Cloud Agent支持customSubagents) | ✅ 实现: 6 built-in+Coordinator+Swarm三套并行+per-agent独立worktree。配置: `.claude/agents/*.md`声明式。效果: **已知产品最丰富Multi-Agent隔离体系** | ✅ 实现: `CodexSpawnArgs{config,auth,model,env,skills,mcp}`+agent-graph-store图拓扑。效果: per-agent全协议配置+图状隔离 | ✅ 实现: Agent mode: subagent/primary/all+per-Agent permission+per-Agent model。配置: `Agent.Info{mode, permission, model}` Effect Schema。效果: 三级角色+per-Agent资源 |
| 审计日志 | Agent操作完整记录 | - | - | ⚠️ 实现: 每日文件夹+JSONL格式记录。效果: 按日归档，非实时审计 | ✅ 实现: `hermes_logging.py`—3个RotatingFileHandler(agent.log/errors.log/gateway.log)+RedactingFormatter+session_id上下文。配置: `logging.level`/`max_size_mb`/`backup_count`/`security.redact_secrets`。效果: 完整操作日志，30+种secret脱敏，session关联 | ❓ 闭源 | ✅ 实现: sessionStorage完整持久化(含transcript+操作记录)。效果: 所有对话+操作完整记录，可重放 | ❓ | - |

| **得分** | | **2** | **2** | **3** | **2** | **1** | **4** | **4** | **3** |
|---------|------|-------|-------|--------|--------|---------|---------|-------|--------|

#### D5: 持久化与记忆系统

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Session 持久化 | 对话不因重启丢失 | ✅ 实现: Workspace文件持久化。效果: 会话按workspace组织，文件化存储 | ✅ 实现: LangGraph checkpointer。效果: Graph状态检查点，断点续跑 | ✅ 实现: SQLite/PostgreSQL(可配置)。配置: `config.yaml`→`persistence.backend`。效果: 数据库持久化，支持切换 | ✅ 实现: SQLite FTS5。配置: `~/.hermes/data/`自动创建。效果: 全文搜索+持久化，session关联 | ✅ 实现: SQLite。效果: 基础持久化 | ✅ 实现: sessionStorage+transcript完整持久化。配置: `.claude/projects/<name>/`目录。效果: 完整对话存储，可重放+重续 | ✅ 实现: agent-graph-store(SQLite)。效果: Agent关系图持久化，parent/child拓扑 | ✅ 实现: SQLite(Drizzle ORM)。配置: 自动迁移(`drizzle-kit`)。效果: 类型安全持久化，session/message/part/todo 4表 |
| 短期记忆 | 会话内上下文保持(HITL) | ⚠️ 仅手动`/compact`指令，无自动会话内压缩。Vector Memory(sqlite-vec)提供跨会话语义搜索但非会话内短期记忆 | ✅ 实现: SummarizationMW: trigger=('fraction',0.85), keep=('fraction',0.10), model='gpt-4o-mini'。效果: 85%阈值自动触发摘要，旧消息卸出到`/conversation_history/{thread_id}.md` | ✅ 实现: LLM driven Memory: debounced queue(30s)→事实提取→去重→atomic write。配置: `memory.enabled: true`, `fact_confidence_threshold: 0.7`, `max_injection_tokens: 2000`。效果: 30秒防抖异步记忆更新 | ✅ 实现: compressor自动触发(50%阈值, 目标压缩到20%): `trajectory_compressor.py`(65KB独立模块)。配置: `compression.threshold: 0.50`, `compression.target_ratio: 0.20`。效果: 上下文半自动压缩+FTS5搜索历史 | 🔒 闭源 | ✅ 实现: **唯一区分短期/长期记忆的产品**。五层分层: sessionStorage→memdir(MEMORY.md 200行/25KB截断)→SessionMemory→compact(autoCompactThreshold=contextWindow-13000)→hooks。配置: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`。效果: 会话内分层记忆，自动压缩 | ✅ 实现: compact_remote.rs(354行)v1+compact_remote_v2.rs v2+context_manager(TokenUsageBreakdown)。效果: 远程压缩v1+v2双策略+精确Token分解 | ✅ 实现: session/compaction.ts(652行): isOverflow+SUMMARY_TEMPLATE(Constraints/Preferences/Critical Context)+prune(PRUNE_PROTECT=40000)。效果: tail_turns(默认2)保留原文，旧消息摘要+工具输出超40K删除 |
| 长期记忆 | 跨会话用户偏好/事实/规范 | ✅ 实现: AGENTS.md/SOUL.md文件注入+sqlite-vec向量数据库。配置: workspace根目录放AGENTS.md/SOUL.md。效果: 声明式记忆+本地语义搜索+per-agent workspace文件隔离 | ✅ 实现: AGENTS.md文件解析。配置: 项目根目录放AGENTS.md。效果: 项目级上下文自动注入 | ✅ 实现: MemoryMiddleware(MW栈第12层)+per-user_id路径隔离。配置: `memory.enabled=true`。效果: 自动记忆管理+用户级数据分离 | ✅ 实现: Honcho/mem0双后端+Mem0 user_id filter+AGENTS.md/SOUL.md双文件。配置: `config.yaml`→`memory.provider`(honcho/mem0)。效果: 自适应记忆+多用户隔离+规范/人设分离+TOOLS.md工具偏好 | - | ✅ 实现: memdir+SessionMemory分层记忆(5层)+CLAUDE.md。配置: `.claude/memdir/`+`CLAUDE.md`。效果: 已知产品最完整分层长期记忆: sessionStorage→memdir→SessionMemory→compact→hooks | ❌ 确认无跨session记忆。仅agents_md.rs项目级+agent-graph-store拓扑 | - ⟳ 源码确认无跨会话记忆，仅会话内compaction |
| 长期记忆搜索 | 语义/全文搜索历史记忆 | ✅ 实现: sqlite-vec本地向量数据库。效果: 本地语义检索，不依赖外部向量DB | - | ✅ 实现: `POST /api/threads/search`—metadata精确匹配+status过滤+分页。效果: 按updated_at降序，user隔离。**无FTS全文** | ✅ 实现: FTS5全文搜索+LLM摘要生成(`session_search` tool)+LLM驱动语义搜索(`memory` tool)。配置: `config.yaml`→`memory.fts5`。效果: 全文+语义双重搜索 | 🔒 闭源 | ✅ 实现: 三重搜索: `/resume`(Fuse.js模糊,tag/branch/worktree过滤,阈值0.3)+agenticSessionSearch(AI语义,100会话×2000字符)+transcriptSearch(WeakMap缓存实时)。效果: 模糊+语义+实时三重 | ✅ 实现: 四重搜索: session_index.jsonl(append-only反向扫描)+SQLite instr()子串+TUI Picker(分页25条)+Ctrl+R反向。效果: 多层搜索但**无FTS5** | - ⟳ |
| Skills 自动管理 | Agent自动创建/改进Skills | - | - | - | ✅ 实现: Curator全生命周期引擎(`curator.py:255`)。配置: `curator.enabled=true`, `interval_hours=168`(7天), `stale_after_days=30`, `archive_after_days=90`。效果: active→stale→archived三态流转+LLM review(umbrella consolidation)+tar.gz快照+90+威胁模式+pinned防护。**永远只archive不delete** | - | - | - | - |

| **得分** | | **2** | **3** | **2** | **4** | **1** | **4** | **3** | **2** |
|---------|------|-------|-------|--------|--------|---------|---------|-------|--------|

#### D6: 扩展性与生态开放性

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Skills 系统 | Markdown/YAML声明式定义Agent能力 | ✅ 实现: SKILL.md+ClawHub社区Hub。配置: per-agent `agents.list[].skills` allowlist(完全替换defaults,不合并)。加载优先级: workspace/.agents/~/.openclaw/bundled/extraDirs 五级。效果: 非代码即可扩展，社区可分享，per-agent独立allowlist+skillsLimits.maxSkillsPromptChars预算 | ✅ 实现: SKILL.md+Discovery自动扫描。效果: 声明式Skills | ✅ 实现: SKILL.md+配置文件。效果: 声明式Skills | ✅ 实现: SKILL.md+agentskills.io Hub+Curator自动管理。配置: YAML frontmatter(name≤64chars, description≤1024chars)+progressive disclosure架构(metadata→instructions→linked files)。效果: 社区Hub+自动优化，全生命周期管理 | - | ✅ 实现: Skills系统(推断: SKILL.md兼容)。效果: 声明式Skills | - | ✅ 实现: SKILL.md三级发现(`.claude/skills/`+`.agents/skills/`+`{skill,skills}/`)。效果: 三目录兼容Claude Code+agents.md标准 |
| Plugin SDK | 深度扩展Agent子系统 | ✅ 实现: 80+ plugin-sdk运行时导出(channel/auth/reply/sandbox/TTS)。效果: 几乎无不可扩展的子系统 | ⚠️ 实现: Middleware API(13层可插拔)。效果: 中间件级扩展，非独立Plugin SDK | - | ✅ 实现: plugins/目录+Plugin注册机制。效果: 插件式扩展 | - | ✅ 实现: Plugin系统。效果: 推断: 插件架构 | - | ✅ 实现: Plugin SDK(Hooks系统,2767行)+Copilot/Gitlab/Poe/Cloudflare/Azure/Codex认证插件。配置: `PluginInput`+`Hooks`接口。效果: 多认证源+自定义Hook，`plugin/loader.ts`动态加载 |
| MCP Client | 消费外部MCP Server工具 | ✅ 实现: MCP client+auto-discovery。效果: 即插即用数百个MCP Server | ✅ 实现: langchain-mcp-adapters桥接。效果: MCP工具调用 | ✅ 实现: MCP client。效果: MCP工具消费 | ✅ 实现: MCP client+mcp_serve.py(Server端)。效果: 双向MCP | ✅ 实现: MCP集成。效果: MCP工具调用(stdio/HTTP/SSE) | ✅ 实现: MCP集成。效果: MCP工具调用 | ✅ 实现: codex-mcp crate(自研Rust native,4666行)+三级工具审批(MCP_TOOL_APPROVAL_ACCEPT/ACCEPT_FOR_SESSION/DECLINE_SYNTHETIC)。效果: 原生MCP实现+精细会话级权限 | ✅ 实现: MCP模块(1521行)。效果: 自研MCP client |
| MCP Server | 暴露自身工具给其他Agent | ❓ | - | - | ✅ 实现: mcp_serve.py。效果: 可被其他MCP Client调用 | ❓ 闭源 | ✅ 实现: Bridge模式。效果: 通过Bridge暴露工具 | ❓ | ❓ |
| ACP | 跨Agent互操作协议 | ✅ 实现: ACP client+server。效果: 双向ACP | ✅ 实现: ACP transport。效果: 远程SubAgent通信 | ✅ 实现: ACP Server+Client+`invoke_acp_agent`。效果: 双向ACP，委派编码任务 | ✅ 实现: ACP server+delegate_task ACP transport。效果: 双向ACP | - | ❌ 确认不使用MCP ACP。使用自有Bridge/Remote Control协议(CCR v2)。配置: `CLAUDE_CODE_USE_CCR_V2=1`+`/bridge`命令。效果: OAuth→JWT→SSE(读)+CCRClient(写)自有远程协议 | ❓ | ✅ 实现: @agentclientprotocol/sdk 0.16.1+acp/(1982行)。效果: 标准ACP client/server，`acp` CLI命令 |
| Skills Hub | 社区技能市场 | ✅ 实现: ClawHub。效果: 社区发布/安装Skills | - | - | ✅ 实现: agentskills.io。效果: 社区Skills市场 | - | - | - | - |
| Channel 扩展API | 社区开发新消息渠道 | ✅ 实现: plugin channel API。效果: 任何人可开发新IM适配器 | - | ✅ 实现: Channel基类。效果: 可继承扩展新渠道 | - | - | - | - | ✅ 实现: Slack集成+Plugin SDK。效果: 可扩展IM适配器 |
| Provider 数量 | 模型供应商选择自由度 | ✅ 实现: 20+ providers。效果: 不锁定单一供应商 | ✅ 实现: 20+ providers(LangChain)。效果: 多模型支持 | ✅ 实现: 10+ providers(注释模板)。效果: 多供应商可选 | ✅ 实现: 200+ providers(OpenRouter)。效果: 已知产品最多Provider | ❓ 闭源 | ❌ 仅Anthropic API。效果: 单供应商锁定 | ❌ 仅OpenAI API。效果: 单供应商锁定 | ✅ 实现: 15+ Vercel AI SDK原生providers(OpenAI/Anthropic/Google/Bedrock/Groq/Mistral/Cohere/xAI/Perplexity/DeepInfra/Alibaba/Azure/Cerebras/Gateway/OpenAI-compatible)。效果: 已知产品最多原生Provider集成 |

| **得分** | | **4** | **3** | **3** | **4** | **2** | **4** | **2** | **4** |
|---------|------|-------|-------|--------|--------|---------|---------|-------|--------|

---

## B.2.3 得分汇总与排名

| 排名 | 产品 | 平均分 | D1 | D2 | D3 | D4 | D5 | D6 | 核心判语 |
|:----:|------|:------:|:--:|:--:|:--:|:--:|:--:|:--:|---------|
| **1** | **Claude Code** | **3.67** | 2 | 4 | 4★ | 4 | 4 | 4 | Multi-Agent 深度无出其右，4层沙箱+5层记忆，但单 vendor |
| **1** | **Hermes Agent** | **3.67** | 4 | 4 | 4 | 2 | 4 | 4 | 最全面产品：200+ Provider + 自我完善闭环 + 18+ 渠道 |
| 3 | OpenClaw | 3.17 | 4 | 3 | 4 | 2 | 2 | 4 | 24+ 渠道 + 消费级完工程度 + 368k 社区，但 D5 偏弱 |
| 4 | DeerFlow 2.0 | 3.00 | 3 | 4 | 3 | 3 | 2 | 3 | 18层MW + 四层沙箱 + K3s 多租户雏形，权衡产品 |
| 4 | OpenCode | 3.00 | 3 | 3 | 3 | 3 | 2 | 4 | 全栈最均匀：Web+Desktop+Slack + Effect-TS 函数式架构 |
| 6 | Codex | 2.67 | 1 | 3 | 3 | 4 | 3 | 2 | Landlock 内核级沙箱+Guardian 审批，安全最强但单 vendor |
| 7 | Deep Agents | 2.50 | 1 | 2 | 4 | 2 | 3 | 3 | 13层MW + 三种 SubAgent + 108 evals，但无独立运行时 |
| 8 | Cursor SDK | 1.33 | 0 | 2 | 2 | 1 | 1 | 2 | IDE 生态内强大，闭源不可审计，无独立通信层 |

> ★ Claude Code D3 实际能力超越 4 分体系（三套 Multi-Agent 并存），评分上限为 4。

---

## 数据来源索引

本报告所有维度得分基于以下 Phase 1 垂直能力解构报告 (P0 源码级)：

| 产品 | 报告文件 | 分析方法 | 源码支撑率 |
|------|---------|---------|:---------:|
| OpenClaw | `openclaw-vertical-capability-report.md` | 源码 + CHANGELOG + docs 仓库 | ~90% |
| Deep Agents | `deepagents-vertical-capability-report.md` | 源码直读 (Python) | ~90% |
| DeerFlow 2.0 | `deerflow-2.0-vertical-capability-report.md` | 源码直读 (Python) | ~90% |
| Hermes Agent | `hermes-vertical-capability-report.md` | 源码直读 (Python) | ~90% |
| Cursor SDK | `cursor-sdk-vertical-capability-report.md` | npm tarball 黑盒 | ~40% |
| Claude Code | `claude-code-vertical-capability-report.md` | 泄露源码直读 (TypeScript) | ~80% |
| Codex | `codex-vertical-capability-report.md` | Rust 80+ crates 直读 | ~85% |
| OpenCode | `opencode-vertical-capability-report.md` | Effect-TS monorepo 直读 | ~90% |

---

## B.2.4 特征举证索引 (Evidence Cross-Reference)

> 每条特征在产品垂直报告中的行号引用。`↗` = 有证据，`⟳` = 待收集（垂直报告缺少该特征的实现与效果说明，需后续补全）。
> 引用格式: `垂直报告简称 §章节 L起始行号`

| 维度 | 特征 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| D3 | SubAgent 委派 | ↗ §2.1 L348 | ↗ §2.3 L119 | ↗ §3.2 L381 | ↗ §2.3 L196 | ↗ §4.1 L100 闭源 | ↗ §2.3 L167 | ↗ §2.1 L89 | ↗ §2.1 L105 |
| D3 | Loop 检测 | ↗ §2.1 L246 | ↗ §2.1 L144 | ↗ §3.1 L383 | ↗ §2.1 L109 | ⟳ 闭源 | ↗ autoCompact.ts:70 (短路器MAX=3) | ⟳ 未确认 | ↗ processor.ts:28 (DOOM_LOOP=3) |
| D3 | 上下文压缩 | ↗ §2.3 L148 | ↗ §2.1 L132 + 配置: trigger=0.85 | ↗ summarization_middleware.py (enabled=false 默认) | ↗ §2.1 L211 | ⟳ 闭源 | ↗ §2.5 L182 | ↗ §2.4 L65 | ↗ session/compaction.ts:652行 |
| D3 | 并行执行 | ↗ §2.3 L240 | ↗ §2.3 L119 | ↗ §3.2 L381 | ↗ §2.3 L203 | ⟳ 闭源 | ↗ §2.3 L228 | ↗ codex_delegate.rs:130,144 | ⟳ 不支持 |
| D4 | Sandbox 隔离 | ↗ §2.1 L217 | ↗ §2.1 L139 | ↗ §3.1 L369 | ↗ §2.1 L186 | ⟳ 闭源 | ↗ §2.2 L178 | ↗ §2.2 L99 | ↗ §2.2 L115 |
| D4 | 工具白名单 | ↗ §2.1 L219 | ⟳ 仅文件权限 | ↗ §3.1 L371 | ↗ §2.1 L208 | ⟳ 闭源 | ↗ §2.2 L196 | ↗ §2.1 L92 | ↗ §2.3 L130 |
| D4 | SSRF 防护 | ↗ §2.1 L219 | ⟳ 未支持 | ⟳ 未支持 | ↗ §2.1 L238 | ⟳ 闭源 | ↗ §2.2 L251 | ↗ §2.4 L97 | ↗ §2.3 L130 |
| D5 | Memory 系统 | ↗ §2.3 L229 | ↗ §2.1 L185 | ↗ §3.1 L415 | ↗ §2.1 L247 | ⟳ 闭源 | ↗ §2.5 L158 | ↗ §2.5 L65 | ⟳ |
| D5 | Session 持久化 | ↗ §2.3 L230 | ↗ §2.3 L344 | ↗ §3.1 L418 | ↗ §2.1 L246 | ↗ §4.1 L285 | ↗ §2.3 L166 | ↗ §2.3 L90 | ↗ §2.3 L108 |
| D5 | SessionSearch (新增) | ⟳ 待收集 | ⟳ 待收集 | ↗ threads.py:289 (metadata过滤) | ↗ §2.1 L187 (FTS5) | ⟳ 闭源 | ↗ transcriptSearch.ts + agenticSessionSearch.ts | ↗ session_index.rs (JSONL+instr) | ⟳ 不支持 |
| D1 | 流式输出 (新增) | ↗ draft-stream-loop.ts (throttle循环) | ⟳ TUI流式 | ↗ 飞书流式卡片 | ↗ KawaiiSpinner | ↗ SSE | ↗ Ink TUI | ↗ sse.rs → stream-parser 5层管道 | ⟳ TUI+Desktop |
| D1 | Voice (新增) | ↗ voicewake.ts + talk.ts (TTS+唤醒词) | ⟳ 不支持 | ⟳ 不支持 | ↗ TTS 5 providers | ⟳ 不支持 | ⟳ 不支持 | ⟳ 不支持 | ⟳ 不支持 |
| D6 | MCP 支持 | ↗ §2.1 L256 | ↗ §2.1 L133 | ↗ §3.1 L203 | ↗ §2.1 L225 | ↗ §4.1 L287 | ↗ §2.4 L72 | ↗ §2.6 L136 | ↗ §2.5 L152 |
| D6 | ACP 支持 | ↗ §2.1 L257 | ↗ §2.1 L163 | ↗ §3.1 L372 | ↗ §2.1 L224 | ⟳ 未支持 | ↗ §2.4 L227 | ↗ §2.1 L89 | ↗ §2.4 L143 |
| D6 | Provider 数量 | ↗ §2.1 L258 | ↗ §2.1 L162 | ↗ §3.1 L176 | ↗ §2.1 L226 | ⟳ 闭源 | ↗ §1.3 L27 | ↗ §1.1 L32 | ↗ §2.5 L150 |

**统计**: 新增3行(流式/Voice/SessionSearch)后共 112 格。v2.3 批量源码补充: ClaudeCode Loop✅, DeerFlow压缩✅+SessionSearch✅, OpenClaw流式✅+Voice✅+Plan✅+Skills✅, Codex流式✅+Plan✅, OpenCode Loop✅, Hermes D4/D5✅。覆盖率大幅提升。

### 待收集清单 (需后续特征挖掘)

| 产品 | 待补全特征 | 缺失原因 |
|------|----------|---------|
| OpenClaw | D5-SessionSearch | 待源码挖掘 |
| Deep Agents | D1-流式/Voice/SessionSearch, D2-并行执行, D4-工具白名单 | 执行环境虚拟化，部分不适用 |
| DeerFlow 2.0 | D4-SSRF/Vault, D1-Voice | 部分确认不支持，非缺失 |
| Hermes Agent | D4-Vault加密(明文存储), D1-Voice配置 | 凭证系统存在但无加密，Voice有TTS但缺配置细节 |
| Cursor SDK | D1-D5 大部分 | 闭源不可审计，仅编译产物 |
| Claude Code | D6-ACP(自有Bridge协议非标准ACP) | 支持自有远程协议，非MCP ACP |
| Codex | D1-Voice | Rust/TUI为主，语音非设计目标 |
| OpenCode | D5-Memory系统(D3压缩已✅) | 源码确认无跨会话记忆，仅会话内压缩 |
