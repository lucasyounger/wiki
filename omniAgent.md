# OmniAgent 设计文档

> OmniAgent 代码仓深度分析：基于 Mastra 的多智能体编排平台，以复合式工程和上下文工程为核心设计理念。本文档反映 2026-05-12 代码仓实际状态。

---

## 1. 概述

OmniAgent 是一个本地 Mastra 智能体团队（Agent Team），通过持久化协调协议将四个专业智能体和一个 Runtime facade 层编排在一起。它以 LibSQL 支持的对话记忆和文件支持的 docs 系统作为规范长期记忆。Omni Gateway 作为独立的通道适配层，将移动消息应用接入 Team Runtime 协议。

**技术栈：** TypeScript, Mastra `1.32.1`, LibSQL, Vitest, Zod `4.x`, Croner
**模型供应商：** DeepSeek（通过 Mastra AI SDK）
**模型 ID：** `deepseek/deepseek-v4-flash`（所有智能体）
**入口：** `src/mastra/index.ts` → `npm run dev`（Mastra Studio 默认运行在 `localhost:4111`）
**Gateway 入口：** `src/gateway/main.ts` → `npm run gateway`（HTTP 服务默认运行在 `localhost:4120`）

---

## 2. 架构：复合式工程

### 2.1 多智能体拓扑

OmniAgent 采用**轮辐式（hub-and-spoke）**智能体架构，OmniRouterAgent 为 supervisor 式中心协调器，通过注册 sub-agents 和 workflows 实现委派。任意智能体均可通过 Team Runtime 协议发起工作任务。Omni Gateway 作为外部通道层，将移动消息应用的消息转换为 Team Runtime 任务。

`src/mastra/runtime/` 作为统一的 facade 层，封装对底层库模块的调用，并为向 Mastra 原生 workflow/scheduler/storage 能力的迁移提供统一接口。

```
                         ┌──────────────────────────────┐
                         │       Omni Gateway            │
                         │   (HTTP/OneBot/QQBot 通道)     │
                         └────────────┬─────────────────┘
                                      │ /task → Team Runtime
                                      │ 自然语言 → OmniRouterAgent
                                      ▼
                    ┌──────────────────────┐
                    │   OmniRouterAgent    │
                    │  (Supervisor 中心)    │
                    │  + sub-agents        │
                    │  + workflows         │
                    └──────┬───────────────┘
                           │ 通过 Team Runtime 委派
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────┐    ┌──────────┐    ┌──────────────┐
   │CodeAgent │    │CronAgent │    │KnowledgeAgent│
   │(ClaudeCLI)│   │(调度器)   │    │(文档记忆)     │
   └──────────┘    └──────────┘    └──────────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
   ┌──────────────────┐          ┌──────────────────┐
   │ TaskRuntime      │          │ Tool Gateway     │
   │ SchedulerRuntime │          │ (risk/audit)     │
   │ MemoryRuntime    │          │                  │
   │ Events/Bootstrap │          │                  │
   └──────────────────┘          └──────────────────┘
          │                                 │
          ▼                                 ▼
   ┌──────────────────────────────────────────────┐
   │     Mastra 原生能力                             │
   │  (workflows, backgroundTasks, scheduler,       │
   │   LibSQL storage, Memory)                      │
   └──────────────────────────────────────────────┘
```

### 2.2 智能体角色

| 智能体 | 状态 | 职责 | Sub-Agents / Workflows |
|---|---|---|---|
| **OmniRouterAgent** | 活跃 Mastra Agent | Supervisor 式意图路由、委派、最终回复合成 | 注册 CodeAgent, CronAgent, KnowledgeAgent 为 sub-agents；注册 taskOrchestrationWorkflow, runCodeTaskWorkflow, memoryMaintenanceWorkflow |
| **CodeAgent** | 活跃 Mastra Agent | Claude Code CLI 任务执行、进度汇报、持久化任务索引 | `start-claude-code-task` 工具 |
| **CronAgent** | 活跃 Mastra Agent | 定时任务生命周期、进程内调度器、标准 cron 表达式支持 | 定时任务的增删改查、手动触发、下次执行时间解释 |
| **KnowledgeAgent** | 活跃 Mastra Agent | 文档支持的记忆维护、索引刷新、用户档案管理 | 6 个 memory 工具（全部通过 Tool Gateway 执行） |
| **TaskAgent** | 协议角色（非 Agent 类） | Team Runtime 协议所有权 | 14 个运行时工具 |

