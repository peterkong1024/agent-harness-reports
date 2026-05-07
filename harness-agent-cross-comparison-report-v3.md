# Agent Harness 产品横向对比报告 v3

> 产出日期: 2026-05-07
> 分析对象: OpenClaw / Deep Agents / DeerFlow 2.0 / Hermes Agent / Cursor SDK / Claude Code / Codex / OpenCode
> 数据基础: 8 份源码级垂直能力解构报告 (Phase 1 产出)
> 方法论: 基于 Harness 框架定义的新维度体系，等权重子特性计分，[wl] 标记特性半权重
> v3 变更: 去除成熟度评分，采用子特性计数法；7 大维度 37 子特性

---

## 评分规则

| 规则 | 说明 |
|------|------|
| **等权重** | 每个子特性 1 分（✅=1, ⚠️=0.5, ❌/🔒=0） |
| **[wl] 半权重** | 标记 [wl] 的特性仅 1/2 权重（✅=0.5, ⚠️=0.25） |
| **总分** | 所有子特性得分之和（满分 = 29 + 8×0.5 = 33） |
| **数据来源** | 直接来自垂直分析报告；报告缺失则回查源码确认 |

---

## 1. 任务编排（任务拆解与逻辑控制）

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| SubAgent 委派 | 拆解为独立子Agent | ✅ 机制: sessions_spawn+registry 管理子会话。效果: 多subagent独立workspace | ✅ 机制: SubGraphAgent三种形态(声明式/预编译/远程)。效果: 最灵活定义方式 | ✅ 机制: 双线程池(3+3)并发调度。效果: 主副线程池隔离 | ✅ 机制: delegate_task tool 支持 leaf/orchestrator。效果: 独立工具集+上下文 | ⚠️ 机制: customSubagents配置。效果: IDE内自定义(闭源) | ✅ 机制: 6 built-in+3套并行(SubAgent/Coordinator/Swarm)。效果: 最丰富体系 | ✅ 机制: codex_delegate.rs SubAgentSource+图拓扑。效果: 图状非树状关系 | ✅ 机制: agent mode subagent/primary/all。效果: 三级角色+per-Agent permission |
| 并行执行 | 互不依赖任务同时执行 | ✅ 机制: parallel subagent+grouped results。效果: 总耗时=max(子任务) | ✅ 机制: AsyncSubAgent+LangGraph并行节点。效果: 图式任意拓扑 | ✅ 机制: MAX=3并发限制。效果: 最多3并行 | ✅ 机制: delegate_task max=3。效果: 最多3并行 | ❓ | ✅ 机制: Swarm teammates全并行。效果: Swarm模式所有teammate并发 | ✅ 机制: tokio::spawn 异步event/op通道。效果: 多子Agent异步并发 | ❌ |
| Plan/Todo | 自动创建待办跟踪进度 | ✅ 机制: update_plan tool(experimental)。效果: 需显式opt-in开启 | ✅ 机制: TodoListMiddleware(MW栈第6层)。效果: 自动生成更新 | ✅ 机制: TodoListMiddleware(MW栈第5层)。效果: 自动Todo | ❌ | ❌ | ✅ 机制: TaskCreate/TaskStop工具。效果: Agent自主管理任务 | ✅ 机制: 三层体系(Goal+PlanMode+TodoList)。效果: 持久化目标+回合清单+协作planning | ❌ |
| Loop 检测 | 防止陷入死循环 | ✅ 机制: 3种detector+circuitBreaker+postCompactionGuard。效果: 三级检测+熔断 | ✅ 机制: PatchToolCalls自动修复。效果: 修复即破坏循环 | ✅ 机制: LoopDetectionMW独立中间件。效果: 硬中断循环 | ✅ 机制: tool_loop_guardrails。效果: Guardrails内置检测 | ❓ 闭源 | ✅ 机制: autoCompact circuit breaker(MAX=3连续失败)。效果: 跳闸后停止, 每天节省~250K API调用 | ❌ 机制: 仅GuardianRejectionCircuitBreaker(拒绝限流器,非工具循环检测)。效果: turn loop无界运行,注释称"as long as compaction works...shouldn't worry" | ✅ 机制: DOOM_LOOP=3+continue_loop_on_deny+maxSteps。效果: 5层防护 |
| 上下文压缩 | Token超限自动压缩 | ✅ 机制: auto-compaction+midTurnPrecheck+retry+notifier。效果: 自动触发+中途检查+重试 | ✅ 机制: SummarizationMW(85%阈值触发)。效果: 保留10%最近消息 | ✅ 机制: DeerFlowSummarizationMiddleware(enabled=false默认)。效果: 启用后摘要+skill保护 | ✅ 机制: compressor自动触发(50%阈值)。效果: 半自动压缩 | ❓ 闭源 | ✅ 机制: compact+hooks回调。效果: 可编程压缩, threshold=contextWindow-13000 | ✅ 机制: compact_remote v1+v2远程压缩。效果: 双策略迭代 | ✅ 机制: isOverflow+SUMMARY_TEMPLATE+prune。效果: tail_turns保留原文+旧消息摘要 |
| 异常重启恢复 | crash后自动/手动恢复 | ✅ 机制: Lobster引擎检查点+SQLite task_registry+restart-sentinel。效果: 自动扫描interrupted任务恢复 | ✅ 机制: LangGraph Checkpointer(SqliteSaver)。效果: graph状态检查点, 断点续跑 | ⚠️ 机制: 依赖LangGraph Checkpointer(SQLite/PG)。效果: 无显式crash recovery命令 | ✅ 机制: batch_runner checkpoint+SQLite FTS5。效果: batch模式恢复+session持久化 | 🔒 闭源(Cloud Agent支持resume) | ✅ 机制: /resume(Fuse.js模糊)+--continue+远程URL恢复。效果: 模糊匹配+远程+CLI续跑 | ⚠️ 机制: TurnState+agent-graph-store持久化。效果: Session Picker间接恢复 | ⚠️ 机制: retry/revert/run-state三层恢复。效果: 会话级恢复 |
| 工具调用恢复 | 失败后补丁/重试/降级 | ✅ 机制: orphan recovery+compaction retry+tool-result guard。效果: 孤儿恢复+压缩重试+守卫 | ✅ 机制: PatchToolCalls自动修复参数。效果: 工具调用修复 | ✅ 机制: LoopDetection hard stop即恢复。效果: 硬中断恢复 | ✅ 机制: Guardrails hard_stop。效果: Guardrails干预恢复 | ❓ 闭源 | ✅ 机制: StreamingToolExecutor含重试。效果: 推断支持 | ❌ 机制: FunctionCallError::RespondToModel 仅将错误文本注入对话, 模型自行决定重试。传输层重试仅针对WebSocket断开/超时, is_retryable()对工具错误返回false。效果: 无自动恢复 | ❌ |
| 对话崩溃恢复 [wl] | 上下文撑爆等恢复 | ✅ 机制: compaction质量守卫+retry+安全margin。效果: auto-summarization含重试+标识符保留 | ✅ 机制: SummarizationMW自动摘要。效果: Token超预算时自动压缩 | ⚠️ 机制: SummarizationMiddleware存在但enabled=false(默认)。效果: 启用后含token触发+skill救援, 禁用时上下文溢出直接报错崩溃 | ⚠️ 机制: max_iterations+budget tracking+checkpoint。效果: 无auto-compaction恢复 | ⚠️ 机制: 受限于IDE宿主上下文限制 | ✅ 机制: auto-compaction+quality guard。效果: 逼近上限时自动压缩 | ⚠️ 机制: 上下文窗口管理+截断。效果: 无显式压缩重试 | ⚠️ 机制: 基础compaction无质量guard。效果: 无重试机制 |

