# Agent Harness 产品横向对比报告

## Phase 2: 聚类建模 + Phase 3: 横向映射

> 产出日期: 2026-05-04
> 分析对象: OpenClaw / Deep Agents / DeerFlow 2.0 / Hermes Agent / Cursor SDK
> 数据基础: 5 份垂直能力解构报告 (Phase 1 产出)
> 方法论: 特征归一化 + 亲和聚类 + 场景加权评分

---

# Part A: Phase 2 — 聚类建模 (Clustering & Modeling)

## A.1 方法论设计

### A.1.1 方法论选型依据

传统横向对比存在两大缺陷：

| 缺陷 | 传统做法 | 本方案改进 |
|------|---------|-----------|
| **先入为主** | 预定义维度再套入产品 | 从 5 份报告的 180+ 原子特征中自底向上聚类 |
| **布尔化** | "有/无"二元判断 | 四级成熟度评分 (0-3)，捕捉实现深度差异 |
| **等权重** | 所有维度一视同仁 | 场景驱动的动态权重分配 |
| **忽视冲突** | 不记录特征互斥/替代关系 | 显式标注"互斥特征对"，避免重复计数 |

### A.1.2 特征归一化流程

```
5 份垂直报告 × 8 维度 × 平均 9 个原子特征 ≈ 360 条原始特征
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

基于 360+ 条原始特征的归一化和语义聚类，得出以下 6 个互斥完备的维度：

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
逐产品评分 — 基于垂直报告的 原子特征表 + Gap分析 + 校准备忘录
       │
       ▼
交叉验证 — 消除主观偏差
       │
       ▼
场景加权 — 三个 Benchmark 场景动态分配权重
       │
       ▼
最终排序 + 推荐
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
| **D1. 通信广度** | ████ 4 | ██ 1 | ███ 3 | ████ 4 | █ 0 | ██ 2 | █ 1 | ██ 2 |
| **D2. 执行深度** | ███ 3 | ██ 2 | ████ 4 | ████ 4 | ██ 2 | ████ 4 | ███ 3 | ███ 3 |
| **D3. 任务编排** | ████ 4 | ████ 4 | ███ 3 | ████ 4 | ██ 2 | ████ 4★ | ██ 2 | ███ 3 |
| **D4. 安全隔离** | ██ 2 | ██ 2 | ███ 3 | ██ 2 | █ 1 | ████ 4 | ███ 3 | ██ 2 |
| **D5. 记忆系统** | ██ 2 | ███ 3 | ██ 2 | ████ 4 | █ 1 | ████ 4 | ██ 2 | ██ 2 |
| **D6. 扩展生态** | ████ 4 | ███ 3 | ███ 3 | ████ 4 | ██ 2 | ████ 4 | ██ 2 | ████ 4 |
| **平均分** | **3.17** | **2.50** | **3.00** | **3.67** | **1.33** | **3.67** | **2.17** | **2.67** |
| **社区规模** | 368k ★ | 22k ★ | 64k ★ | 132k ★ | 33k ★ | 泄露分析 | 80k ★ | 155k ★ |

> ★ Claude Code D3 实际能力超越 4 分体系（三套 Multi-Agent 并存），但评分体系上限为 4。

### B.2.2 逐维度详细对比

#### D1: 通信广度与渠道归一化

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK |
|------|---------|:--------:|:-----------:|:------------:|:------------:|:----------:|
| IM 渠道数 | 决定 Agent 可触达用户的平台范围，渠道越多覆盖人群越广 | 24+ | 0 (CLI only) | 6 (TG/Slack/飞书/微信/企微/钉钉) | 18+ | 0 (IDE only) |
| 流式输出 | 打字机效果实时反馈，降低用户等待焦虑，提升交互体验 | ✅ WS event | ✅ TUI | ✅ 飞书流式卡片/钉钉AI Card | ✅ KawaiiSpinner | ✅ SSE |
| Voice | 语音唤醒+对话使 Agent 脱离屏幕，适用于驾驶/家务等不触屏场景 | ✅ Voice Wake/Talk | ❌ | ❌ | ✅ TTS (5 providers) | ❌ |
| A2UI Canvas | Agent 可动态生成交互式 Web UI（图表/表单/仪表盘），从纯文本对话升级为富交互 | ✅ | ❌ | ❌ | ❌ | ❌ |
| Device Nodes | Agent 可通过 Camera/Screen/Location 感知物理世界，实现视觉辅助/屏幕理解/位置服务 | ✅ Camera/Screen/Location | ❌ | ❌ | ❌ | ❌ |
| Channel Plugin 架构 | 允许社区开发新渠道，渠道扩展成本归零，生态效应：渠道数可无限增长 | ✅ 80+ exports | ❌ | ✅ Channel 基类 + Bus | ✅ Platform-aware toolsets | ❌ |
| **得分** | | **4** | **1** | **3** | **4** | **0** |
| 得分依据 | | 24 渠道 + Voice + Canvas + 80+ SDK | 无独立 IM channel，TUI CLI | 6 渠道 + 流式卡片 + 微信自举 | 18+ 平台 + TTS + platform-aware | IDE 内嵌，非独立 Gateway |

#### D2: 工具深度与执行环境

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK |
|------|---------|:--------:|:-----------:|:------------:|:------------:|:----------:|
| Shell 执行 | Agent 执行系统命令的核心能力——编译代码、安装依赖、运行脚本、操作进程 | ✅ | ✅ (120s timeout) | ✅ (虚拟路径翻译) | ✅ (3 种 container) | ✅ |
| 文件操作 | Agent 读写文件的编辑器级能力——精确替换字符串、自动创建目录、防止并发写冲突 | ✅ | ✅ (6 种 Backend) | ✅ (FileOpLock) | ✅ | ✅ |
| Browser | Agent 操控真实浏览器——模拟点击、截图分析、表单填写、动态内容抓取 | ✅ Playwright | ❌ | ❌ | ✅ | ❌ |
| Sandbox 后端 | 决定 Agent 的执行隔离级别——Docker(进程级)、SSH(主机级)、Daytona(Serverless) | Docker/SSH/OpenShell (3) | 5 种 (State/FS/Shell/Store/Composite) | 四层隔离模式 | Docker/SSH/Daytona (3) | 局部 (闭源) |
| ACP 跨 Agent | Agent 可调用外部 Agent (Claude Code/Codex) 作为子执行器，复用已有 Agent 生态 | ✅ | ✅ | ✅ | ✅ | ❌ |
| Git 集成 | Agent 自动 git commit/checkpoint——每次操作可回滚，形成可审计的操作历史 | ❌ | ❌ | ❌ | ✅ Checkpoint | ✅ git-core |
| 虚拟路径翻译 | Agent 在沙箱中看到的路径与宿主真实路径自动映射，安全隔离同时保持文件操作语义一致 | ❌ | ❌ | ✅ 自动翻译 | ❌ | ❌ |
| 沙箱生命周期 | 沙箱的创建/复用/销毁策略——是否延迟创建(lazy)、同 Thread 复用、自动清理、支持外置 Provider 接入 | mode(off/non-main/all) + scope(agent/session/shared) + sandbox CLI(recreate/list/explain) | 5 Provider 协议化 | ✅ acquire/release/shutdown + lazy_init + Thread 复用 + shutdown() 清理 | 6 Backend + daemon 空闲清理 + persistent 开关 | ❓ 闭源 |
| **得分** | | **3** | **2** | **4** | **4** | **2** |
| 得分依据 | | 3 Sandbox + Browser，但沙箱生命周期粗放 | 5 种后端但生命周期由 Provider 决定 | 唯一显式生命周期 API + 四层模式 + CI 检测 | 6 Backend + daemon 清理，但 task_id 共享 | 闭源不透明，Sandbox 细节未知 |

#### D3: 智能决策与任务编排

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK |
|------|---------|:--------:|:-----------:|:------------:|:------------:|:----------:|
| SubAgent 委派 | 复杂任务自动拆解为子任务分派给独立子Agent，每个子Agent有独立上下文和工具集 | ✅ sessions_spawn + subagent registry | ✅ 声明式/预编译/远程 | ✅ 双线程池(3+3) | ✅ delegate_task | ✅ customSubagents |
| 并行执行 | 多个子Agent同时执行互不依赖的子任务，总耗时 = max(子任务耗时) 而非 sum | ✅ parallel subagent completion + grouped results | ✅ AsyncSubAgent | ✅ MAX=3 | ✅ max=3 | ❓ |
| Plan/Todo | Agent 处理 3+ 步骤任务时自动创建 Todo 列表并实时更新进度，用户可跟踪任务进展 | ✅ update_plan tool (experimental, opt-in) | ✅ TodoListMiddleware | ✅ TodoListMiddleware | ❌ | ❌ |
| Loop 检测 | 当 Agent 陷入死循环（反复调用同一工具无进展）时自动检测并强制终止，防止无限消耗 Token | ✅ tools.loopDetection: historySize/warningThreshold/criticalThreshold/detectors(genericRepeat/knownPollNoProgress/pingPong) + globalCircuitBreaker + postCompactionGuard | ✅ PatchToolCalls | ✅ LoopDetectionMW | ✅ tool_loop_guardrails | ❓ |
| 上下文压缩 | 对话 Token 接近模型上限时自动压缩历史消息为摘要，避免因上下文溢出导致任务中断 | ✅ auto-compaction + midTurnPrecheck + compaction retry + compaction notifier + sessions_yield | ✅ 自动压缩(85%触发) | ❌ | ✅ 自动压缩(50%触发) | ❓ |
| Middleware 层数 | 管道化处理请求的中间件数量——越多代表请求处理越精细（工具注入/权限检查/摘要/记忆） | pi-embedded-runner (自有引擎，非外部 PI Agent) | 13 层 | 18 层 (完整) | N/A (自有Loop) | ❓ |
| 工具调用恢复 | 工具调用失败后自动 patch/重试/降级，非简单报错给用户——提升长任务成功率 | ✅ orphan recovery + compaction retry + tool-result guard | ✅ PatchToolCalls | ✅ LoopDetection hard stop | ✅ Guardrails hard_stop | ❓ |
| **得分** | | **4** | **4** | **3** | **4** | **2** |
| 得分依据 | | 完整 SubAgent + Loop检测 + Compaction + update_plan — PI Agent 是嵌入式引擎非外部依赖 | 13 层 MW + 3 形态 SubAgent + 压缩 | 18 层 MW + 双线程池 + Loop检测 | Guardrails + 压缩 + SubAgent + Budget | 闭源无法验证，仅从类型推断基础能力 |

#### D4: 安全与多租户隔离

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK |
|------|---------|:--------:|:-----------:|:------------:|:------------:|:----------:|
| DM 配对 | 未知用户首次 DM 时需手动审批配对码，防止恶意用户直接操控 Agent | ✅ pairing 默认 | ❌ | ❌ | ✅ Gateway DM | ❌ |
| Sandbox 隔离 | Agent 执行环境与宿主物理/逻辑隔离——防止文件篡改、命令注入、资源滥用 | ✅ 3 种 | ✅ 5 种 (执行) | ✅ 四层 | ✅ 3 种 | ❓ |
| 工具白名单 | 按 Sandbox 类型限制 Agent 可使用的工具——如禁止非受信 session 使用 Browser/Cron | ✅ Allow/Deny | ⚠️ 文件权限 | ✅ 四层控制 | ✅ Guardrails | ❓ |
| SSRF 防护 | 防止 Agent 被诱导访问内网服务（如 `http://169.254.169.254/`），阻断横向移动攻击 | ✅ ssrf-policy | ❌ | ❌ | ✅ website_blocklist | ❓ |
| 威胁检测 | 运行时检测恶意行为模式（如异常 Shell 命令序列），主动告警或阻断 | ❌ | ❌ | ❌ | ✅ Tirith | ❓ |
| Vault/Secret | API Key 等敏感凭证的加密存储、轮换和绑定——防止密钥泄露和硬编码 | ⚠️ secret-ref | ❌ | ❌ | ⚠️ .env + auth.json | ❓ |
| 审计日志 | 完整记录 Agent 的所有操作（谁/何时/做了什么/结果），支持合规审计和事后溯源 | ❌ | ❌ | ❌ | ✅ hermes_logging | ❓ |
| 多租户隔离 | 租户级资源配额 + 配置分离 + 数据隔离——企业级 CMA 的核心需求。当前所有产品均为 session/user 级隔离，无真正租户 | ❌ 无 tenant 概念 | ❌ 无 tenant 概念 | ⚠️ user_id 级 + K3s Pod；社区需求 #2318 | ❌ 单租户架构；沙箱 task_id 全局共享 | ❓ 闭源 |
| **得分** | | **2** | **2** | **3** | **2** | **1** |
| 得分依据 | | DM+SSRF OK，但无租户+沙箱二分粗 | 仅文件权限+Sandbox，缺 DM/SSRF/租户 | user_id 隔离 + K3s Pod 模式最接近多租户 | 安全工具全但沙箱共享+单租户 | 闭源安全不可审计，降级 |