### 2.3 Runtime Facade 层 —— 统一的迁移边界

`src/mastra/runtime/` 是 Round 1 引入的统一 facade 层。每个 facade 委托给现有的库模块，同时为向 Mastra 原生能力的渐进式迁移提供清晰的边界：

| 模块 | 文件 | 委托至 | 用途 |
|---|---|---|---|
| **TaskRuntime** | `runtime/task-runtime.ts` | `lib/team-runtime-store.ts` | Task 创建、查询、状态映射、取消、重试 |
| **SchedulerRuntime** | `runtime/scheduler-runtime.ts` | `lib/cron-store.ts` | Schedule 创建、列表、暂停、恢复、删除、手动触发 |
| **MemoryRuntime** | `runtime/memory-runtime.ts` | `lib/docs-memory.ts` | Docs 列表/读取、情景日志追加、更新提案、索引刷新、用户档案 |
| **Tool Gateway** | `runtime/tool-gateway.ts` | — | 工具调用包装器：风险级别、审计记录、敏感字段脱敏 |
| **Events** | `runtime/events.ts` | `lib/team-runtime-store.ts` | Team Event 追加 |
| **Store** | `runtime/store.ts` | — | LibSQL 存储实例（`omni-agent.db`） |
| **Memory** | `runtime/memory.ts` | `@mastra/memory` | Agent Memory 构造（使用 LibSQL storage） |
| **Bootstrap** | `runtime/bootstrap.ts` | — | 启动恢复：interrupted 标记、timed_out 扫描、cron 调度器启动 |

导出的 `RuntimeTask` 类型提供 Team Task 到统一状态模型的映射：
```
queued→pending, running→running, completed→succeeded, failed/cancelled/interrupted/timed_out→failed
```

### 2.4 Team Runtime 协议 —— 通用协调层

Team Runtime 是 OmniAgent 中**最关键的架构决策**。它是一个协议，不绑定任何单一智能体。所有智能体（以及 Omni Gateway）都通过它完成委派工作。

```
源智能体/Gateway ──► Team Task ──► Team Run ──► Team Events（进度记录）
                           │              │
                           │              ▼
                           │         Team Result（持久化输出）
                           │              │
                           ▼              ▼
                    Inbox 通知 ←──────────┘
                           │
                           ▼
                    OmniRouterAgent 通过 resultRef 读取结果
```

**核心记录（文件存储，后续迁移至 LibSQL）：**

| 记录 | 存储位置 | 用途 |
|---|---|---|
| **Task** | `docs/runs/team/tasks.json` | 持久化工作请求（源智能体 → 目标智能体） |
| **Run** | `docs/runs/team/runs.json` | 单次执行尝试，含尝试计数器 |
| **Event** | `docs/runs/team/events.jsonl` | 追加式进度/状态变更日志 |
| **Inbox** | `docs/runs/team/inbox/{agentId}.jsonl` | 跨智能体通知（非真相来源） |
| **Result** | `docs/runs/team/results/{runId}.json` | 持久化执行输出 |

**任务状态状态机：**
```
queued → running → completed / failed / cancelled / interrupted / timed_out
                       │
                       ▼ （重试）
                  新的 queued 任务（带 retryOfTaskId）
```

**可靠性保证：**
- 启动恢复：`bootstrapRuntimeCompatibility()` 在 `src/mastra/index.ts` 中调用，将 `running` 状态的 run 标记为 `interrupted`，标记 `timed_out` 的过期 run
- 超时扫描：定期轮询（`OMNI_TEAM_TIMEOUT_POLL_INTERVAL_MS`，默认 30 秒）
- 重试：创建新任务并通过 `retryOfTaskId` 链接，保留完整历史
- 取消：取消运行中的 run 或直接取消排队中的 task
- 通知：完成/失败/中断/超时始终向源智能体和 `omni-router-agent` 发送收件箱消息

### 2.5 Tool Gateway —— 风险管控与审计层

Round 4 引入的 Tool Gateway 是所有工具调用的统一执行包装器。每个工具调用经过网关时记录审计事件，敏感字段自动脱敏。

**风险级别：**

| 风险 | 含义 | 示例 |
|---|---|---|
| `safe` | 只读操作，无副作用 | memory.read |
| `medium` | 写入操作，可审查 | memory.write |
| `dangerous` | 代码执行，系统控制 | code.execution |