| **得分** | 8子特性 | OC=8 | DA=7.75 | DF=5.5 | HA=6.75 | CS=0.75 | CC=7.75 | CX=4.5 | OP=4.75 |
|------|------|------|------|------|------|------|------|------|------|

## 2. 记忆系统（数据持久化与存储）

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Session 持久化 | 对话不随重启丢失 | ✅ 机制: Workspace文件持久化。效果: 会话文件化, per-agent隔离 | ✅ 机制: LangGraph checkpointer。效果: Graph状态检查点 | ✅ 机制: SQLite/PostgreSQL可切换。效果: 数据库持久化 | ✅ 机制: SQLite FTS5全文搜索。效果: 搜索+持久化双重 | ✅ 机制: SQLite基础存储。效果: 基础持久化 | ✅ 机制: sessionStorage+transcript完整存储。效果: 可重放+重续 | ✅ 机制: agent-graph-store(SQLite)。效果: Agent关系图持久化 | ✅ 机制: SQLite(Drizzle ORM)。效果: 类型安全, 4表结构 |
| 短期记忆 | 会话内上下文保持 | ✅ 机制: compaction自动压缩(reserveTokens+maxHistoryShare+qualityGuard)。效果: Token超限自动触发, safeguard模式 | ✅ 机制: SummarizationMW(85%阈值)।效果: 自动摘要, 卸出到文件 | ✅ 机制: LLM驱动Memory(30s防抖队列)。效果: 事实提取+去重+atomic write | ✅ 机制: compressor自动触发(50%阈值)。效果: 压缩到20%+FTS5搜索 | 🔒 闭源 | ✅ 机制: 五层分层记忆(唯一区分短期/长期)。效果: sessionStorage→memdir→SessionMemory→compact→hooks | ✅ 机制: compact_remote v1+v2+TokenUsageBreakdown。效果: 远程压缩+精确Token分解 | ✅ 机制: compaction.ts(652行) isOverflow+SUMMARY_TEMPLATE+prune。效果: tail_turns保留原文+旧消息摘要 |
| 长期记忆 | 跨会话用户偏好/事实/规范 | ✅ 机制: AGENTS.md/SOUL.md+sqlite-vec向量库。效果: 声明式+语义搜索+per-agent隔离 | ✅ 机制: AGENTS.md文件解析。效果: 项目级上下文注入 | ✅ 机制: MemoryMiddleware(MW栈#12)。效果: 自动记忆管理+user隔离 | ✅ 机制: Honcho/mem0双后端+AGENTS.md/SOUL.md。效果: 自适应+多用户+规范分离 | ❌ | ✅ 机制: memdir+SessionMemory五层+CLAUDE.md。效果: 最完整分层长期记忆 | ❌ 机制: 仅agents_md.rs项目级+拓扑持久化。效果: 无跨session用户记忆 | ❌ 机制: 仅会话内compaction(SUMMARY_TEMPLATE)。效果: 无跨会话记忆 |
| 长期记忆搜索 | 语义/全文检索历史 | ✅ 机制: sqlite-vec本地向量数据库。效果: 本地语义检索 | ❌ | ✅ 机制: /api/threads/search metadata过滤+分页。效果: 无FTS全文 | ✅ 机制: FTS5全文+LLM摘要+LLM语义三重搜索。效果: 全文本+语义 | 🔒 闭源 | ✅ 机制: /resume(Fuse.js)+agenticSearch(AI语义)+transcriptSearch(实时)三重。效果: 模糊+语义+实时 | ✅ 机制: JSONL索引+SQLite instr()+TUI Picker(25条)+Ctrl+R四重。效果: 多层但无FTS5 | ❌ |
| Skills 自动进化 [wl] | 自动创建和改进技能 | ✅ 机制: Skill Workshop插件(transcript扫描+before_prompt钩子)。效果: auto/pending双模, 快照刷新 | ❌ | ❌ | ✅ 机制: Curator全生命周期(active→stale→archived)。效果: 3态流转+LLM review+90+威胁扫描+快照回滚 | ❌ | ❌ | ❌ | ❌ |

| **得分** | 5子特性 | OC=4.25 | DA=3 | DF=2.5 | HA=4.25 | CS=0.75 | CC=4 | CX=2.5 | OP=1.5 |
|------|------|------|------|------|------|------|------|------|------|

## 3. 执行沙箱（代码与环境运行）

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Shell 执行 | 系统命令执行能力 | ✅ 机制: child_process+sandbox隔离。效果: Docker/SSH/OpenShell可选 | ✅ 机制: execute tool+120s timeout+5种Backend。效果: background异步执行 | ✅ 机制: bash tool+虚拟路径翻译+4层隔离。效果: 输出截断(20K chars) | ✅ 机制: 6种终端后端(local/docker/ssh/daytona/modal/singularity)。效果: PTY交互支持 | ✅ 机制: IDE内Shell。效果: 受限于IDE沙箱 | ✅ 机制: Shell.ts(bwrap封装)。效果: 容器内执行 | ✅ 机制: Rust std::process+Landlock LSM。效果: 内核级沙箱约束 | ✅ 机制: node-pty伪终端。效果: 支持vim/npm等交互CLI |
| 文件操作 | 读写文件能力 | ✅ 机制: fs promises+原子写入。效果: 读/写/追加/创建目录 | ✅ 机制: 6种Backend虚拟文件系统。效果: 与执行环境解耦 | ✅ 机制: read_file/write_file/str_replace(per-sandbox锁)。效果: 防并发冲突 | ✅ 机制: read_file(行号)/search_files(ripgrep)/write_file(自动mkdir)。效果: 编辑器级 | ✅ 机制: IDE文件API。效果: 受限于IDE | ✅ 机制: 文件操作tool。效果: 推断完整 | ✅ 机制: Rust std::fs。效果: 原生IO | ✅ 机制: AppFileSystem(Effect-TS)。效果: 类型安全+自动错误处理 |
| Browser | 操控真实浏览器 | ✅ 机制: Playwright集成。效果: 点击/截图/表单填写 | ❌ | ❌ | ✅ 机制: 12个browser_* tools(Playwright/CDP)。效果: 导航/点击/iframe+坐标点击 | ❌ | ❌ | ❌ | ✅ 机制: agent-browser CLI。效果: snapshot获取交互ref |
| 代码执行 | 直接运行代码 | ❌ | ✅ 机制: LocalShellBackend+BaseSandbox。效果: 隔离Python/shell执行 | ⚠️ 机制: bash tool+Docker沙箱。效果: 仅bash非专用Python/JS沙箱 | ✅ 机制: PTC沙箱+UDS socket RPC。效果: 子进程运行LLM脚本, 结果不进入上下文 | ❌ | ❌ | ✅ 机制: Rust exec crate。效果: 沙箱化Python/JS/TS执行 | ❌ |
| Sandbox 后端 | 多种沙箱接入 | ✅ 机制: Docker/SSH/OpenShell(3种)。效果: 进程/主机/Serverless可选 | ✅ 机制: 5种Backend(State/FS/Shell/Store/Composite)。效果: 虚拟化执行 | ✅ 机制: 4层隔离(Session/Request/Resource/Network)。效果: 纵深防御 | ✅ 机制: Docker/SSH/Daytona(3种)+daemon管理。效果: persistent开关+空闲清理 | ⚠️ 机制: 局部沙箱(闭源)。效果: 不可审计 | ✅ 机制: bwrap+Seatbelt+macOS runtime(3层)+SandboxDoctor。效果: 跨平台双重隔离+诊断 | ✅ 机制: Landlock(Linux 5.13+ LSM)+bwrap。效果: 唯一内核级LSM沙箱 | ✅ 机制: containers/(Docker+预构建镜像)。效果: bun-node/rust/tauri-linux |
| 沙箱生命周期 | 创建/复用/销毁 | ✅ 机制: mode(off/non-main/all)+scope(agent/session/shared)+CLI管理。效果: 四级配置 | ✅ 机制: 5 Provider协议化+acquire/release/shutdown。效果: Provider决定 | ✅ 机制: acquire/release/shutdown+lazy_init+Thread复用。效果: 显式API | ✅ 机制: 6 Backend+daemon空闲清理+persistent。效果: 自动清理。注意:task_id默认共享 | 🔒 闭源 | ✅ 机制: cleanupAfterCommand+SandboxManager。效果: 命令结束自动清理 | ⚠️ 机制: Landlock进程级。效果: 命令结束即销毁 | ⚠️ 机制: Docker容器生命周期。效果: 无内置管理 |
| 沙箱空闲管理 | idle销毁/active重建 | ✅ 机制: scope(agent/session/shared)自动清理。效果: session结束scope触发 | ⚠️ 机制: Provider决定, 无统一idle管理 | ✅ 机制: lazy_init+Thread复用+shutdown()。效果: idle释放, active重建 | ✅ 机制: _cleanup_inactive_envs+daemon+persistent开关。效果: daemon自动清理空闲 | 🔒 闭源 | ✅ 机制: cleanupAfterCommand+SandboxManager。效果: 下次执行重建 | ⚠️ 机制: Landlock进程级, 无idle概念 | ⚠️ 机制: docker-compose管理, 无内置idle检测 |

| **得分** | 7子特性 | OC=5 | DA=4.5 | DF=5 | HA=6.5 | CS=1 | CC=5.5 | CX=4 | OP=4 |
|------|------|------|------|------|------|------|------|------|------|

## 4. 安全隔离（权限与合规）

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| 工具白名单 | 限制可调用工具 | ✅ 机制: Allow/Deny per-agent配置。效果: 每Agent独立白名单 | ⚠️ 机制: 文件权限检查(wcmatch.glob)。效果: 仅文件操作 | ✅ 机制: 4层权限+Guardrail AllowlistProvider。效果: Session/Resource/Network级 | ✅ 机制: Guardrails工具拦截。效果: 运行时拦截+循环检测 | 🔒 闭源 | ✅ 机制: bashPermissions+Tool Permission双层。效果: 命令级+工具级 | ✅ 机制: Guardian审批系统。效果: 独立审批子系统 | ✅ 机制: allow/deny/ask三级+DB持久化+pattern match。效果: 精确命令匹配 |
| SSRF 防护 | 防止访问内网 | ✅ 机制: ssrf-policy中间件。效果: 阻断内网IP | ❌ | ⚠️ 机制: SandboxAuditMiddleware拦截/dev/tcp等bash级网络访问。效果: bash级阻断, 无URL级HTTP过滤 | ✅ 机制: website_blocklist。效果: 黑名单URL | 🔒 闭源 | ✅ 机制: bwrap net namespace网络隔离。效果: 全网络隔离 | ✅ 机制: CODEX_SANDBOX_NETWORK_DISABLED。效果: 全局禁用 | ✅ 机制: FenceMiddleware。效果: 服务端拦截 |
| 工具审批 | 调用前人工审批(HITL) | ✅ 机制: approval-handler多级+exec-approval+LobsterApprovalWaitState。效果: Tool/Group级+断点热启动+Pending Queue | ✅ 机制: HumanInTheLoopMiddleware per-tool interrupt_on。效果: 每工具独立HITL | ✅ 机制: GuardrailMiddleware(Allowlist/OAP/Custom)。效果: 模式匹配审批 | ✅ 机制: approval.py四级(permanent/session YOLO/gateway/Smart Approvals)。效果: 最细粒度 | 🔒 闭源 | ✅ 机制: bashPermissions+PermissionRule+弹窗审批。效果: 双重审批+UI | ✅ 机制: Guardian独立审批子系统(approve/reject)。效果: 独立审批 | ✅ 机制: allow/deny/ask+pattern+DB持久化。效果: 精确匹配+持久化 |
| MCP 审批 [wl] | MCP调用人工审批 | ⚠️ 机制: before_tool_call hook+runTrustedToolPolicies统一通道。效果: 非独立MCP通道 | ❌ | ❌ | ❌ | 🔒 闭源 | ⚠️ 机制: MCP作为native tool通过bashPermissions统一。效果: 非独立MCP | ✅ 机制: 三级审批(ACCEPT/ACCEPT_FOR_SESSION/DECLINE)。效果: 唯一独立MCP审批 | ⚠️ 机制: 统一Permission+continue_loop_on_deny。效果: 非独立 |
| MCP 凭证 | MCP凭证管理 | ✅ 机制: mcp-stdio/transport config+env/headers/secrets per-server。效果: per-server凭证隔离 | ⚠️ 机制: 通过langchain-mcp-adapters桥接。效果: 凭证委托适配器 | ✅ 机制: McpServerConfig(env+headers)+McpOAuthConfig(OAuth2.0 client_credentials/refresh_token)+$VAR解析+OAuthTokenManager缓存刷新。效果: 三层凭证(env/headers/OAuth) | ✅ 机制: mcp_oauth.py OAuth2.1+PKCE+HermesTokenStorage。效果: 动态注册+持久化token | ⚠️ 机制: 通过扩展机制。效果: 凭证管理依赖扩展 | ✅ 机制: MCP集成含凭证管理 | ✅ 机制: MCP集成含凭证配置 | ✅ 机制: 基础MCP server配置+env凭证 |
| Skill 审批 [wl] | 技能调用前确认 | ❌ | ✅ 机制: HumanInTheLoopMiddleware interrupt_on含技能。效果: 审批/修改/拒绝 | ❌ | ✅ 机制: skills_guard.py安全扫描(90+威胁模式)+slash_confirm。效果: 信任分级+威胁检测+按钮确认 | ⚠️ 机制: 依赖扩展权限。效果: 非一等特性 | ✅ 机制: Permission系统ask/allow/deny。效果: 含技能调用审批 | ✅ 机制: Permission系统用户确认。效果: 工具调用确认 | ❌ |
| Agent 隔离 | 同实例多Agent资源隔离 | ✅ 机制: agents.list[] per-agent skills/tools/model。效果: 完全替换非合并, per-agent skillsLimits | ✅ 机制: SubGraphAgent三种形态+per-subagent middleware。效果: 独立配置 | ✅ 机制: custom_agents独立system_prompt/tools/model。效果: per-agent隔离 | ⚠️ 机制: delegate_task子Agent+独立工具集。效果: 无agents.list语法 | 🔒 闭源(Cloud Agent支持) | ✅ 机制: 6 built-in+3套+per-agent worktree。效果: 最丰富Multi-Agent隔离 | ✅ 机制: CodexSpawnArgs全协议+图拓扑。效果: per-agent全配置 | ✅ 机制: Agent mode+per-Agent permission+model。效果: 三级角色 |
| 审计日志 | 完整记录操作 | ✅ 机制: io.audit.ts双层(config-audit.jsonl+Gateway JSONL)+进程级审计。效果: 前/后hash+inode+uid/gid+备份恢复 | ❌ | ⚠️ 机制: 每日文件夹+JSONL归档。效果: 非实时审计 | ✅ 机制: 3个RotatingFileHandler+RedactingFormatter+30+脱敏+session关联。效果: 实时审计 | 🔒 闭源 | ✅ 机制: sessionStorage完整持久化+transcript。效果: 可重放 | ✅ 机制: RolloutRecorder JSONL持久化(EventMsg/ResponseItem/CompactedItem全量)到~/.codex/sessions/。效果: 每turn持久化, 可重放+审查 | ❌ |

| **得分** | 8子特性 | OC=5.75 | DA=2.75 | DF=3.5 | HA=5.75 | CS=0.25 | CC=6.25 | CX=4.75 | OP=4.5 |
|------|------|------|------|------|------|------|------|------|------|

## 5. 扩展生态（协议与外部交互）

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Skill 定义 | Markdown/YAML声明式 | ✅ 机制: SKILL.md+ClawHub+5级加载优先级+per-agent allowlist。效果: 社区分享+per-agent控制 | ✅ 机制: SKILL.md+Discovery自动扫描。效果: 声明式 | ✅ 机制: SKILL.md+配置文件。效果: 声明式 | ✅ 机制: SKILL.md+agentskills.io Hub+YAML frontmatter+progressive disclosure。效果: 社区+分层加载 | ❌ | ✅ 机制: Skills系统(SKILL.md兼容)。效果: 声明式 | ❌ | ✅ 机制: SKILL.md三级发现(.claude/.agents/skill/)。效果: 三目录兼容 |
| mcp client | 消费外部MCP工具 | ✅ 机制: MCP client+auto-discovery。效果: 即插即用数百Server | ✅ 机制: langchain-mcp-adapters桥接。效果: MCP工具调用 | ✅ 机制: MCP client。效果: 工具消费 | ✅ 机制: MCP client+mcp_serve(Server端)。效果: 双向MCP | ✅ 机制: MCP集成(stdio/HTTP/SSE)。效果: 工具调用 | ✅ 机制: MCP集成。效果: 工具调用 | ✅ 机制: codex-mcp crate(自研Rust 4666行)。效果: 原生实现 | ✅ 机制: MCP模块(1521行)。效果: 自研client |
| 自定义工具 | 扩展外部工具 | ✅ 机制: 80+ plugin-sdk导出(channel/auth/reply/sandbox/TTS)。效果: 几乎无不可扩展 | ✅ 机制: Middleware API 13层+tools参数(Callable/dict)。效果: 代码式扩展 | ⚠️ 机制: 反射式Provider加载。效果: 无独立Plugin SDK | ✅ 机制: Skill定义工具+toolset组合。效果: 声明式+代码式 | 🔒 闭源(Plugin存在) | ✅ 机制: 184工具文件+Plugin架构。效果: 文件扩展 | ⚠️ 机制: core-plugins+core-skills crate。效果: 细节待确认 | ✅ 机制: Plugin SDK(Hooks 2767行)+多认证插件。效果: 动态加载 |
| A2A [wl] | Agent间发现/通信/协作 | ✅ 机制: sessions-send-tool.a2a.ts ping-pong+announce/reply。效果: 跨Agent路由+agentToAgent配置 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| 虚拟文件系统 [wl] | Agent长期记忆交互层 | ❌ | ✅ 机制: 6种Backend(State/FS/Shell/Store/Composite)+BackendProtocol。效果: 完整VFS抽象 | ✅ 机制: 虚拟路径翻译+skills目录自动挂载。效果: host↔container映射 | ❌ | ❌ | ❌ | ❌ | ❌ |

| **得分** | 5子特性 | OC=3.5 | DA=2.5 | DF=2.5 | HA=3 | CS=0.75 | CC=3 | CX=1.5 | OP=3 |
|------|------|------|------|------|------|------|------|------|------|

## 6. 可观测性 (Observability)

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| 对接 Langfuse | 链路追踪管理 | ❌ | ❌ | ❌ | ✅ 机制: plugins/observability/langfuse/ 完整集成。效果: trace/span/generation追踪 | ❌ | ❌ | ❌ | ❌ |
| Session Event Type | 定义监控会话事件 | ✅ 机制: transcript-events+subagent-lifecycle-events。效果: 会话事件管道 | ⚠️ 机制: LangGraph checkpoint事件。效果: 无显式类型定义 | ✅ 机制: run_events(memory/db/jsonl)+token追踪。效果: 事件存储管道 | ⚠️ 机制: SessionDB+FTS5+session文件。效果: 无显式事件总线 | ⚠️ 机制: VS Code扩展事件。效果: 非专用 | ⚠️ 机制: 基础session/transcript追踪。效果: 无类型系统 | ⚠️ 机制: 内部事件追踪。效果: 无公开类型 | ❌ |

| **得分** | 2子特性 | OC=1 | DA=0.5 | DF=1 | HA=1.5 | CS=0.5 | CC=0.5 | CX=0.5 | OP=0 |
|------|------|------|------|------|------|------|------|------|------|

## 7. 社区规模 (Community Metrics)

| 子特性 | 特征作用 | OpenClaw | Deep Agents | DeerFlow 2.0 | Hermes Agent | Cursor SDK | Claude Code | Codex | OpenCode |
|------|---------|----------|-------------|--------------|--------------|------------|-------------|-------|----------|
| Star 数 | 项目受关注度 | ✅ 368k ★ | ✅ 22k ★ | ✅ 64k ★ | ✅ 132k ★ | ✅ 33k ★ | ⚠️ 泄露分析 | ✅ 80k ★ | ✅ 155k ★ |
| Commit 日更频率 | 代码活跃度 | ✅ 高(日更) | ✅ 中 | ✅ 中高 | ✅ 高 | ⚠️ 闭源 | ⚠️ 泄露(无法统计) | ✅ 中 | ✅ 高 |

| **得分** | 2子特性 | OC=2 | DA=2 | DF=2 | HA=2 | CS=1 | CC=0.5 | CX=2 | OP=2 |
|------|------|------|------|------|------|------|------|------|------|

---

## 总分汇总

| 排名 | 产品 | 总分(/33) | 任务编排 | 记忆系统 | 执行沙箱 | 安全隔离 | 扩展生态 | 可观测性 | 社区规模 |
|:---:|------|:------:|:------:|:------:|:------:|:------:|:------:|:------:|:------:|
| **1** | **Hermes Agent** | **29.75** | 6.75 | 4.25 | 6.5 | 5.75 | 3 | 1.5 | 2 |
| **2** | **OpenClaw** | **29.50** | 8 | 4.25 | 5 | 5.75 | 3.5 | 1 | 2 |
| 3 | Claude Code | 27.50 | 7.75 | 4 | 5.5 | 6.25 | 3 | 0.5 | 0.5 |
| 4 | DeerFlow 2.0 | 22.00 | 5.5 | 2.5 | 5 | 3.5 | 2.5 | 1 | 2 |
| 5 | Codex | 19.75 | 4.5 | 2.5 | 4 | 4.75 | 1.5 | 0.5 | 2 |
| 6 | Deep Agents | 20.00 | 7.75 | 3 | 4.5 | 2.75 | 2.5 | 0.5 | 2 |
| 7 | OpenCode | 17.75 | 4.75 | 1.5 | 4 | 4.5 | 3 | 0 | 2 |
| 8 | Cursor SDK | 4.25 | 0.75 | 0.75 | 1 | 0.25 | 0.75 | 0.5 | 1 |

> 说明: Cursor SDK 因闭源，多个特性无法确认，得分偏低。满分 = 29(全权重) + 8×0.5([wl]) = 33。

---

## 数据来源

本报告所有特性判定基于以下 Phase 1 垂直能力解构报告及源码补充验证：

| 产品 | 报告文件 | 分析方法 | 源码支撑率 |
|------|---------|---------|:---------:|
| OpenClaw | `openclaw-vertical-capability-report.md` | 源码 + docs 仓库 | ~90% |
| Deep Agents | `deepagents-vertical-capability-report.md` | 源码直读 (Python) | ~90% |
| DeerFlow 2.0 | `deerflow-2.0-vertical-capability-report.md` | 源码直读 (Python) | ~90% |
| Hermes Agent | `hermes-vertical-capability-report.md` | 源码直读 (Python) | ~90% |
| Cursor SDK | `cursor-sdk-vertical-capability-report.md` | npm tarball 黑盒 | ~40% |
| Claude Code | `claude-code-vertical-capability-report.md` | 泄露源码直读 (TypeScript) | ~80% |
| Codex | `codex-vertical-capability-report.md` | Rust 80+ crates 直读 | ~85% |
| OpenCode | `opencode-vertical-capability-report.md` | Effect-TS monorepo 直读 | ~90% |