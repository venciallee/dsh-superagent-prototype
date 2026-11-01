# dsh-superagent 源码精读：从模块细节到整体架构

> 基于 `dsh-superagent-prototype` 仓库 commit `6337fc8`，对照 DeepSeek Harness 主仓库 commit `ce7073d5`。2026-11-01。
> 姊妹篇《dsh-superagent 深度拆解》（`dsh-superagent-explainer.md`）讲概念、谱系，以及它和 DeepSeek Harness、ReAct、Agent Team 的对比。本篇只讲代码：逐模块精读，再从代码反推架构。

一个常驻在团队 GitHub 和飞书里的 agent，最容易出问题的不是“模型会不会写代码”，而是这几件事：消息会不会丢或重复，权限会不会被一段评论改写，重启后正在做的事能不能接上，人的注意力会不会被刷屏。这个仓库约 1.5 万行源码（另有约 1.5 万行测试），大部分代码都在解决这几件事。本篇按“先高层、再模块、再流程、最后取舍”的顺序展开。读完你应该能说清三点：每个模块的职责和它守住的不变量；一条消息在代码里经过哪些函数、哪些事务；作者在哪些地方选择了不显然的做法，代价是什么。

**怎么读：**

| 部分 | 内容 | 适合谁 |
|---|---|---|
| 第一部分：高层架构（§1–§6） | 系统形状、代码地图、端口与适配器、进程里的几个循环、五个核心数据结构、全系统伪代码 | 所有人，15 分钟 |
| 第二部分：模块精读（§7–§14） | 逐模块读源码：职责、数据结构、关键代码、不变量、亮点与取舍 | 想改或想抄这套设计的工程师 |
| 第三部分：流程精讲（§15–§20） | 六条端到端流程，每一步落到函数和事务 | 想理解“为什么不丢不重”的人 |
| 第四部分：亮点、取舍与风险（§21–§24） | 设计决策矩阵、可疑点、可借鉴清单 | 做架构决策的人 |

---

# 第一部分　高层架构

## 1. 系统在做什么

**一句话：**外部消息先过一道纯代码的权限门，再由一个专职分诊的 LLM 会话（主 Session）决定是否开成工单；每个工单交给一个独立的 LLM 会话（执行器 Session），在独立的 git worktree 里完成，结果回到提需求的原对话。

![图 1：系统形状](img/l0-system.svg)

**图 1：LLM 只出现在两处，即分诊（主 Session）和执行（每工单一个执行器 Session）。谁能触发、谁被授权、工单处于什么状态，全由确定性内核和 SQLite 决定；所有离开内核的箭头都是虚线：先落盘，再异步执行。**

全篇用四个词组织：**触发**（谁唤醒 agent）、**授权**（凭什么允许它动手）、**状态**（“谁在做什么”存在哪）、**执行**（谁动手、在哪动手）。这是本文的组织方式，不是仓库的官方分类。领域名词（Source、Trigger、Thread、Eval、Mandate、Ticket、Subscription、Assignment、Channel）的定义见姊妹篇 §1.1，或仓库的 `docs/001-domain.md`。下文第一次用到时也会给一句解释。

## 2. 代码地图

| 目录 | 行数 | 职责 | 对应的轴 |
|---|---|---|---|
| `src/core/` | 2,088 | 领域核心：Engine 及其 5 个部件、领域类型、阶段分类、确定性 id、错误与原因码。**不做任何 I/O** | 状态 |
| `src/policy.ts` 等根目录文件 | 544 | 配置 schema、Gate、Mandate 重验证、研判结果校验、策略版本 | 授权 |
| `src/app/`（不含 tools、views） | 4,230 | 应用服务：组合根 Runtime、Store、Effects、Outbox、Passes、Sessions、MasterAssessor、Settings、Prompts、资源与卡片 | 状态、授权 |
| `src/app/tools/` | 441 | 两类模型的工具面：主 Session 14 个，执行器 6 个 | 授权、执行 |
| `src/executor/` | 838 | DshExecutor：一工单一会话一目录、工位、后台工作、重试与恢复 | 执行 |
| `src/sources/` | 2,059 | Source 插件：GitHub、飞书、注册表、轮询公共逻辑、研判上下文 | 触发 |
| `src/infra/` | 347 | CLI 传输、进程环境与命令 shim、健康板、文本工具 | 执行 |
| `src/app/views/`、`src/app/http.ts` | 868 | 管理 API 与面板视图 | — |
| `src/client/` | 4,037 | Web 面板（React，注入 DSH 侧栏） | — |
| `prompts/`、`skills/` | 438 行提示词 | 内置提示词与 GitHub、飞书 skill | 授权、执行 |
| `tests/` | ~15,000 | 59 个测试文件 | — |

**依赖规则**（`docs/007-architecture.md` 写明，并由测试守护）：

- `src/core` 除了 Node 的哈希函数，不引入任何外部模块，也不做 I/O。
- `src/sources` 不引入 `src/app`。Source 只能通过 `SourceContext` 这个窄接口和宿主打交道。
- 面板只引入宿主侧的**类型**，以及少数纯值模块（准入表选项、限值、错误码等）。
- **每种状态只有一个写者**：领域对象只由 Engine 写，读位置只由各自的 Source 写，配置只由 Settings 写，effect 行只由 Effects 写，outbox 回执只由 Outbox 写，每个健康条目只由认领它的部件写。