**审计记录：**
- 位置：`docs/runs/gateway/tool-audit.jsonl`
- 内容：toolId, policy, status (succeeded/failed), startedAt, completedAt, input, output/error
- 脱敏规则：匹配 `/(api[_-]?key|token|secret|password|credential|authorization|cookie)/i` 的 key 替换为 `[redacted]`；超过 4000 字符的字符串截断

**当前已迁移到 Tool Gateway 的模块：**
- Memory tools（6 个工具全部通过 `executeWithToolGateway` 执行）
- Code tools, Cron tools, Team Runtime tools（兼容路径保留）

**`defineGatewayTool`** 辅助函数将工具定义与策略绑定，供未来的强制执行和审批流程使用。

### 2.6 Task Orchestration Workflow

Round 2 引入的 `task-orchestration-workflow` 是 Mastra 原生 workflow，提供委派工作的稳定入口点：

```
taskOrchestrationWorkflow
  └─ create-runtime-task step
       └─ taskRuntime.createTask() → Team Task
```

输入 schema：sourceAgentId, targetAgentId, objective, priority, timeoutMs, metadata
输出 schema：统一 RuntimeTask 结构体（id, status, timestamps, metadata）

### 2.7 Code Task Store —— 持久化任务索引

Round 2 引入的持久化机制确保代码任务在进程重启后仍可查询：

- **内存 Map**：进程内高速访问（`tasks` Map）
- **持久化索引**：`docs/runs/code-runs/tasks.json` — 包含 taskId, teamTaskId, teamRunId, workspacePath, objective, status, startedAt, endedAt, exitCode, logFile
- **事件日志**：`docs/runs/code-runs/{taskId}.jsonl` — 追加式 stdout/stderr 事件流
- **恢复逻辑**：`getCodeTask()` 优先查内存，回退到持久化索引；`listCodeTasks()` 合并内存和持久化记录

### 2.8 Cron Store —— 多格式调度支持

Round 3 升级的 cron store 支持三种调度格式（通过 `croner` 库解析标准 cron）：

| 格式 | 示例 | 解析方式 |
|---|---|---|
| 标准 cron (5-7 字段) | `*/5 * * * *`, `0 9 * * 1-5` | `new Cron(expression)` → `nextRun()` |
| 一次性调度 | `2026-06-15 14:30` | 正则 + `new Date()` |
| 每日调度 (多语言) | `daily 09:30`, `每天 09:30`, `每日 09:30` | 正则 + 时间计算 |

**核心函数：**
- `getCronJobNextRunAt(job, now)` — 计算下次执行时间（支持所有三种格式）
- `runDueCronJobs(now)` — 轮询扫描，触发到期任务；一次性任务执行后自动 pause
- `startCronScheduler()` — 进程内调度器，默认 30 秒间隔（`OMNI_CRON_POLL_INTERVAL_MS`）

---

## 3. Omni Gateway —— 通道适配层

Omni Gateway 是一个独立的 Node.js HTTP 服务，作为外部消息应用（QQ Bot、OneBot 兼容桥等）与 OmniAgent 之间的适配层。它不是 Mastra Agent，而是以独立进程运行，通过 Team Runtime 协议和 Mastra Agent API 与 OmniAgent 交互。

**设计原则：**
- Gateway 仅是任务来源和投递层，不持有业务逻辑
- 所有执行流始终通过 Team Runtime 协议
- 通道适配器处理消息协议差异，统一为内部 `ChannelMessage` / `OutboundMessage` 类型

**架构组件（8 个文件）：**

| 组件 | 文件 | 职责 |
|---|---|---|
| **入口** | `src/gateway/main.ts` | 启动 HTTP 服务、投递工作器、可选 QQ Bot 适配器 |
| **配置** | `src/gateway/config.ts` | 环境变量加载（端口、配对令牌、允许列表、OneBot URL、QQBot 凭据） |
| **类型** | `src/gateway/types.ts` | `ChannelMessage`, `OutboundMessage`, `ChannelTarget`, `ChannelSession`, `DeliveryRecord` |
| **消息处理** | `src/gateway/message-handler.ts` | 消息授权、命令路由（`/task`, `/help`, `/status`, `/pair`）、自然语言转发至 OmniRouterAgent |
| **投递** | `src/gateway/delivery.ts` | 出站消息发送（demo/OneBot/QQBot）、Team Runtime 收件箱轮询、投递记录 |
| **HTTP 服务** | `src/gateway/http-server.ts` | 原生 Node.js HTTP 服务（`GET /health`, `POST /message`, `POST /onebot`） |
| **存储** | `src/gateway/gateway-store.ts` | JSON 文件持久化（`sessions.json`, `deliveries.json`） |
| **QQ Bot** | `src/gateway/qqbot-adapter.ts` | QQ Bot 凭证管理、WebSocket 连接（可选启用） |

