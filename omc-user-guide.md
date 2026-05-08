# oh-my-claudecode 用户使用手册

本文基于 `Yeachan-Heo/oh-my-claudecode` 当前公开文档整理，面向想用 OMC 进行 AI Coding 的用户，重点覆盖安装、规划、执行、验证、循环、团队协作、监控与故障处理。

参考来源：

- 项目 README：https://github.com/Yeachan-Heo/oh-my-claudecode
- Reference：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/docs/REFERENCE.md
- Architecture：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/docs/ARCHITECTURE.md
- Team Skill：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/skills/team/SKILL.md
- Autopilot Skill：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/skills/autopilot/SKILL.md
- Ralph Skill：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/skills/ralph/SKILL.md
- Ultrawork Skill：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/skills/ultrawork/SKILL.md
- Deep Interview Skill：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/skills/deep-interview/SKILL.md
- Ralplan Skill：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/skills/ralplan/SKILL.md
- Performance Monitoring：https://github.com/Yeachan-Heo/oh-my-claudecode/blob/main/docs/PERFORMANCE-MONITORING.md

## 1. OMC 是什么

oh-my-claudecode，简称 OMC，是 Claude Code 的多智能体编排层。它不是一个普通代码生成脚本，而是一套围绕 Claude Code 构建的工作流系统。

它的核心能力包括：

- 根据用户输入自动识别工作模式。
- 调用不同专长的 Agent：探索、分析、规划、架构、执行、调试、测试、评审、安全、文档等。
- 把复杂 Coding 任务拆成阶段、子任务、验收标准和修复循环。
- 支持 Team、Autopilot、Ralph、Ultrawork、Deep Interview、Ralplan 等技能。
- 用状态文件保存进度，支持中断后恢复。
- 用 HUD、Replay、Session Summary 观察 Agent、成本、工具耗时和并行效率。

OMC 的执行链路可以理解为：

```text
用户输入
  -> Hooks 检测关键词和生命周期事件
  -> Skills 注入工作流行为
  -> Agents 执行专业任务
  -> State 保存进度、循环和恢复信息
```

## 2. 安装与初始化

推荐使用 Claude Code 插件方式安装。

在 Claude Code 会话中逐行执行：

```bash
/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode
```

然后：

```bash
/plugin install oh-my-claudecode
```

初始化：

```bash
/setup
```

或：

```bash
/omc-setup
```

如果使用 npm runtime 路径，包名不是仓库名，而是：

```bash
npm i -g oh-my-claude-sisyphus@latest
omc setup
```

基础要求：

- 已安装 Claude Code CLI。
- Claude Max/Pro 订阅，或配置 `ANTHROPIC_API_KEY`。
- Team CLI worker、限流等待等功能需要 `tmux`。Windows 可用 `psmux`，或在 WSL2 内安装 `tmux`。

## 3. 你应该选择哪个模式

### 常用模式总览

| 场景 | 推荐模式 | 说明 |
|---|---|---|
| 想直接从想法做到可运行代码 | `/autopilot` | 全生命周期自动执行 |
| 需求还模糊，不想 AI 误解 | `/deep-interview` | 苏格拉底式澄清需求 |
| 需要先出高质量方案 | `/ralplan` | Planner → Architect → Critic 共识规划 |
| 必须做到完成且验证通过 | `/ralph` | PRD 驱动的持久循环 |
| 大任务要多 Agent 并行 | `/team` | 官方推荐的团队编排入口 |
| 有多个独立任务想并发做 | `/ultrawork` | 只提供并行执行，不提供持久完成保证 |
| 想让 Codex/Gemini 参与建议 | `/ccg` 或 `omc ask` | 多模型顾问式分析 |

### 推荐完整 Coding 流程

需求不清晰时：

```text
/deep-interview
  -> /ralplan
    -> /autopilot
```

这条链路对应三道质量门：

```text
Deep Interview：需求是否足够清晰
Ralplan：方案是否架构合理
Autopilot：代码是否实现并验证通过
```

需求已经明确时：

```text
/team 3:executor "实现 X，并通过测试"
```

或：

```text
/ralph "修复 src/auth/login.ts 的空指针问题，并补测试"
```

## 4. Coding 规划：Deep Interview 与 Ralplan

### 4.1 Deep Interview：先把需求问清楚

适合场景：

- 你只有一个模糊想法。
- 你担心 AI 做出来不是你想要的。
- 需求涉及产品、业务、边界、验收标准。
- 需要在动手前暴露隐藏假设。

用法：

```bash
/deep-interview "我想做一个团队任务管理工具"
```

工作方式：