## 3. 端口与适配器：Engine 如何与外界隔离

如果你熟悉“六边形架构”（端口与适配器），可以直接套用：领域核心在中间，只通过端口与外界交互；适配器分成两侧，一侧驱动核心（调用它），一侧被核心驱动（实现它的端口）。这个类比基本贴切。唯一的差别是：被驱动侧的适配器在核心的事务里**只能排队**，不能真的执行 I/O。

![图 2：代码架构](img/d1-hexagon.svg)

**图 2：`src/core` 只通过 6 个端口与外界交互。左侧驱动方按角色拿到 Engine 的一个子集接口；右侧被驱动方实现端口，其中会产生外部动作的三个端口只往持久化的 Effects 队列里写。真正的 I/O 发生在事务提交之后。**

**驱动侧：按角色裁剪的接口**（`src/core/engine.ts:264-270`）：

| 角色类型 | 包含的方法 | 谁拿着它 |
|---|---|---|
| `EngineForSources` | `receive`、`bindThread`、`threadClosed` | Source 插件，经 `SourceContext` 转交 |
| `EngineForMainSession` | `candidate`、`candidateDelivered`、`decide`、`attach`、`reply`、`requestStatus`、`note` | 主 Session 的工具、MasterAssessor |
| `EngineForPeople` | `stopTicket`、`retryTicket`、`reevaluate`、`unsubscribe`、`dismiss` | 管理 API（面板）、部分主 Session 工具 |
| `EngineForSettings` | `policyChanged` | Settings |
| `EngineForExecutor` | `assignmentStarted`、`inputsDelivered`、`inputsRead`、`turnEnded`、`deliveryFailed`、`assignmentInterrupted`、`report`、`markUnrelated`、`handback`、`followForContext`、`reply` | DshExecutor、执行器工具 |

这样做的效果是：Source 在**类型层面**就调不到 `decide()`，执行器调不到 `stopTicket()`。权限边界不靠约定，靠编译器。

**被驱动侧：6 个端口**（`src/core/ports.ts`）：

| 端口 | 实现 | 语义要点 |
|---|---|---|
| `Repositories` | `Store`（SQLite） | 所有方法都是同步的，`atomic(work)` 包住一个工作单元 |
| `Policy` | `settingsPolicy(settings, scope)` | **每次读取都新建一个无状态对象**，所以不会读到过期配置 |
| `SourceScope` | `SourceRegistry` | 只回答一个问题：Source 是否在听这个会话、是否“全收”、属于准入表哪一行 |
| `Assessor` | `MasterAssessor` | `request(candidate)` 只排一个 `assessment` effect；结果通过 `Engine.decide` 回来 |
| `Executor` | `DshExecutor` | `start`、`deliver` 只排 `launch`、`delivery` effect；`stop` 挂在 `afterCommit` 上 |
| `Outbox` | `Outbox` | `send` 写回执并排 `reply` effect；`status`、`withdraw` 查询和撤回 |

还有 3 个可选钩子：`prepareTicket`（由 DshExecutor 在建工单时算出短名、目录和派工文本）、`now`（测试用的时钟）、`emit`（领域事件，Runtime 把其中几种转成给主 Session 的通知）。

**命名约定也是协议：**请求用祈使句（`stopTicket`、`bindThread`），可能被拒绝，抛 `UserError`；事实用过去时（`assignmentStarted`、`deliveryFailed`），记录已经发生的事，通常不会被拒绝，过期的事实会被静默丢弃。

## 4. 一个进程里同时在跑的几件事

整个系统运行在一个 DSH 进程里（一个专用 profile）。理解它的关键，是看清有几个“循环”在并发推进状态，以及它们之间只通过什么交接：

| 循环 | 节奏 | 做什么 | 只通过什么影响领域状态 |
|---|---|---|---|
| GitHub Source 轮询 | 至少 60 秒，遵守 GitHub 返回的 `X-Poll-Interval` | 读 notifications、订阅线程、PR 与 CI | `engine.receive`、`bindThread`、`threadClosed` |
| 飞书 Source | 实时事件 + 历史补偿 + 话题轮询 | 读群消息与话题 | 同上 |
| Passes | 每 2 秒一轮，有新事件或命令时立即补一轮，同一时刻只跑一轮 | 先踢一下 Effects，再 `engine.evaluateDue()` | `evaluateDue` |
| Effects 分发器 | 入队时用 microtask 立即调度，到期行用定时器 | 6 种副作用：assessment、launch、delivery、relay、notice、reply | 适配器回调 Engine 的“事实”方法 |
| 主 Session 的 agent loop | 有消息才跑，空闲即结束 turn | 分诊、研判、转达 | 14 个工具 → `decide`、`attach`、`reply`…… |
| N 个执行器 Session 的 agent loop | 有输入才跑 | 写代码、开 PR | 6 个工具 → `report`、`handback`……；DSH 事件 → `turnEnded`、`inputsRead` |
| 管理 API 与面板 | 面板轮询 | 展示、停止、重试、改配置 | `EngineForPeople`、Settings |