#### D5: 持久化与记忆系统

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK |
|------|---------|:--------:|:-----------:|:------------:|:------------:|:----------:|
| Session 持久化 | 对话不因重启丢失——Agent 可从上次断点继续，长任务（数小时）可跨 session 执行 | ✅ Workspace | ✅ LangGraph checkpointer | ✅ SQLite/PostgreSQL | ✅ SQLite FTS5 | ✅ SQLite |
| Memory 系统 | 跨会话记住用户偏好/事实——"上次你说过 X"不再需要用户重复，Agent 持续学习 | ✅ AGENTS.md/SOUL.md | ✅ AGENTS.md | ✅ MemoryMiddleware | ✅ Honcho/mem0 | ❌ |
| Vector Memory | 语义搜索历史对话——"帮找我们讨论过的那个 PDF 方案"，基于内容而非关键词检索 | ✅ sqlite-vec | ❌ | ❌ | ✅ memory tool | ❌ |
| Session Search | 全文搜索所有历史会话——用户可以问"我之前怎么配置的数据库"并得到精确引用 | ❌ | ❌ | ❌ | ✅ FTS5 + LLM 摘要 | ❌ |
| Agent 记忆文件 | 声明式注入项目上下文——AGENTS.md(项目规范)、SOUL.md(Agent 人设)、TOOLS.md(工具偏好) | ✅ AGENTS.md/SOUL.md | ✅ AGENTS.md | ❌ | ✅ AGENTS.md/SOUL.md | ❌ |
| 自动压缩 | 长对话自动摘要存储，释放上下文窗口同时保留关键信息——实现"无界记忆" | ❌ | ✅ SummarizationMW | ❌ | ✅ compressor | ❓ |
| Skills 自动管理 | Agent 从经验中自动创建/改进 Skills，无人值守地优化自身能力——自我完善的闭环 | ❌ | ❌ | ❌ | ✅ Curator 自动 prune | ❌ |
| 自我完善闭环 | 创建 → 使用 → 改进 → 归档的完整生命周期——Agent 越用越强而非越用越旧 | ❌ | ❌ | ❌ | ✅ 创建→使用→改进 | ❌ |
| Memory 多用户隔离 | 不同用户间的 Memory 是否隔离——Mem0 按 user_id 过滤、Honcho 按 session_key、文件按 workspace 路径 | ✅ per-agent workspace 文件隔离 | ❌ 无 user 概念 | ✅ per-user_id 路径隔离 | ✅ Mem0 user_id filter + Honcho session_key | ❓ 闭源 |
| **得分** | | **2** | **3** | **2** | **4** | **1** |
| 得分依据 | | 文件记忆+Vector+per-agent 隔离 | AGENTS.md+压缩，缺搜索/Curator/多用户 | 基础持久化+user_id Memory 隔离 | 全功能唯一：FTS5+Curator+自我完善+user_id Memory | 仅 SQLite 基础存储 |