1. 判断是新项目还是已有项目改造。
2. 如果是已有项目，先调用 `explore` Agent 探查代码，而不是让你重复说明代码事实。
3. 每轮只问一个问题。
4. 每轮识别最薄弱的清晰度维度。
5. 对目标、约束、成功标准、上下文进行打分。
6. 用歧义度门槛控制是否能进入执行。
7. 输出规格文档到：

```text
.omc/specs/deep-interview-{slug}.md
```

核心原则：

- 一次只问一个问题。
- 不问代码里已经能查到的问题。
- 每轮显示歧义度。
- 歧义度低于阈值后才建议执行。
- 可以提前退出，但会提示风险。

### 4.2 Ralplan：让 Planner、Architect、Critic 达成共识

适合场景：

- 你要先要方案，不想马上写代码。
- 任务涉及架构、安全、迁移、公共 API、生产风险。
- 你想避免“计划看起来可以，但其实不可执行”。

用法：

```bash
/ralplan "为现有系统增加 OAuth 登录"
```

等价于：

```bash
/oh-my-claudecode:omc-plan --consensus
```

共识流程：

```text
Planner 生成方案
  -> Architect 审查架构合理性
    -> Critic 审查质量、风险、验收标准
      -> 不通过则回到 Planner 修改
```

最大循环通常到 5 次。最终计划应包含：

- 决策原则。
- 关键驱动因素。
- 至少两个可选方案及取舍。
- ADR：决策、驱动、备选、选择理由、后果、后续事项。
- 可测试验收标准。
- 具体执行步骤。

交互模式：

```bash
/ralplan --interactive "重构支付模块"
```

高风险任务可强制 deliberate 模式：

```bash
/ralplan --deliberate "迁移用户表并保持线上兼容"
```

## 5. Coding 执行：Autopilot、Ralph、Team、Ultrawork

### 5.1 Autopilot：从想法到可运行代码

适合场景：

- “帮我做完整功能”。
- “build me ...”
- “I want a ...”
- 希望系统自动完成需求分析、设计、编码、测试和验证。

用法：

```bash
/autopilot "build a REST API for bookstore inventory with CRUD using TypeScript"
```

Autopilot 的阶段：

```text
Phase 0 Expansion：把想法扩展成详细规格
Phase 1 Planning：生成实现计划
Phase 2 Execution：用 Ralph + Ultrawork 执行
Phase 3 QA：构建、lint、测试、修复循环
Phase 4 Validation：架构、安全、代码质量多角度验证
Phase 5 Cleanup：清理状态文件并总结
```

输出文件：

```text
.omc/autopilot/spec.md
.omc/plans/autopilot-impl.md
```

QA 规则：

- QA 最多循环 5 次。
- 同一个错误连续出现 3 次，应停止并报告根因。
- 验证不通过时修复后重新验证。
- 所有验证者通过后才算完成。

最适合 Autopilot 的输入：

```text
autopilot build me a CLI habit tracker with streak counting using Node.js
```

不适合 Autopilot 的输入：

```text
fix the login bug
```

这种单点修复更适合 Ralph 或直接 executor。

### 5.2 Ralph：做到验收通过才停止

Ralph 是 PRD 驱动的持久循环。它适合“必须完成”的任务。

用法：

```bash
/ralph "修复所有 TypeScript 编译错误，并保证 npm run build 通过"
```

Ralph 会维护或生成 PRD：

```text
.omc/state/sessions/{sessionId}/prd.json
```

Ralph 的循环：

```text
初始化/细化 PRD
  -> 选择下一个未通过 user story
    -> 实现
      -> 逐条验收标准验证
        -> 标记 passes: true
          -> 所有 story 通过？
            -> Reviewer 验证
              -> AI slop 清理
                -> 回归验证
                  -> cancel 清理状态
```

关键要求：

- PRD 里的验收标准必须是具体的，不能保留“实现完成”“代码能编译”这类空泛标准。
- 每个 story 必须用新鲜证据验证。
- 不能缩减范围。
- 不能删测试来制造通过。
- 默认需要 Architect 或指定 reviewer 验证。
- 审核通过后还要运行 `ai-slop-cleaner`，再做回归验证。

可选参数：

```bash
/ralph --no-deslop "..."
/ralph --critic=architect "..."
/ralph --critic=critic "..."
/ralph --critic=codex "..."
```

完成检查清单：

- `prd.json` 所有 story 都是 `passes: true`。
- 所有验收标准已用测试、构建、lint 或手动证据验证。
- TODO 没有 pending 或 in_progress。
- reviewer 验证通过。
- deslop 后回归测试通过。
- 运行 cancel 清理状态。