**HTTP 端点：**

| 端点 | 方法 | 用途 |
|---|---|---|
| `/health` | GET | 健康检查，返回 `{ ok: true, service: "omni-gateway" }` |
| `/message` | POST | 通用 HTTP 通道，接受 `ChannelMessage` 结构体 |
| `/onebot` | POST | OneBot 兼容端点，自动转换 OneBot 消息格式（`post_type`, `message_type`, `sender`, `raw_message`） |

**消息处理流程：**

```
通道消息 → authorizeMessage()
   ├─ allowSenders 列表匹配 → 自动配对
   ├─ 已有配对会话 → 放行
   ├─ /pair <token> 匹配 → 新建配对会话
   └─ 未授权 → 返回配对提示

已授权消息 → 命令匹配
   ├─ /help → 返回帮助文本
   ├─ /status → 返回在线状态
   ├─ /task <workspace> :: <objective>
   │    → createTeamTask(source: "channel-gateway", target: "code-agent")
   │    → startClaudeCodeTask(teamTaskId, ...)
   │    → 回复任务已创建（含 Team Task ID, Team Run ID, Code Task ID）
   └─ 自然语言
        → POST /api/agents/omni-router-agent/generate
        → 同步返回 Router 回复
```

**投递工作器（Delivery Worker）：**

```
setInterval(2s) → listAgentInbox("channel-gateway", unread)
   → 从 Task metadata.source 提取通道 target
   → getRunResult(resultRef) 读取执行结果
   → sendOutbound() 推送回复到通道
      ├─ demo 通道 → console.log
      ├─ onebot 通道 → POST /send_private_msg 或 /send_group_msg
      └─ qqbot 通道 → POST /v2/users/{openid}/messages 或 /v2/groups/{group_openid}/messages
   → createDelivery() 记录投递
   → markInboxMessageRead()
```

**安全控制：**

| 机制 | 配置 | 说明 |
|---|---|---|
| 配对令牌 | `OMNI_GATEWAY_PAIRING_TOKEN` | 一次性配对令牌，用户发送 `/pair <token>` 完成授权 |
| 发送者允许列表 | `OMNI_GATEWAY_ALLOW_SENDERS` | 分号分隔的 senderId 列表，自动授权 |
| 路径隔离 | `assertAllowedWorkspace()` | 通过 `normalizeInside()` 进行严格的路径前缀检查，防止路径遍历 |

**Gateway 存储：**

| 文件 | 内容 | 用途 |
|---|---|---|
| `docs/runs/gateway/sessions.json` | `ChannelSession[]` | 配对会话记录（channel:account:conversation:sender） |
| `docs/runs/gateway/deliveries.json` | `DeliveryRecord[]` | 投递记录（pending → sent/failed） |

---

## 4. 文档系统：规范长期记忆

### 4.1 设计原则

> `docs/` 是规范长期记忆。Mastra Memory (LibSQL) 是运行时对话连续性。两者服务于不同的目的和时间跨度。

**为什么选择文件支持的文档记忆：**
- 可在 git 中审查和版本控制
- 可纠正（不同于不透明的聊天历史向量）
- 通过结构化卡片以低 token 成本供 AI 阅读
- 不受数据库重置和模型变更的影响

### 4.2 文档目录结构