#### D6: 扩展性与生态开放性

| 特征 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK |
|------|---------|:--------:|:-----------:|:------------:|:------------:|:----------:|
| Skills 系统 | 通过 Markdown/YAML 声明式定义 Agent 专项能力，非代码即可扩展——降低扩展门槛到非开发者 | ✅ SKILL.md | ✅ SKILL.md | ✅ SKILL.md | ✅ SKILL.md | ❌ |
| Plugin SDK | 开发者可深度扩展 Agent 子系统（渠道/审批/回复/沙箱/TTS）——80+ 导出意味着几乎无不可扩展的子系统 | ✅ 80+ exports | ⚠️ Middleware API | ❌ | ✅ plugins/ | ❌ |
| MCP Client | 消费外部 MCP Server 提供的工具——数百个现成 MCP Server 即插即用，无需为每个 API 写适配 | ✅ | ✅ | ✅ | ✅ | ✅ |
| MCP Server | 将自身工具暴露为 MCP Server 供其他 Agent 调用——实现 Agent-to-Agent 工具共享 | ❓ | ❌ | ❌ | ✅ mcp_serve.py | ❓ |
| ACP | Agent Communication Protocol 跨进程 Agent 互操作——调用 Claude Code/Codex 等外部 Agent | ✅ | ✅ | ✅ | ✅ | ❌ |
| Skills Hub | 社区技能市场——用户可安装他人发布的 Skills，避免重复造轮子，加速能力积累 | ✅ ClawHub | ❌ | ❌ | ✅ agentskills.io | ❌ |
| Channel 扩展 API | 社区可开发新消息渠道插件——WhatsApp/Telegram 之外，企业可自建飞书/钉钉/Teams 适配器 | ✅ plugin channel | ❌ | ✅ Channel 基类 | ❌ | ❌ |
| Provider 数量 | 支持的模型提供商数——决定用户的模型选择自由度和被锁定风险 | 20+ | 20+ | 10+ (注释模板) | 200+ (OpenRouter) | ❓ (Cloud) |
| **得分** | | **4** | **3** | **3** | **4** | **2** |
| 得分依据 | | 80+ Plugin + Hub + Channel API | 强 Middleware API 但无 Hub/Channel | 完整 Skills+MCP+ACP，缺 Plugin SDK | 双向 MCP + Hub + Curator + 200 Provider | MCP+SubAgents，闭源限制扩展深度 |