### 5.3 Team：推荐的多 Agent 协作模式

Team 是 OMC 当前推荐的多 Agent 编排入口。

用法：

```bash
/team 3:executor "fix all TypeScript errors"
```

也可以自动决定 Agent 数量：

```bash
/team "refactor auth module with tests and security review"
```

带 Ralph 持久循环：

```bash
/team ralph "build a complete REST API for user management"
```

Team 标准流水线：

```text
team-plan
  -> team-prd
    -> team-exec
      -> team-verify
        -> team-fix
          -> team-exec
            -> team-verify
```

也就是：

```text
规划 -> 需求/验收 -> 执行 -> 验证 -> 修复循环
```

各阶段路由：

| 阶段 | 必需 Agent | 可选 Agent | 目标 |
|---|---|---|---|
| team-plan | explore, planner | analyst, architect | 探查代码并拆任务 |
| team-prd | analyst | critic | 明确范围和验收 |
| team-exec | executor | debugger, designer, writer, test-engineer | 实现 |
| team-verify | verifier | test-engineer, security-reviewer, code-reviewer | 验证 |
| team-fix | executor | debugger | 修复验证问题 |

停止条件：

- 验证通过且没有剩余修复任务。
- 或达到明确的 blocked/failed 终态并附带证据。
- fix 循环有最大次数，避免无限循环。

Team 的上下文交接：

每个阶段结束前必须写 handoff 文件：

```text
.omc/handoffs/*.md
```

内容包括：

- 已决定事项。
- 被拒绝的备选方案。
- 风险。
- 涉及文件。
- 下一阶段剩余事项。

### 5.4 Team 中的任务拆分和执行

Team Lead 会：

1. 解析用户输入，得到 Agent 数量、类型和任务。
2. 用 `explore` 或 `architect` 分析代码。
3. 拆出文件级或模块级子任务。
4. 建立任务依赖。
5. 创建 Team。
6. 创建子任务。
7. 预分配 owner，避免抢任务。
8. 并行启动 teammate。
9. 监控 TaskList 和 SendMessage。
10. 验证、修复、关闭团队。

Worker 协议：

```text
CLAIM：领取分配给自己的 pending 任务
WORK：直接完成任务，不再生成子 Agent
COMPLETE：标记任务完成
REPORT：通过 SendMessage 报告 lead
NEXT：继续下一个任务
SHUTDOWN：收到 shutdown_request 后确认退出
```

Worker 禁止事项：

- 不要再 spawn 子 Agent。
- 不要运行 Team/Ultrawork/Autopilot/Ralph 等编排命令。
- 不要执行 tmux 编排命令。
- 必须用绝对路径。
- 必须通过 SendMessage 向 lead 报告进度。

### 5.5 Ultrawork：并行执行层

Ultrawork 是并行执行引擎，不是完整的持久工作流。

用法：

```bash
/ultrawork "implement user authentication with OAuth"
```

适合：

- 多个独立任务可并发。
- 你想要高吞吐并行执行。
- 不需要 Ralph 那种“必须完成并验证”的保证。

不适合：

- 需要严格完成保证，使用 Ralph。
- 需要完整生命周期，使用 Autopilot。
- 单一顺序任务，直接用 executor。

Ultrawork 流程：

```text
确认意图
  -> 并行收集上下文
    -> 识别独立任务和依赖任务
      -> 创建任务图
        -> 按模型等级路由
          -> 并行发射独立任务
            -> 顺序执行依赖任务
              -> 轻量验证
```

模型层级：

| 任务类型 | 模型 |
|---|---|
| 简单查找、小改动 | Haiku |
| 标准实现、调试、测试 | Sonnet |
| 架构、复杂重构、深度分析 | Opus |

## 6. Agent 体系

OMC 通过专门 Agent 分工完成 Coding 生命周期。

常用 Agent：

| Agent | 角色 |
---|---|
| `explore` | 快速探索代码、文件、符号 |
| `analyst` | 分析需求、隐藏约束、验收标准 |
| `planner` | 拆任务、排序、产出计划 |
| `architect` | 架构设计、接口边界、取舍 |
| `debugger` | 根因分析、构建错误、回归定位 |
| `executor` | 编码、重构、实现 |
| `verifier` | 验证完成度、测试充分性 |
| `test-engineer` | 测试策略、覆盖率、稳定性 |
| `security-reviewer` | 安全漏洞、信任边界、认证授权 |
| `code-reviewer` | 代码审查、兼容性、质量 |
| `designer` | UI/UX 设计与前端交互 |
| `writer` | 文档、迁移说明、用户指南 |
| `git-master` | Git 操作、提交、rebase |
| `document-specialist` | 外部文档/API/SDK 查询 |
| `critic` | 挑战计划和设计，找缺口 |