**所有循环都只能通过 Engine 的同步方法改领域状态。**Engine 的每个方法包在一个 SQLite 事务里，进程又是单线程的 Node，所以领域状态上不存在并发写冲突。并发只存在于 I/O 一侧（Effects 的多个 handler、多个执行器会话），而那一侧的结果只能以“事实”的形式回到 Engine。

## 5. 五个核心数据结构

源码里所有规则都围绕这五个类型（`src/core/model.ts`）。逐个看字段，就能看到大部分设计决策。

**Trigger 与它的去向记录（Disposition）**

```ts
interface Trigger {
  id: string                 // Source 给的稳定 id：同 id 只存一次，接单天然幂等
  threads: ThreadRef[]       // threads[0] 是主 Thread，Eval 在它上面跑；其余只提供关联上下文
  actor: ActorRef            // Source 自己的身份体系，跨 Source 不做映射
  human: boolean             // Source 声明：人写的（不是 bot、CI 或 agent 自己）
  addressed?: 'mention' | 'direct'   // Source 声明：@ 了机器人，或是私聊
  channel?: ChannelRef       // 回复往哪写
  meta: Record<string, unknown>      // Source 私有；核心层从不解释
}
interface TriggerDisposition {   // “这条消息后来去了哪”，只用于审计和裁决归属
  filtered?: Reason              // 被 Source 范围过滤（仍然存下来，可以“重新评估”）
  evalId?, decisionId?           // 进了哪个 Eval，得到哪个 Decision
  fanout: string[]               // 扇出到了哪些工单
  context?: string[]             // 其中哪些只是“为上下文而关注”，不归它们裁决
  requeued?: true                // 人或主 Session 把它重新送去评估
  handbacks?: { ticketId, reason, at }[]   // 哪些工单交回过它，理由是什么
  acked?: string[]               // 哪些工单的执行器确认或回答过它（视为已接手）
}
```

去向记录（disposition）是“后续消息归谁裁决”这一规则的唯一依据（见 §7.8 的 `followupOwners`）。它把“一条消息被谁处理过”变成可查询的事实，而不是散落在各个会话的上下文里。

**Thread 与 Evaluation**

```ts
interface Thread {
  subscriptions: string[]    // 订阅它的工单：新消息会扇出给其中的活工单
  context?: string[]         // 其中只为上下文而关注的工单：收消息，但不获得裁决权，也不会被复活
  correlated: string[]       // 与它共享过 Trigger 的线程：只加上下文，不参与路由
  pending: string[]          // 等下一轮 Eval 的 Trigger（去抖缓冲区）
  evaluating?: Evaluation    // 正在进行的 Eval：每个线程最多一个
  expired?: number           // 连续超时次数；任何 Decision 都会清零
}
interface Evaluation {
  triggerIds: string[]; primary?: string   // 这一批，以及给出最强强度的那条（将来是工单的 origin）
  strength; actor; basis                   // Gate 的结论，接单时变成 Mandate
  policyVersion: string                    // 按哪版策略判的；版本变了，这个 Eval 作废重排
  delivered?: string; deadline: string     // 截止时间从卡片送达主 Session 时才开始算
  pausedAt?: string                        // 暂停期间不过期，恢复后截止时间顺延
}
```

**Mandate 与 MandateBasis**

```ts
interface Mandate { strength; policyVersion; basis: MandateBasis; actor; scope: string }   // 不可变
type MandateBasis = { source?, scope?, row?, sender?, rules? } & (
  | { via: 'addressed'; addressed }                 // 名单内的人 @ 了机器人
  | { via: 'followup' }                             // 已有工单的线程上的后续消息
  | { via: 'admission'; cell; observation? }        // 准入表的某个格子
  | { via: 'bot'; observation }                     // 名单上的 bot：最多 inferred
  | { via: 'listed'; ... })                         // 旧版本数据的兼容分支
```

basis 记录的是“当初凭什么准入”，而不只是“准入了”。工单复活前要按**当前**配置重新检查这份依据（`revalidate`），所以必须把依据原样存下来。

**Ticket**

```ts
interface Ticket {
  mandate: Mandate; decisionId; origin: { triggerId; triggerIds }
  channels: ChannelRef[]           // 接单那批消息的回复地址（去重）
  ended?: { at; by: 'executor' | 'human'; outcome; reason? }   // 没有这个字段 = open
  attention?: { id; question; triggerId? }   // 当前要问人的那一个问题
  assignments: Assignment[]        // 一生可有多次派工；最新一次的报告就是工单“最后说的话”
  inputs: TicketInput[]; undelivered: string[]   // 输入的全集，以及还没投递的那些
  follows?: { thread; own; closed? }[]           // 跟随的 PR 线程：决定工单何时结束
  answers?: string[]               // 执行器已回答、主 Session 可以转达的问题
  execution?: { name; directory; assignment }    // 短名（也是分支名）、目录、派工文本
  delivery?: { error; at }         // 最近一次投递失败
}
```

注意 Ticket **没有** `state` 字段。它的阶段（等人回答、等人评审、工作中、排队……）是由 `ticketStage()` 从上面这些事实**读时计算**出来的（§7.9），所以永远不会和事实不一致。

**Decision**

```ts
interface Decision { id; evalId; threadKey; triggerIds; policyVersion; kind; reason: Reason[];
  strength; ticketId?; mandate?; audit?; dismissed? }
```