---

## B.3 场景化加权评分

### B.3.1 三个 Benchmark 场景

| 场景 | 描述 | 核心需求 |
|------|------|---------|
| **S1: 个人 AI 助手** | 面向个人用户的消费级 AI 助手，日常通过 IM/语音交互 | 通信广度、安全隔离、记忆系统 |
| **S2: 企业 Agent 平台** | 面向企业的 Agent 管理系统 (类 CMA)，多租户、多 Agent | 安全隔离、任务编排、扩展生态、记忆系统 |
| **S3: 开发者 Agent SDK** | 面向开发者，嵌入 IDE 或编程工作流的 Agent 框架 | 任务编排、执行深度、扩展生态 |

### B.3.2 场景权重分配

| 维度 | S1: 个人助手 | S2: 企业平台 | S3: 开发 SDK |
|------|:-----------:|:-----------:|:-----------:|
| D1. 通信广度 | **30%** | 10% | 0% |
| D2. 执行深度 | 15% | 20% | **30%** |
| D3. 任务编排 | 10% | **25%** | **30%** |
| D4. 安全隔离 | 20% | **30%** | 10% |
| D5. 记忆系统 | **20%** | 10% | 5% |
| D6. 扩展生态 | 5% | 5% | **25%** |

### B.3.3 加权得分