典型链路：

```text
explore -> analyst -> planner -> critic -> executor -> verifier
```

## 7. 外部 CLI Worker：Codex 与 Gemini

OMC 支持 `omc team` 通过 tmux 启动外部 CLI worker：

```bash
omc team 2:codex "review auth module for security issues"
omc team 2:gemini "redesign UI components for accessibility"
omc team 1:claude "implement payment flow"
```

适合：

| Worker | 适合任务 |
---|---|
| Codex CLI | 架构审查、安全分析、代码审查、重构 |
| Gemini CLI | UI/UX、文档、大上下文任务 |
| Claude Agent | 需要 Claude Code 工具和团队通信的迭代任务 |

注意：

- Codex/Gemini CLI worker 是一次性 tmux 任务。
- 它们没有 Claude Team 的 TaskList/SendMessage 能力。
- Lead 负责创建 prompt 文件、启动 worker、读取输出、更新任务状态。

## 8. 验证、修复与循环

### 8.1 验证层级

轻量验证：

- build/typecheck 通过。
- 受影响测试通过。
- 没有新增错误。
- 手动 QA 完成。

Ralph/Autopilot/Team 验证：

- 验收标准逐条验证。
- Verifier 必跑。
- 安全相关必须加 security-reviewer。
- 大范围或架构变更必须加 code-reviewer。
- 验证失败进入 fix loop。

### 8.2 循环策略

| 模式 | 循环类型 |
---|---|
| Ralph | story -> verify -> reviewer -> fix -> reverify |
| Autopilot | QA cycle + validation re-run |
| Team | team-exec -> team-verify -> team-fix |
| Deep Interview | question -> score -> next weakest dimension |
| Ralplan | Planner -> Architect -> Critic -> revise |

### 8.3 何时停止

应该停止：

- 所有验收标准通过。
- 测试、构建、lint/typecheck 有新鲜通过证据。
- reviewer 明确批准。
- 状态清理完成。

应该报告阻塞：

- 缺失凭据、外部服务不可用。
- 需求仍存在根本冲突。
- 同一错误连续多轮出现。
- 超过最大修复/验证轮数。

不应该停止：

- “看起来可以”但没有验证证据。
- Architect 批准后还没 deslop 和回归。
- 只有部分 user story 完成。
- 还有 pending/in_progress TODO。

## 9. 状态、恢复与取消

OMC 会把状态写到 `.omc/` 下。

常见路径：

```text
.omc/state/
.omc/handoffs/
.omc/specs/
.omc/plans/
.omc/sessions/
.omc/artifacts/
```

恢复：

- Autopilot 中断后再次运行 `/autopilot`，会尝试从状态恢复。
- Deep Interview 中断后再次运行 `/deep-interview`，会读取 interview state。
- Team 通过 state 和 handoff 恢复阶段上下文。

取消：

```bash
/cancel
```

或：

```bash
/oh-my-claudecode:cancel
```

Team 取消时会：

1. 请求 teammate shutdown。
2. 等待确认。
3. 标记 phase 为 cancelled。
4. 删除团队资源。
5. 保留 handoff 以便恢复。

## 10. 监控与可观测性

### 10.1 HUD

HUD 可显示：

- 当前模式。
- 活跃 Agent。
- TODO/PRD 进度。
- 上下文窗口使用率。
- 限流状态。
- token/cost 估计。

配置示例：

```json
{
  "omcHud": {
    "preset": "focused"
  }
}
```

### 10.2 Agent Observatory

可观察：

- Agent 生命周期。
- 工具调用次数。
- token 估计。
- 成本估计。
- 文件触碰数量。
- 慢工具瓶颈。
- 并行效率。

典型输出：

```text
Agent Observatory (3 active, 85% efficiency)
[a1b2c3d] executor 45s tools:12 tokens:8k $0.15 files:3
```

### 10.3 Session Replay

Replay 文件：

```text
.omc/state/agent-replay-{sessionId}.jsonl
```

事件类型：

- `agent_start`
- `agent_stop`
- `tool_start`
- `tool_end`
- `file_touch`
- `intervention`

查看命令：

```bash
tail -20 .omc/state/agent-replay-*.jsonl
ls .omc/sessions/*.json
```

## 11. 常见 Coding 用法模板

### 11.1 从模糊想法到上线级实现

```bash
/deep-interview "我想做一个可以给销售团队使用的客户跟进系统"
```