`kind` 只有五种：`accept`、`stop`、`ignore`、`unavailable`，以及旧版本留下的 `reopen`。`reason` 是结构化的原因码列表，不是自由文本：面板按代码本地化，agent 读英文模板（`src/core/reasons.ts`）。

## 6. 全系统伪代码（用源码里的名字）

```ts
// 每一行都是源码里真实存在的调用；// 后面是它所在的事务或时机
source.loop:      engine.receive(trigger)                 // 事务：存一次、扇出给订阅工单、排进 Eval 队列
passes.every2s:   effects.kick(); engine.evaluateDue()     // 每个 Eval 一个事务：过期、复活、启动 Eval
  evals.start:    policy.gate(...) → assessor.request()    //   → 排 assessment effect
effects.dispatch: for row of due(kind): handler(row)       // 事务提交之后才执行；失败退避
  assessment:     mainSession.steer(card); engine.candidateDelivered(evalId)
  launch:         prepare(); sessions.ensure(); prompt('queue'); engine.assignmentStarted()
  delivery:       worker.steer(inputs); engine.inputsDelivered()
  relay | notice: mainSession.steer(...) | mainSession.inject(...)
  reply:          registry.send(channel, text); outbox 回执落定
mainSession:      orchestrator_route → engine.decide() / attach()   // 事务：Decision + Ticket + 订阅 + 派工 + 确认
worker:           orchestrator_report → engine.report()             // 事务：报告、提问，可能结束工单
dsh events:       agent/status idle → engine.turnEnded()
                  session/event user/message → engine.inputsRead()
                  agent/error → 瞬时故障重试，否则 report(failed)
github.observe:   engine.threadClosed(pr, 'merged')                 // 事务：可能结束工单 → executor.stop（提交后）
```

**最关键的一行**是 `effects.dispatch` 那行的注释：**事务提交之后才执行**。Engine 里的任何决定，都先和它要引发的副作用一起写进 SQLite，再由分发器执行。所以崩溃发生在任何位置，要么什么都没发生，要么决定和待办都在，重启后继续。第三部分的六条流程都会回到这一点。

---

# 第二部分　模块精读

## 7. 领域核心 `src/core/`：只做决定，不做 I/O

**一句话：**这是一组同步的纯逻辑。它读写 Repositories，往端口里排待办，从不碰网络、文件或 LLM。

**为什么要先读它：**其余所有模块，要么在给它喂事实，要么在执行它排下的待办。

### 7.1 Engine：一个门面，五个部件

`Engine`（`src/core/engine.ts:24`）本身几乎没有逻辑。构造时创建五个部件（`engine.ts:31-38`），每个公开方法都是同一个模式：

```ts
receive(trigger: Trigger): ReceiveResult { return this.repo.atomic(() => this.intake.receive(trigger)) }
decide(triggerId: string, outcome: Outcome): Decision { return this.repo.atomic(() => this.evals.decide(triggerId, outcome)) }
```

即“一次调用 = 一个事务”。只有 `evaluateDue`（`engine.ts:75-88`）例外：它为**每个** Eval、每个工单各开一个事务。

```ts
evaluateDue(): void {
  const policy = this.options.policy()
  for (const thread of this.repo.threads.active()) if (thread.evaluating) this.repo.atomic(() => { this.evals.due(thread, policy) })
  if (policy.paused) return
  for (const ticket of this.repo.tickets.active()) {
    if (!liveTicket(ticket) && ticket.undelivered.length) this.repo.atomic(() => { this.tickets.wake(ticket.id) })
  }
  for (const thread of this.repo.threads.active()) {
    if (thread.pending.length && !thread.evaluating) this.repo.atomic(() => { this.evals.start(thread.key, policy) })
  }
}
```

拆成多个小事务有两个好处：一个线程的 Eval 出错不会回滚其他线程；每 2 秒一轮的 pass 也不会持有一个很大的事务。顺序也有讲究：先处理过期，再复活暂停期间收到人类消息的工单，最后才启动新的 Eval。

**五个部件的依赖是单向的：**Intake 和 Evals 依赖 Tickets；Assignments 依赖 Tickets 和 Replies；Tickets 依赖 Replies；Replies 不依赖任何部件。跨部件的命令（比如 `attach`）由 Engine 按顺序调用各部件（`engine.ts:109-122`）。

### 7.2 EnginePart：几个共享原语

`src/core/part.ts` 是五个部件的基类，提供几个反复用到的原语：

- `owners(trigger)`：这条人类消息现在归哪些工单裁决（转调 `followupOwners`，见 §7.8）。
- `leftToWorkers(trigger)`：如果有工单还在裁决这条消息，就拒绝主 Session 对它做 stop 以外的路由（抛 `workerDecides`）。
- `handBack(ticketId, trigger, reason)`：记录交回，并且**只有当这个工单确实拥有它、交回后也没有别的工单拥有它**时，才重新排进 Eval（`part.ts:42-48`）。
- `queueForEval(trigger)`：如果它已在排队或正在被评估，就什么也不做（幂等，`part.ts:57-63`）。
- `thread(ref)`：取线程，不存在时返回一个**未保存的空聚合**（`part.ts:69-72`）。线程是按需创建的，只有真正写入时才落库。

### 7.3 Intake：一条消息进门

`Intake.receive`（`src/core/intake.ts:19-62`）按这个顺序执行：