| 目录 | 用途 | 示例 |
|---|---|---|
| `docs/agents/` | 紧凑智能体卡片（每张 < 2KB） | `OMNI_ROUTER_AGENT.md`, `CODE_AGENT.md`, `CRON_AGENT.md`, `KNOWLEDGE_AGENT.md`, `TASK_AGENT.md` |
| `docs/knowledge/` | 持久化实现知识、已知陷阱 | `TEAM_RUNTIME.md`, `CLAUDE_CODE.md`, `CRON.md`, `MASTRA.md` |
| `docs/memory/` | 规范长期记忆和索引 | `USER.md`, `OMNI.md`, `DECISIONS.md`, `EPISODIC_LOG.md`, `MEMORY_INDEX.json` |
| `docs/context/` | 上下文包组装 | `router-context.md`, `code-agent-context.md` |
| `docs/skills/` | 可复用技能定义 | `code-task.md`, `cron-task.md`, `doc-sync.md`, `memory-maintenance.md` |
| `docs/channels/` | 通道/Gateway 文档 | `README.md`, `HTTP.md`, `ONEBOT.md`, `QQBOT.md`, `SECURITY.md` |
| `docs/schemas/` | 数据形状定义 | `team-task.schema.json`, `cron-job.schema.json`, `memory-card.schema.json` 等 |
| `docs/runs/` | 运行时产物（不入索引） | `team/`, `code-runs/`, `cron-runs/`, `gateway/` |

### 4.3 文档更新风险模型

KnowledgeAgent 将更新分为三个风险等级：

| 风险 | 处理方式 | 存储位置 |
|---|---|---|
| **低** | 直接追加到情景日志 | `memory/EPISODIC_LOG.md` |
| **中** | 创建文档更新提案供审查 | `memory/doc-update-proposals.jsonl` |
| **高** | 创建文档更新提案供审查 | `memory/doc-update-proposals.jsonl` |

### 4.4 文档同步契约

```
代码变更 → 更新智能体卡片 → 更新知识文档 → 更新/添加测试 → 刷新 MEMORY_INDEX.json
```

---

## 5. 上下文工程

### 5.1 核心理念

OmniAgent 被设计为**供未来的 AI 智能体修改**。文档系统中的每一个设计决策都为此用例优化：AI 智能体应能够加载所需的最小上下文，理解系统，进行更改，并更新文档 —— 全程无需扫描整个代码库。

### 5.2 上下文包 —— 面向任务的文档捆绑

上下文包是核心的上下文组装机制。每个包精确捆绑特定任务类型所需的文档：

```
路由与委派:
  agents/OMNI_ROUTER_AGENT.md → agents/TASK_AGENT.md → knowledge/TEAM_RUNTIME.md

代码执行:
  agents/CODE_AGENT.md → knowledge/CLAUDE_CODE.md → agents/TASK_AGENT.md

定时调度:
  agents/CRON_AGENT.md → knowledge/CRON.md → agents/TASK_AGENT.md

文档记忆:
  agents/KNOWLEDGE_AGENT.md → skills/doc-sync.md → skills/memory-maintenance.md

通道网关:
  channels/README.md → channels/SECURITY.md → channels/HTTP.md / ONEBOT.md
  → src/gateway/main.ts → src/gateway/message-handler.ts → src/gateway/delivery.ts
```

---

## 6. 数据流分析

### 6.1 代码任务执行（完整流程）

```
1. 用户/OmniRouterAgent 调用 start-claude-code-task(workspacePath, objective)
2. CodeAgent 验证 workspace ∈ OMNI_ALLOWED_WORKSPACES
3. 如未提供 teamTaskId，自动创建 Team Task → 启动 Team Run
4. 持久化任务快照到 docs/runs/code-runs/tasks.json
5. Windows: 使用 PowerShell 启动 Claude Code（通过临时 prompt 文件传递完整提示）
   Linux/Mac: 直接 spawn('claude', ['-p', prompt])
6. 捕获 stdout/stderr → 追加到 docs/runs/code-runs/{taskId}.jsonl
7. stdout/stderr 同时作为 Team Events 追加
8. 进程关闭时：
   - 成功 (code=0) → completeTeamRun(), 写入 Result, 发送 inbox 通知
   - 失败 (code≠0) → failTeamRun(), 写入 Result, 发送 inbox 通知
   - 进程错误 → failTeamRun()
9. OmniRouterAgent 读取收件箱 → getRunResult(resultRef) → 呈现给用户
```

### 6.2 定时任务执行流程

```
1. CronAgent / 用户创建 cron job → docs/runs/cron-runs/jobs.json
2. startCronScheduler() 每 30s 调用 runDueCronJobs()
3. runDueCronJobs() 扫描 active jobs:
   a. 标准 cron → parseCronSchedule() → nextFireAt() 判断是否到期
   b. 一次性 → parseOneTimeSchedule() → 判断是否达到指定时间且未执行
   c. 每日 → parseDailySchedule() → 判断是否达到今日指定时间且今日未执行
4. 到期 job → executeCronJob() → startClaudeCodeTask()
5. job 记录 lastRunAt, lastRunTaskId, lastRunTeamTaskId, lastRunTeamRunId
6. 一次性 job 执行后自动 pause
```