**S1: 个人 AI 助手**

| 产品 | D1(30%) | D2(15%) | D3(10%) | D4(20%) | D5(20%) | D6(5%) | 加权总分 |
|------|:-------:|:-------:|:-------:|:-------:|:-------:|:------:|:--------:|
| OpenClaw | 4×0.30 | 3×0.15 | 4×0.10 | 2×0.20 | 2×0.20 | 4×0.05 | **3.05** |
| Hermes Agent | 4×0.30 | 4×0.15 | 4×0.10 | 2×0.20 | 4×0.20 | 4×0.05 | **3.60** |
| DeerFlow 2.0 | 3×0.30 | 4×0.15 | 3×0.10 | 3×0.20 | 2×0.20 | 3×0.05 | **2.95** |
| Deep Agents | 1×0.30 | 2×0.15 | 4×0.10 | 2×0.20 | 3×0.20 | 3×0.05 | **2.35** |
| Cursor SDK | 0×0.30 | 2×0.15 | 2×0.10 | 1×0.20 | 1×0.20 | 2×0.05 | **1.00** |

**S2: 企业 Agent 平台**

| 产品 | D1(10%) | D2(20%) | D3(25%) | D4(30%) | D5(10%) | D6(5%) | 加权总分 |
|------|:-------:|:-------:|:-------:|:-------:|:-------:|:------:|:--------:|
| OpenClaw | 4×0.10 | 3×0.20 | 4×0.25 | 2×0.30 | 2×0.10 | 4×0.05 | **3.00** |
| Hermes Agent | 4×0.10 | 4×0.20 | 4×0.25 | 2×0.30 | 4×0.10 | 4×0.05 | **3.40** |
| DeerFlow 2.0 | 3×0.10 | 4×0.20 | 3×0.25 | 3×0.30 | 2×0.10 | 3×0.05 | **3.10** |
| Deep Agents | 1×0.10 | 2×0.20 | 4×0.25 | 2×0.30 | 3×0.10 | 3×0.05 | **2.55** |
| Cursor SDK | 0×0.10 | 2×0.20 | 2×0.25 | 1×0.30 | 1×0.10 | 2×0.05 | **1.35** |