1. 时间戳规范成 ISO 格式（GitHub 只精确到秒），后面就能直接按字符串比较时间。
2. 同 id 已存在就返回 `duplicate`。**这是整个系统幂等的第一道闸。**
3. 问 `SourceScope` 在不在听。不在听也**照样存下来**，同时记 `filtered` 及原因码，以便事后在面板上“重新评估”。
4. 对它所属的每个线程：更新 `lastAt` 和关联线程列表，并遍历线程的订阅者，决定能扇出给谁：

```ts
const followed = !!thread.context?.includes(ticketId)
const resumes = trigger.human && ticket?.ended?.by !== 'human' && !followed
if (!ticket || (!liveTicket(ticket) && !resumes)) continue
reached.set(ticketId, { ticket, context: followed && (reached.get(ticketId)?.context ?? true) })
```

活工单收所有消息；**已结束的工单只收“人在非上下文线程上写的消息”**，而且不能是被人叫停的工单。bot 和 CI 消息不会唤醒已结束的工单，因为执行器复活时会自己重读线程。

5. 给每个被触达的工单加一条 Input，id 是 `key('input', ticketId, 'trigger', triggerId)`，同一条消息对同一个工单只会记一次。
6. 写去向记录。**如果没有工单拥有这条消息（`owners` 为空），就排进主线程的 Eval 队列。**bot 消息总是走这条路，因为 `followupOwners` 只认人类消息。
7. 对每个被触达的工单调 `wake`（只有已结束的工单才会真的被复活，见 §7.5）。

### 7.4 Evals：批处理、取最强、可重放

**启动一个 Eval**（`src/core/evals.ts:71-119`）：

1. 先剔除已经被工单拥有的后续消息。这是给旧版本数据的兼容处理：旧版本可能把它们也排进了队列。
2. 剔除**已经开过工单的 Trigger**，记一个 `startedTicket` 的 ignore（`evals.ts:83-86`）。同一条消息被重放（比如有人点了“重新评估”），也不会开出第二个工单。
3. 对批内每条 Trigger 跑 Gate，挑出“最强”的一条作为这个 Eval 的依据：

```ts
for (const trigger of triggers) {
  const gate = policy.gate(trigger, thread, tickets)
  if (!gate.admissible) { reasons.push({ code: gate.reason, triggerId: trigger.id }); continue }
  const later = gate.strength === strongest?.strength && trigger.at >= strongest.trigger.at
  const weaker = !strongest || stronger(strongest.strength, gate.strength) !== strongest.strength
  if (weaker || later) strongest = { trigger, strength: gate.strength, basis: gate.basis }
}
```

规则是 explicit 胜过 inferred；强度相同时，**取最晚的那条**。这样工单的 origin 就是批内最新、最强的那条请求，确认消息也会回复在它下面。

4. 一条都不准入，就整批记 ignore，每条带各自的原因码。否则建 `Evaluation`：id 由 `key('eval', 线程, 策略版本, 时间, ...批内 id)` 算出；清空 pending；给每条 Trigger 的去向记录写上 evalId；然后 `assessor.request(candidate)`。

**截止时间**（`evals.ts:35-69`）：Eval 刚创建时，截止时间按“现在 + 超时”占位，但**在卡片送达主 Session 之前不会过期**（`due` 里检查 `evaluation.delivered`）。`candidateDelivered` 被调用时，截止时间才重算为“送达时间 + 超时”。所以主 Session 忙、卡片在队列里排了很久，也不会被误判为超时。暂停时记下 `pausedAt`，恢复后截止时间顺延暂停的时长。过期后把整批重新排到队首；连续过期第 3 次（`REQUEUES = 2`）记为 `unavailable`。

**做出决定**（`evals.ts:131-159`）有一串前置检查，每一条都对应一种真实的竞态：

- 调用方给的 `evalId` 不是当前持有这条 Trigger 的 Eval，就拒绝（`evalFinished`）。主 Session 拿着旧卡片做决定时会碰到这种情况。
- accept 一条已经开过工单的 Trigger，就**返回原来那个 Decision**，不报错、不重复建单（幂等重放）。
- 不是 stop 的决定，要先确认没有工单在裁决这条消息（`leftToWorkers`）。
- 没有 Eval 持有这条 Trigger（例如主 Session 直接路由一条 explicit 请求，或叫停一条后续消息），就建一个只跑 Gate 的“直接 Eval”（`directEval`，`evals.ts:271-280`）。它的 id 里包含决定的种类，所以对同一条消息同时做 stop 和 accept 会得到两个 Decision。
- inferred 的 accept 必须满足 `admitsInferred(rules)`：模式是 auto，且命中的规则都启用着。

**应用决定**（`decideEval`，`evals.ts:282-304`）的第一步是版本检查：

```ts
if (this.policy().version !== evaluation.policyVersion) {
  if (thread.evaluating?.id === evaluation.id) return this.reassess(thread, { code: 'policyChanged' })
  return this.finish(thread, evaluation, { kind: 'unavailable', reason: [{ code: 'policyChanged' }], ...audit })
}
```

即使 `policyChanged` 事件因为重启或升级丢失了，按旧策略做出的决定也不会生效。之后：没有研判就忽略的 inferred 候选，会被打上 `ignoredFromCard` 标记，与“逐条研判后忽略”区分开；stop 的目标必须是活工单，且强度必须是 explicit，否则记成 ignore 并附原因码；accept 时生成 Mandate，工单 id = `key('ticket', evalId)`，再交给 `Tickets.create`。