### 6.3 通道消息执行流程（Omni Gateway）

```
1. 外部消息应用 → POST /message 或 /onebot
2. HTTP 服务解析 → normalizeHttpMessage() / normalizeOneBotMessage()
3. handleChannelMessage():
   a. authorizeMessage() — 配对令牌或允许列表验证
   b. 命令路由:
      - /task <workspace> :: <objective>
        → createTeamTask(source: "channel-gateway", target: "code-agent")
        → startClaudeCodeTask() 持久化任务
      - 自然语言
        → POST /api/agents/omni-router-agent/generate → 同步返回 Router 回复
4. sendOutbound() 推送回复回通道
5. Delivery Worker 轮询 inbox → 推送异步任务结果
```

### 6.4 存储边界

| 数据类型 | 存储位置 | 生命周期 | 访问模式 |
|---|---|---|---|
| 对话记忆 | LibSQL（通过 Mastra Memory）`omni-agent.db` | 运行时会话 | 每轮读写 |
| 智能体卡片 | `docs/agents/*.md` | Git 版本控制 | 上下文组装时读取 |
| 知识文档 | `docs/knowledge/*.md` | Git 版本控制 | 实现时读取 |
| 长期记忆 | `docs/memory/*` | Git 版本控制 | 通过 KnowledgeAgent 读写 |
| 通道文档 | `docs/channels/*.md` | Git 版本控制 | 通道开发时读取 |
| Team Runtime 记录 | `docs/runs/team/*` | 持久化（文件） | 执行时写入，状态查询时读取 |
| 代码任务索引 | `docs/runs/code-runs/tasks.json` | 持久化（文件） | 任务恢复和列表查询 |
| 代码任务日志 | `docs/runs/code-runs/{taskId}.jsonl` | 短暂 | 执行时写入，调试时读取 |
| 定时任务存储 | `docs/runs/cron-runs/jobs.json` | 持久化（文件） | 调度操作时读写 |
| 工具审计 | `docs/runs/gateway/tool-audit.jsonl` | 追加式日志 | 工具调用时写入 |
| Gateway 会话/投递 | `docs/runs/gateway/sessions.json`, `deliveries.json` | 持久化（文件） | 配对/投递时写入 |
| 数据模式 | `docs/schemas/*.json` | Git 版本控制 | 数据形状变更时读取 |

---

## 7. Mastra 原生能力利用

OmniAgent 在 `src/mastra/index.ts` 中启用了以下 Mastra 原生能力：

| 能力 | 配置 | 用途 |
|---|---|---|
| **Background Tasks** | enabled, globalConcurrency=5, perAgentConcurrency=2, defaultTimeoutMs=30min | 长期运行的 Claude Code 任务可标记为后台执行 |
| **Scheduler** | enabled, tickIntervalMs=10s | Mastra 原生调度器（后续迁移目标） |
| **LibSQL Storage** | `file:./omni-agent.db` | 所有 Agent Memory 的持久化后端 |
| **Workflows** | 3 个已注册 | taskOrchestrationWorkflow, runCodeTaskWorkflow, memoryMaintenanceWorkflow |

---

## 8. 安全设计

### 8.1 工作区隔离
- `OMNI_ALLOWED_WORKSPACES` 环境变量定义允许的路径（默认 `L:\Code`）
- `assertAllowedWorkspace()` 在执行前验证所有代码任务路径
- 通过 `normalizeInside()` 进行严格的路径前缀检查，防止路径遍历

### 8.2 通道安全
- `OMNI_GATEWAY_PAIRING_TOKEN` — 一次性配对令牌
- `OMNI_GATEWAY_ALLOW_SENDERS` — 发送者白名单（分号分隔）
- 配对会话持久化，后续消息无需重复配对
- 远程通道输入默认不可信；不应直接暴露到公网

### 8.3 工具审计与脱敏
- Tool Gateway 对所有已迁移的工具调用进行审计
- 敏感字段自动脱敏：匹配 `apiKey`, `token`, `secret`, `password`, `credential`, `authorization`, `cookie` 的 key
- 超过 4000 字符的字符串自动截断
- 审计记录写入 `docs/runs/gateway/tool-audit.jsonl`