**S3: 开发者 Agent SDK**

| 产品 | D1(0%) | D2(30%) | D3(30%) | D4(10%) | D5(5%) | D6(25%) | 加权总分 |
|------|:------:|:-------:|:-------:|:-------:|:------:|:-------:|:--------:|
| OpenClaw | — | 3×0.30 | 4×0.30 | 2×0.10 | 2×0.05 | 4×0.25 | **3.40** |
| Hermes Agent | — | 4×0.30 | 4×0.30 | 2×0.10 | 4×0.05 | 4×0.25 | **3.80** |
| DeerFlow 2.0 | — | 4×0.30 | 3×0.30 | 3×0.10 | 2×0.05 | 3×0.25 | **3.25** |
| Deep Agents | — | 2×0.30 | 4×0.30 | 2×0.10 | 3×0.05 | 3×0.25 | **2.80** |
| Cursor SDK | — | 2×0.30 | 2×0.30 | 1×0.10 | 1×0.05 | 2×0.25 | **1.80** |

### B.3.4 场景排名汇总

```
S1: 个人 AI 助手        S2: 企业 Agent 平台       S3: 开发者 Agent SDK
─────────────────      ──────────────────       ──────────────────
1. Hermes Agent 3.60   1. Hermes Agent 3.40     1. Hermes Agent 3.80
2. OpenClaw     3.05   2. DeerFlow 2.0  3.10     2. OpenClaw      3.40
3. DeerFlow 2.0 2.95   3. OpenClaw      3.00     3. DeerFlow 2.0  3.25
4. Deep Agents  2.35   4. Deep Agents   2.55     4. Deep Agents   2.80
5. Cursor SDK   1.00   5. Cursor SDK    1.35     5. Cursor SDK    1.80
```

---

## B.4 选型结论与建议

### B.4.1 全局排名