**记录**（`record`，`evals.ts:327-339`）：Decision 的 id = `key('decision', evalId)`，同时把 decisionId 写进批内每条 Trigger 的去向记录。“直接 Eval”的 Decision 不会清掉线程上正在进行的 Eval（`finish`，`evals.ts:318-325`）。

**交回与路由：**

- `routed`：主 Session 把消息 attach 到某个工单后，持有它的 Eval 就该结束。批里只有这一条，记一个 `routed` 的 ignore；否则用 `reassess` 把**其余的**重排。
- `handback`：校验这个工单确实收到过这条消息、而且没交回过，然后调用 `handBack`。

### 7.5 Tickets：工单的一生

这是核心里最大的部件（`src/core/tickets.ts`，423 行），关键逻辑集中在几个函数里。

**创建**（`create`，`tickets.ts:37-56`）：

- origin 取 Eval 的 primary，也就是给出最强强度的那条 Trigger。
- 回复渠道取批内所有 Trigger 的 channel，去重。
- 调 `prepareTicket` 钩子：DshExecutor 在这里算出短名、目录和派工文本。
- 建第一个 pending 派工；订阅批内所有 Trigger 的所有线程。
- **只有当 origin 是人写的、并且有回复渠道时**，才排确认消息。
- 最后 `executor.start`，即排一个 launch effect。

**加输入**（`addInput`，`tickets.ts:234-245`）：

```ts
if (ticket.inputs.some(existing => existing.id === input.id)) return
ticket.inputs.push(input); ticket.updatedAt = this.now()
// Hold back automated inputs while the worker is waiting on a person.
const held = liveTicket(ticket) && automatic(input) && waitsForPerson(ticket)
if (!held) ticket.undelivered.push(input.id)
this.repo.tickets.save(ticket)
if (!liveTicket(ticket) || held) return
const running = runningAssignment(ticket), latest = latestAssignment(ticket)
if (running) this.options.executor.deliver(ticket, running)
else if (latest?.state === 'pending' && !waitsForPerson(ticket)) this.options.executor.start(ticket, latest)
```

三个细节：

1. 按 id 幂等。
2. 工单在“等人”（最新报告是 blocked 或 failed，且之后没有人发言）时，CI、bot 和主 Session 的便条**只记录，永不投递**。这样受阻的执行器不会被自动消息反复唤醒。
3. 有运行中的派工就投递；派工还是 pending（例如上次启动失败），就再启动一次。

**复活**（`wake`，`tickets.ts:252-270`）：只对“已结束、不是被人叫停、有待投递输入”的工单生效。暂停时什么都不做，等暂停结束后的第一轮 pass。Mandate 重验证失败，就**拒收**（`refuse`）：清空待投递，把这些消息还给 Eval（`letGo`）。只有人类消息能复活工单：bot 和 CI 的输入只是跟着排队。复活时在待投递队列**最前面**插入一条 `resumed` 输入，执行器收到的第一句话就是“这个工单之前结束了，下面这条消息让它复活了……”（§12 有原文）。

**结束**（`end`，`tickets.ts:345-363`）是保证“人类消息不丢”的关键：

```ts
const until = readUntil(ticket)
const waiting = this.triggersOf(ticket, ticket.undelivered)
const left = this.untaken(ticket, ticket.inputs.flatMap(input => input.kind === 'trigger'
  && (!readBefore(input, until) || ticket.undelivered.includes(input.id)) ? [input.id] : []))
ticket.ended = { at, by, outcome, ...why }; ticket.undelivered = []; ticket.updatedAt = this.now()
delete ticket.attention; delete ticket.delivery
for (const assignment of stopping) { assignment.state = 'ended'; assignment.endedAt = at }
this.repo.tickets.save(ticket)
this.letGo(ticketId, waiting, left)
```

- `until` = 最新一次报告与最后一次 turn 结束中较晚的那个时间（`readUntil`，`model.ts:341`）。
- `waiting` = 从没投递出去的消息。
- `left` = 归这个工单裁决、但在 `until` 之前没被读到（或根本没投递）、也没被确认、回答或交回的**人类消息**。

`letGo`（`tickets.ts:294-309`）把这两类消息从工单的扇出列表里拿掉。对 `left` 中的每一条，如果没有别的工单拥有、没人确认过、也没被某个 Decision 结算过，就重新排进 Eval。

为什么要这么麻烦？想象执行器正在写“ready”报告时，有人在 PR 下补了一句话。工单因为这份 ready 结束了，执行器再也没机会读到那句话，也没机会交回。如果没有 `letGo`，这句话就被悄悄吞掉了。

**PR 关闭时结束**（`threadClosed` / `followsDone` / `unfinished`，`tickets.ts:77-101`、`370-379`）：

- `followsDone`：如果工单自己开过 PR（`own`），就看“自己开的 PR 是否都已关闭”；没开过，就看“跟随的所有 PR 是否都已关闭”。结局取最后一次关闭的结果（merged 或 closed）。
- `unfinished`：即使 PR 都关了，下面两种情况也暂不结束，等下一份 ready 或 dropped 报告：执行器在最近一次非 progress 报告或 turn 结束之后接手过一条人类消息；或者 PR 关闭之后才复活的工作还没报告 ready。