### 8.4 密钥管理
- `.env` 通过 `.gitignore` 排除在 git 之外
- 文档系统明确禁止存储密钥、凭据或 API 密钥
- 记忆更新工具被指示过滤敏感数据

---

## 9. 测试覆盖

当前 6 个测试文件，14 个测试用例，全部通过：

| 测试文件 | 覆盖范围 |
|---|---|
| `tests/team-runtime-store.test.ts` | create/start/complete/result/inbox；interrupted 恢复；timeout 标记；retry 创建；queued 取消 |
| `tests/gateway-message-handler.test.ts` | 发送者配对和允许列表；命令路由；task 命令格式验证 |
| `tests/code-task-store.test.ts` | Dry-run 任务持久化；模块重载后状态和列表恢复；teamTaskId/teamRunId 在持久化索引中 |
| `tests/cron-store.test.ts` | 标准 cron 表达式 next-run 计算；daily schedule next-run 兼容性 |
| `tests/docs-memory.test.ts` | 用户档案事实 upsert 到 USER.md |
| `tests/tool-gateway.test.ts` | 成功调用审计和敏感字段脱敏；失败调用在 rethrow 前审计 |

**运行命令：** `npm test` (vitest run), `npm run typecheck` (tsc --noEmit), `npm run build`

---

## 10. 扩展点

1. **LibSQL 迁移：** `teamRuntimeStoreBackend.kind` 从 `'file'` 切换到 `'libsql'` — Team Runtime 记录从 JSON 文件迁移到 LibSQL
2. **Tool Gateway 扩展：** 将 code/cron/team-runtime tools 的 execute 包装迁移到 `executeWithToolGateway`；添加 sandbox 策略钩子
3. **Scheduler 迁移：** 将自定义 `cron-store` 轮询替换为 Mastra `WorkflowScheduler`；schedules 存储到 Mastra schedule storage
4. **新智能体：** 注册 Mastra Agent，添加智能体卡片，实现 Team Runtime 协议
5. **新工作流：** 添加到 `src/mastra/workflows/`，在 `src/mastra/index.ts` 中注册
6. **新通道适配器：** 实现 `ChannelMessage`/`OutboundMessage` 类型，添加端点和出站推送
7. **向量索引：** `docs/` 结构已为未来基于嵌入的搜索做好准备
8. **Task Orchestration 完善：** 扩展为 intake → plan → confirmation → apply → test → result 多阶段 workflow；使用 workflow `suspend/resume` 处理 `waiting_user_confirm`

---

## 11. 关键设计决策

| 决策 | 理由 |
|---|---|
| docs 作为规范记忆，而非数据库 | 可审查、可版本控制、可被人和 AI 纠正 |
| Runtime facade 层作为迁移边界 | 统一接口，委托给现有模块，为 Mastra 原生迁移提供清晰路径 |
| Tool Gateway 包装所有工具调用 | 集中的风险管控、审计记录、敏感数据脱敏 |
| 先文件存储再迁移 Team Runtime | 在存储迁移之前确保协议稳定 |
| OmniRouterAgent 作为 Supervisor | 通过 Mastra 原生 sub-agents/workflows 委派，工具直接使用作为兼容路径 |
| Gateway 作为独立进程 | 通道层与 Mastra 服务解耦，独立部署和重启 |
| Gateway 仅是任务来源和投递层 | 所有执行流经 Team Runtime，避免碎片化 |
| ChannelMessage 统一消息类型 | 不同通道的消息协议差异封装在 normalize 层 |
| Durable code task index (`tasks.json`) | 确保进程重启后任务状态可恢复 |
| 智能体卡片每张 < 2KB | 最小化 AI 消费者的上下文 token 消耗 |
| Inbox 用于通知，Result 用于数据 | 关注点分离；收件箱是指针，结果是载荷 |
| Cron 仅为任务来源 | 所有智能体使用统一的结果协议；避免碎片化 |
| 上下文包而非组件文档 | 面向任务的组装匹配 AI 智能体的实际工作方式 |
| 陷阱作为一等知识 | 防止 AI 智能体重新发现已知问题 |

---

*2026-05-12 基于完整代码仓分析更新。反映 Round 1-4 代码实现（Runtime facade, Tool Gateway, Task Orchestration, Cron 升级, Code Task 持久化）。*