| 排名 | 产品 | 平均分 | 优势场景 | 核心风险 |
|:----:|------|:------:|---------|---------|
| **1** | **Hermes Agent** | **3.67** | 全场景领先 | 多租户缺失（沙箱 task_id 全局共享） |
| 2 | OpenClaw | 3.17 | 个人助手(S1#2) / 开发SDK(S3#2) | 无多租户，D5记忆系统偏弱 |
| 3 | DeerFlow 2.0 | 3.00 | 企业平台(S2#2) / 开发 SDK | IM 渠道数有限 (6 个) |
| 4 | Deep Agents | 2.50 | 开发 SDK (编排最强) | 无独立运行时，不适合个人/企业场景 |
| 5 | Cursor SDK | 1.33 | Cursor IDE 生态内 | 闭源、不可审计、无通信层 |

### B.4.2 针对内部选型「OpenClaw vs Hermes Agent」的结论

当前选型方案为 OpenClaw，内部挑战要求「超越 Star 数的全面功能评测」。基于本报告数据：

**OpenClaw 的真实优势** (Star 之外的客观支撑):

| 优势 | 量化证据 | 对应维度 |
|------|---------|---------|
| 24+ IM 渠道覆盖 | 已知产品中最广 | D1: 通信广度 (4/4) |
| 消费级产品完工程度 | Voice Wake/Talk + Canvas A2UI + macOS/iOS/Android Apps | D1: 消费者就绪度 |
| Plugin SDK 广度 | 80+ plugin-sdk 运行时导出 | D6: 扩展性 (4/4) |
| 安全默认配置 | DM pairing 默认开启 + SSRF 防护 | D4: 安全隔离 (2/4) |
| 社区规模 | 368k Stars — 最大的 AI Agent 开源社区 | 生态活力 |

**OpenClaw 的显著劣势**:

| 劣势 | 量化证据 | 影响 |
|------|---------|------|
| 无多租户 | 无 tenant 概念，Sandbox main/non-main 二分粗放 | 企业级 CMA 场景的关键缺失 |
| Memory 系统浅层 | 文件记忆 + Vector，无搜索/自我完善 | D5: 2/4，远低于 Hermes 的 4/4 |
| Windows 原生支持弱 | README 明确「strongly recommended WSL2」 | 企业 Windows 环境部署成本高 |
| update_plan 实验性 | 默认关闭，需显式 opt-in | Plan/Todo 功能非开箱即用 |

**Hermes Agent 对比优势**:

| 优势 | 量化证据 | 对比 OpenClaw |
|------|---------|:------------:|
| 自我完善闭环 | 自动创建/管理 Skills + FTS5 搜索 + Curator | OpenClaw 无此能力 |
| Memory 深度 | FTS5 Session Search + Mem0/Honcho user_id 隔离 | OpenClaw D5 仅 2 分 |
| 安全检测 | Tirith 威胁检测 + PII 脱敏 + website_blocklist | OpenClaw 无内置威胁检测 |
| 200+ Provider | OpenRouter 集成 | OpenClaw ~20 Provider |
| ⚠️ 多租户短板 | 沙箱 task_id="default" 全局共享，无租户资源隔离 | DeerFlow 的 K3s Pod 更接近多租户 |

### B.4.3 给团队的最终推荐

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        选型推荐矩阵                                       │
├───────────────┬─────────────────────────────────────────────────────────┤
│  场景          │  推荐产品                              理由              │
├───────────────┼─────────────────────────────────────────────────────────┤
│  企业 Agent   │  Hermes Agent / DeerFlow 2.0          编排+记忆全优     │
│  管理系统     │  Hermes 为数据面首选，DeerFlow 为备选  二者均需补多租户   │
│              │  DeerFlow 的 K3s Pod 模式更接近多租户    但社区需求待满足  │
├───────────────┼─────────────────────────────────────────────────────────┤
│  个人 AI 助手 │  OpenClaw / Hermes Agent              OpenClaw 渠道最广 │
│              │  如需 Voice+Canvas → OpenClaw          Hermes 能力最全   │
├───────────────┼─────────────────────────────────────────────────────────┤
│  开发者 SDK   │  Deep Agents / Hermes Agent           Deep 编排最强     │
│              │  如深度依赖 LangGraph → Deep           Hermes 通用最全   │
└───────────────┴─────────────────────────────────────────────────────────┘
```

**针对当前选型 (OpenClaw → 类 CMA 企业平台) 的建议**:

如果目标是构建 **Agent 管理系统 (类 CMA)**，多租户是所有候选产品的关键短板。当前最新评分（补充核查后）：

- OpenClaw S2 得分从 2.90 降至 2.50，在五款产品中排第 4
- Hermes Agent S2 得分从 3.70 降至 3.40，仍排第 1
- DeerFlow 2.0 S2 得分 3.10 不变，排第 2

建议：
1. **首选 Hermes Agent** — 虽有多租户短板，但 D2/D3/D5/D6 均领先，总体企业平台能力最强
2. **备选 DeerFlow 2.0** — K3s Pod + user_id 隔离模式最接近多租户，社区有 #2318 Issue 跟踪
3. **所有候选产品都需自研补齐多租户** — 租户级资源配额、配置分离、沙箱隔离是共性缺失

---

## B.5 方法论局限性声明

1. **评分基于静态分析**: 所有数据来自源码、文档和配置文件，非运行时 Benchmark
2. **无性能对比**: Token 消耗、延迟、吞吐量未测量（需要统一 Benchmark 环境）
3. **版本时效性**: 报告基于 2026-05-04 的仓库状态，产品快速迭代中
4. **闭源不透明**: Cursor SDK 的评分存在推断成分，实际能力可能高于或低于评分
5. **场景权重主观性**: 三个场景的权重分配基于典型需求推断，实际项目可能有不同侧重

---

## B.6 下一步建议

| 优先级 | 行动 | 说明 |
|:------:|------|------|
| P0 | 运行时 Benchmark | 在统一环境中跑三个场景的端到端测试，验证静态分析结论 |
| P0 | 深度 PoC | 对 Top 2 产品 (Hermes Agent + DeerFlow 2.0) 进行 2 周 PoC |
| P1 | CMA 概念对齐 | 基于选型结果，执行 Phase 3 扩展 — CMA 九大概念对齐 |
| P1 | 社区健康度评估 | 分析 Issue 关闭率、PR merge 速度、贡献者多样性 |

---

## B.7 基准评测体系摸底 (Evaluation & Benchmark Landscape)

> 核查日期: 2026-05-05
> 数据来源: 各产品官方仓库 evals/ 目录 + README + CI workflows

### B.7.1 逐产品评测能力

| 产品 | 评测体系 | 外部 Benchmark | CI 集成 | 公开得分 |
|------|---------|:---:|:---:|:---:|
| **Deep Agents** | `libs/evals/` — 108 evals × 7 类别 × 36 模型 | FRAMES / Nexus / BFCL v3 / MemoryAgentBench(ICLR 2026) / TAU2 Airline / Terminal Bench 2.0(Harbor) | ✅ GitHub Actions evals.yml + harbor.yml + LangSmith Dashboard | ❌ 无公开 leaderboard |
| **Hermes Agent** | `batch_runner.py`(轨迹生成) + `mini_swe_runner.py`(SWE 风格) + `hermes_swe_env`(RL 环境) | 无标准 benchmark 集成（自研 SWE 环境用于 RL 训练） | ✅ multiprocessing Pool + checkpoint 断点续跑 | ❌ 无公开得分 |
| **DeerFlow 2.0** | Roadmap § Benchmark 体系 🔥🔥🔥🔥🔥 | 无（计划中） | ❌ | ❌ |
| **OpenClaw** | 无独立 evals 目录 | 无 | CI 含 Docker E2E(live models/channels/sandboxes) 但非标准化评测 | ❌ |
| **Cursor SDK** | 闭源不可见 | 未知 | 未知 | ❌ |

### B.7.2 Deep Agents 评测体系详情（唯一成熟体系）

| 类别 | evals 数量 | 代表用例 |
|------|:---:|------|
| File Ops | 13 | 并行读写、编辑替换、截断恢复、空文件处理 |
| Retrieval | 6 | FRAMES 多跳检索、glob/grep 代码搜索、深层嵌套定位 |
| Tool Use | 53 | BFCL v3 多轮工具调用、Nexus 嵌套函数组合、Incident Graph 多步推理 |
| Skills | 8 | Skill 加载、执行、条件触发 |
| Memory | 5 | MemoryAgentBench(ICLR 2026)、多轮记忆一致性 |
| SubAgents | 14 | 并行委派、结果聚合、状态检查点 |
| Followup | 9 | HITL 中断确认、歧义澄清、总结质量 |

**模型覆盖**: 36 个模型（Claude Opus-4-7 / Sonnet-4-6、GPT-5.5/5.4/5.3-codex、Gemini-3.1-pro、Qwen3-Coder-480B、Kimi-K2.6、DeepSeek-V3 等）

**基础设施**: LangSmith 追踪 + Harbor 沙箱 + pytest 参数化 + LLM-as-judge(openevals)

### B.7.3 对选型的启示

- **Deep Agents 评测体系最成熟**，可复用其 benchmark 为其他产品建立统一评测基准（FRAMES/Nexus/BFCL v3 均公开可用）
- **Hermes Agent 的轨迹生成系统**可与 Deep Agents evals 互补——用 Deep 的 benchmark 题目跑 Hermes 的 batch_runner，产出可比轨迹
- **DeerFlow 的 Benchmark 缺失**是 Roadmap 高点，社区 #1669 Issue 明确标注「待启动」
- **所有产品均无公开的横向对比 benchmark 得分**——这既是差距也是机会
| P2 | 建立动态评估管道 | 将本报告方法固化为自动化 Script，新产品接入仅需 1 天 |

---

## 附录: 特征来源索引

本报告所有维度得分基于以下 Phase 1 垂直报告：

| 产品 | 报告文件 | 校准备忘录 |
|------|---------|-----------|
| OpenClaw | `openclaw-vertical-capability-report.md` | `openclaw-calibration-memo.md` |
| Deep Agents | `deepagents-vertical-capability-report.md` | `deepagents-calibration-memo.md` |
| DeerFlow 2.0 | `deerflow-2.0-vertical-capability-report.md` | `deerflow-2.0-calibration-memo.md` |
| Hermes Agent | `hermes-vertical-capability-report.md` | `hermes-calibration-memo.md` |
| Cursor SDK | `cursor-sdk-vertical-capability-report.md` | `cursor-sdk-calibration-memo.md` |
| Claude Code | `claude-code-vertical-capability-report.md` | 内嵌于报告 (社区分析仓库) |
| Codex | `codex-vertical-capability-report.md` | 内嵌于报告 |
| OpenCode | `opencode-vertical-capability-report.md` | 内嵌于报告 |