这是“PR 合并了，但有人在合并后提了新要求，执行器正在做”的情形。

**重试与重启**（`retry`、`restart`、`requeueUntaken`，`tickets.ts:168-180`、`318-337`）：都是“结束旧派工、建新派工”。区别在于重启会多加一条恢复说明。两者都会把**投递给旧派工、但会话从没读进历史**的输入（`readAt === null`）放回待投递队列的最前面。这种消息可能被取消的 turn 或中断的 step 丢掉了。

**其他：**

- `attach`（`tickets.ts:121-144`）处理了 4 种“这条消息执行器是否已经有了”的情况，决定是复用原输入、更新便条，还是作为新输入再投一次。
- `markUnrelated` 不允许执行器丢掉起源线程，也不允许丢掉 Source 绑定的 PR 线程；丢掉之后，把它在那个线程上拥有的人类消息全部交回。
- `policyChanged` 按建单时间从早到晚通知受影响的工单。输入 id 里带上时间，所以在两个版本之间来回切换，每次都会产生新的通知。

### 7.6 Assignments：执行器上报的事实

`src/core/assignments.ts` 只做一件事：把执行器报来的事实写进最新的派工。

- `started`：pending → running；有待投递输入就立刻投递。
- `delivered`：从待投递中移除；输入的 `readAt` 置为 `null`（“已投递、还没读”），并记下投给了哪次派工。**事实如果来自已经不在运行的派工，就静默丢弃**（`running()` 守卫，`assignments.ts:136-139`）。
- `read`：`readAt ||= now`。它对应 DSH 的 `user/message` 会话事件，也就是这条消息真正进入了模型历史。
- `turnEnded`：记 `idleAt`。
- `interrupted`（`assignments.ts:83-92`）：进程重启时，对每个运行中的派工做判断：

```ts
const busy = !!running.deliveredAt && !!report && running.deliveredAt >= report.at
const settled = !!report && ['waiting', 'ready', 'blocked', 'failed'].includes(report.state) && !busy
if (!settled) this.tickets.restart(ticket, running, 'hostRestarted', RECOVERY_NOTE)
else if (ticket.undelivered.length) this.options.executor.deliver(ticket, running)
```

如果它已经停在一个“稳定”的报告上，且之后没有新输入，就保留会话，只补投待投递的输入；否则就重启，并附上恢复说明。

- `report`（`assignments.ts:99-134`）：
  - 校验：问题不能和 ready 或回答同时出现；“由哪条消息引发的问题”必须指向一条人写的、到达了本工单、有回复渠道的消息。
  - 提问逻辑：同一个问题沿用原 id；新问题生成新 id `key('attention', ticketId, question, at)`；`null` 或 ready 清除问题。问题被替换或清除时，**撤回 outbox 里还没发出的旧问题**。
  - blocked 或 failed 时，把待投递中的自动消息全部移除。
  - 结束判定：ready 且不跟随 PR 线程 → `completed`；ready 且跟随 → 看 `followsDone`；dropped → 若 PR 已关闭则按关闭结果，否则记 `dropped`。

### 7.7 Replies：每一种发言都有前置条件

`src/core/replies.ts` 把对外发言分成四种用途，每种都由代码检查前置条件，并用确定性 id 防止重复发送：

| 用途 | 前置条件 | id | 发到哪 |
|---|---|---|---|
| `acknowledgement` | 工单是活的；这条消息到达了本工单（不算只为上下文关注的线程）；有回复渠道；是人写的（除非已排过） | `key('ack', triggerId)` | 原消息下方 |
| `answer` | 原消息是人写的；执行器在报告里声明过回答了它（`ticket.answers`） | `key('reply', triggerId, text)` | 原消息下方 |
| `result` | 工单已报告 ready，或已以 completed、merged、dropped 结束 | 同上 | 工单起源的对话 |
| `attention` | 必须与工单**当前**的问题逐字一致 | `key('attention-reply', attentionId)` | 提出问题的那个人所在的对话，否则是起源对话 |

两个精巧之处：

- **确认消息全局唯一。**id 只由 triggerId 决定，所以无论哪个执行器、重启多少次，一条人类消息最多得到一个确认；后来的调用保留**第一次**排队的文字。接单时自动排的确认和执行器后来调 `orchestrator_ack` 的确认，用的是同一个 id。
- **回答权限只能用一次。**发出回答后，从 `ticket.answers` 里移除这个 triggerId；但因为 id 由文本决定，**同一段文字的传输重试**仍然会命中已存在的回执而被放行（`replies.ts:221-222`）。

### 7.8 model.ts 里的几个判定函数

这些纯函数被多个部件共享，是规则一致性的来源：