完成访谈后选择：

```text
Ralplan -> Autopilot
```

### 11.2 明确功能，直接自动实现

```bash
/autopilot "build a task CRUD REST API using TypeScript, Express, PostgreSQL, and Jest tests"
```

### 11.3 必须修完并验证

```bash
/ralph "fix all failing tests from npm test and ensure npm run build passes"
```

### 11.4 多人并行修复

```bash
/team 5:executor "fix all TypeScript errors across src and add missing tests"
```

### 11.5 安全敏感改动

```bash
/team ralph "refactor authentication middleware and include security review"
```

### 11.6 先做高质量计划

```bash
/ralplan --deliberate "migrate billing from Stripe legacy API to current API without breaking existing customers"
```

### 11.7 大范围并行但不需要持久循环

```bash
/ultrawork "update docs, examples, and type exports for the new SDK API"
```

### 11.8 让 Codex/Gemini 参与顾问审查

```bash
/ccg review this PR: backend architecture by Codex, UI consistency by Gemini
```

或终端：

```bash
omc ask codex "identify architecture risks in this migration"
omc ask gemini "review UI accessibility and consistency"
```

## 12. 最佳实践

### 提示词写法

好提示：

```text
/ralph fix src/hooks/useSession.ts null check bug.
Acceptance criteria:
1. Anonymous users do not crash the dashboard.
2. npm test -- useSession passes.
3. npm run build passes.
```

差提示：

```text
fix this
```

如果你不知道怎么描述，先用：

```bash
/deep-interview
```

### 任务拆分原则

- 文件级或模块级拆分，减少冲突。
- 明确依赖关系。
- 每个子任务都有验收标准。
- 独立任务并行，有依赖的任务顺序执行。
- 长时间命令后台运行。
- 快速读文件、查状态前台执行。

### 模型使用原则

- Haiku：查找、简单文档、小变更。
- Sonnet：标准编码、测试、调试。
- Opus：架构、复杂重构、关键评审。

### 验证原则

- 不接受“应该可以”。
- 必须有新鲜命令输出或明确证据。
- 修复后重新跑相关测试。
- 大改动必须跑 reviewer。
- 安全/认证/权限相关必须跑 security-reviewer。

## 13. 常见问题

### Team 和 Ultrawork 有什么区别

Team 是完整协调系统，有任务、owner、消息、阶段、验证、修复循环。Ultrawork 只是并行执行协议，没有持久状态和完整完成保证。

### Ralph 和 Autopilot 有什么区别

Ralph 适合已有明确任务，循环做到验收通过。Autopilot 适合从一个产品想法出发，自动完成扩展需求、规划、实现、QA 和验证。

### Ralplan 和 Deep Interview 有什么区别

Deep Interview 解决“需求清不清楚”。Ralplan 解决“方案好不好、能不能执行”。两者可以串联。

### 为什么 OMC 会把我的 vague prompt 重定向到 Ralplan

因为 heavy execution 模式启动成本高。如果输入像“add auth”“make it better”这种没有文件、函数、验收标准的请求，OMC 会先要求规划，避免多 Agent 在模糊目标上浪费循环。

### 如何强制跳过规划

可以用：

```text
force: ralph ...
```

或：

```text
! ralph ...
```

但不建议对高风险任务这样做。

### 如何排查慢 Agent

查看 HUD、Agent Observatory 或：

```bash
tail -20 .omc/state/agent-replay-*.jsonl
```

如果并行效率低，检查是否任务拆分过粗、文件冲突、Agent 卡住或工具调用太慢。

## 14. 推荐工作流速查

```text
模糊想法：
  deep-interview -> ralplan -> autopilot

明确大功能：
  team 3:executor -> verify -> fix loop

明确且必须完成：
  ralph -> PRD stories -> verify -> reviewer -> deslop -> regression

多独立任务：
  ultrawork -> parallel waves -> lightweight verify

高风险方案：
  ralplan --deliberate -> team/ralph

代码审查：
  code-reviewer / security-reviewer / ccg / omc ask codex

UI/文档：
  designer / writer / gemini worker
```

## 15. 最小上手路径

如果你只想马上用起来，按这个顺序：

1. 安装插件并运行 `/setup`。
2. 对模糊需求用 `/deep-interview`。
3. 对明确功能用 `/team 3:executor "..."`。
4. 对必须完成的修复用 `/ralph "..."`。
5. 对完整项目功能用 `/autopilot "..."`。
6. 完成后看测试、构建、验证结果，不只看总结。
7. 出问题时看 `.omc/state/agent-replay-*.jsonl` 和 HUD。