- `followupOwners(trigger, records)`（`model.ts:75-91`）：一条**人类**消息现在归哪些工单裁决。条件是：到达过这些工单，且不是只为上下文关注；没被这些工单交回过；没有被人重新送去评估；不在某个 Eval 里。然后，工单要么还活着，要么不是被人叫停、并且这条消息还在它的待投递里（等着复活它）。实现上先用去向记录排除掉大部分消息，只有可能有主人的才去读线程和工单。
- `waitsForPerson`（`model.ts:349-353`）：最新报告是 blocked 或 failed，且之后没有人在非上下文线程上发过言。
- `readUntil`（`model.ts:341-347`）：最新报告和最后一次 turn 结束中较晚的那个时间。它是“执行器有机会读到某条消息”的时间上限。
- `reached`（`model.ts:326-332`）：一条消息是否到达了某个工单。`context = false` 时排除只为上下文关注的线程，用于“只能在自己的线程上确认”这类检查。
- `settled`（`model.ts:93-94`）：一个 Decision 是否真正处理了这条消息。stop、unavailable、路由、被拒的 stop 都不算。

### 7.9 stage.ts：阶段是算出来的

```ts
export function ticketStage(ticket: Ticket): TicketStage {
  if (ticket.ended) return { stage: 'ended' }
  if (ticket.delivery) return { stage: 'person', action: 'delivery' }
  if (ticket.attention) return { stage: 'person', action: 'answer' }
  const assignment = latestAssignment(ticket)
  const report = assignment?.report?.state
  if (report === 'ready') return { stage: 'person', action: 'review' }
  if ((report === 'blocked' || report === 'failed') && waitsForPerson(ticket)) return { stage: 'person', action: report }
  if (!assignment || assignment.state === 'pending') return { stage: 'working', action: 'queued' }
  if (report === 'waiting' || assignment.state === 'ended') return { stage: 'idle' }
  return { stage: 'working' }
}
```

这 10 行回答的是“这个工单在等谁”，与 `docs/003-execution.md` 的阶段表逐行对应，由 `tests/stage.spec.ts` 逐行测试。`canRetry` = 工单未结束，且在“等人”或投递失败。面板、工单卡片、重试按钮都用这同一个函数，所以“面板显示可重试”和“重试命令被接受”永远一致。

### 7.10 ids、errors、reasons、memory

**确定性 id 是幂等的来源。**`key(...values)` = 对 JSON 序列化后的参数做 sha256，取前 32 个十六进制字符（`src/core/ids.ts`）。下表列出了核心里所有的 id 规则：

| 对象 | id 公式 | 带来的性质 |
|---|---|---|
| Eval | `key('eval', 线程, 策略版本, 时间, ...批内 id)` | 同一批在不同策略版本下是不同的 Eval |
| 直接 Eval | `key('eval', 线程, 'direct', triggerId, 决定种类, 工单?, 时间)` | 对同一条消息同时做 stop 和 accept 互不覆盖 |
| Decision | `key('decision', evalId)` | 一个 Eval 只有一个 Decision |
| Ticket | `key('ticket', evalId)` | 重放 accept 不会建第二个工单 |
| Assignment | `key('assignment', ticketId, 序号)` | 派工可按序号重建 |
| Input（消息） | `key('input', ticketId, 'trigger', triggerId)` | 同一消息对同一工单只记一次 |
| Input（复活、策略、重启、便条） | 分别带 triggerId、版本对 + 时间、派工 id、时间 + 文本 | 各自幂等，又能区分真正的新事件 |
| 确认 | `key('ack', triggerId)` | 一条人类消息全系统最多一个确认 |
| 提问 | `key('attention', ticketId, question, 时间)`；回复 `key('attention-reply', attentionId)` | 同一问题不重复问；问题替换后能精确撤回旧回复 |
| 回答、结果 | `key('reply', triggerId 或 ticketId, text)` | 同文本的传输重试自然去重 |

**错误码同时也是给模型的纠偏提示。**`USER_ERRORS`（`src/core/errors.ts`）分两类：`missing`（HTTP 404）和 `refused`（HTTP 400）。很多拒绝信息是写给 LLM 看的，会直接告诉它下一步该怎么做。例如 `notReady`：

> “Ticket is not ready for delivery. Keep progress internal and wait for a verified ready report; do not relabel progress as an answer or send it through another channel.”

模型调错工具时收到的不是一个裸的 400，而是一段纠偏指令。

**原因码。**`REASONS`（`src/core/reasons.ts`）有 20 个原因码，每个配一条英文模板（给 agent 读）；面板按 `reason.<code>` 本地化。Gate 和 Source 也可以产出自己的原因码（如 `not_a_member`）。

**内存仓储。**`MemoryRepositories`（`src/core/memory.ts`）用 Map 快照实现回滚，并且刻意**与 SQLite Store 的排序语义保持一致**（例如 `tickets.list()` 最新的在前）。`tests/core-engine.spec.ts` 用它在不碰 SQLite 的情况下测试全部领域规则。

### 7.11 本节小结：亮点与代价

- **亮点：**领域规则全部是同步纯逻辑，事务边界清楚；幂等靠确定性 id 而不是“先查后写”；阶段读时计算；“人类消息不丢”有完整闭环：扇出 → 已读追踪 → 结束时回流。
- **代价：**规则之间耦合很紧。`end`、`letGo`、`unfinished`、`attach` 每个都处理 4–6 种边界情况，注释密度很高，读懂要花时间；任何一条规则改动都可能牵动另外几条，所以 `tests/core-engine.spec.ts` 和 `tests/followups.spec.ts` 写得非常细。另外，大量 JSON 字段（inputs、assignments）直接嵌在工单行里，工单越活跃，这一行就越大，每次保存都要整行重写。
