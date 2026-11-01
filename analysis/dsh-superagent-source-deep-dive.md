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
| Effects 分发器 | 入队时用 microtask 立即调度；没有自己的定时器，退避到期的行靠 Passes 每 2 秒的 kick 捡起 | 6 种副作用：assessment、launch、delivery、relay、notice、reply | 适配器回调 Engine 的“事实”方法 |
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

## 8. 持久化与副作用：Store、Effects、Outbox、Passes、Runtime

**一句话：**这一层把 Engine 排下的待办可靠地执行出去：先落盘，再执行，失败退避，崩溃后继续。它是“不丢、不重”的工程底座。

**涉及文件：**`src/app/store.ts`、`store-read.ts`、`effects.ts`、`outbox.ts`、`passes.ts`、`runtime.ts`、`notices.ts`、`delivery.ts`、`versions.ts`、`src/kv.ts`、`src/infra/health.ts`。

![图 3：决定先落盘，再执行](img/l1-durable-kernel.svg)

**图 3：Engine 只写数据库、不做 I/O。每个对外动作先作为 effect 行，与领域状态在同一个 SQLite 事务里提交，提交后由带并发上限和指数退避的分发器执行；适配器把结果作为“过去时的事实”交回 Engine，开启新的事务。**

### 8.1 Store：一个 SQLite 文件，JSON 行加生成列

- **打开方式**（`store.ts:58-101`）：`journal_mode=WAL`、`synchronous=FULL`，每次提交都 fsync；目录权限 0700，数据库文件 0600。建表、对齐生成列、迁移都在一个事务里完成。
- **表结构：**`trigger`、`disposition`、`thread`、`ticket`、`decision`、`outbox`、`effect`、`kv`，外加倒排表 `trigger_thread`（一条 Trigger 可以属于多个线程，时间线按 `线程 + at` 分页）。领域对象整行存 JSON，**不拆列**。
- **工作集靠生成列 + 部分索引**（`store.ts:26-44`）：

| 生成列 | 何时非空 | 用途 |
|---|---|---|
| `ticket.open` | 没有 `ended` | `tickets.open()` |
| `ticket.active` | 有待投递输入，或最新派工是 pending 或 running | `evaluateDue` 只扫这些 |
| `thread.active` | 有排队的 Trigger，或有进行中的 Eval | 同上 |
| `thread.subscribed` | 有订阅时，取所属 Source | Source 轮询订阅线程 |
| `outbox.pending` | 状态是 pending 且有 ticketId 时，取 `ticketId + ' ' + 会话` | “同一工单、同一会话保序”的分组键 |
| `outbox.dropped` | 被丢弃时，取丢弃时间 | 面板列出被丢弃的消息 |

  条件不满足时生成列为 NULL，再配合 `WHERE x IS NOT NULL` 的部分索引，每 2 秒一轮的 pass 只扫活跃行。JSON 是唯一事实来源，不存在“状态列和 JSON 不一致”的问题。每次打开数据库时，`addColumns()` 按定义文本对齐生成列，所以老库会自动补列。
- **嵌套事务**（`store.ts:140-153`）：每层用一个命名 SAVEPOINT；失败时 `ROLLBACK TO` 加 `RELEASE`；`afterCommit` 的回调按层收集，只在最外层提交后执行，回滚层的回调直接丢弃。注释写明了分工：**纯内存的后续动作用 afterCommit，需要扛过崩溃的副作用用 Effects。**

### 8.2 Effects：持久化的副作用队列

每个 `(kind, key)` 只有一行，字段及其用途：

| 字段 | 作用 |
|---|---|
| `seq` | 队列位置。启动时从 `MAX(seq)` 续上，重启后仍单调递增 |
| `generation` | 每次重新入队加 1，防止旧的执行删掉新的 payload |
| `attempts` | 连续失败次数：决定退避时长、健康状态、outbox 是否放行 |
| `due` | 下次可以运行的时间，退避和 `Later` 都靠推后它实现 |
| `since` | 首次入队时间，用于显示“排队了多久” |

**入队是合并式的 upsert**（`effects.ts:61-66`）：

```ts
INSERT INTO effect (kind, key, value, seq, due, since) VALUES (?, ?, ?, ?, ?, ?) ON CONFLICT(kind, key) DO UPDATE
  SET value = excluded.value, generation = generation + 1, attempts = 0, error = '', due = excluded.due
```

同一个 key 再次入队时：payload 换成最新的；**保留原来的队列位置（seq）**；失败计数清零并立即到期。这些 effect 表达的是“某个对象的最新状态需要同步出去”，例如“把工单 T 的待投递输入投出去”。后来的变化不该插队，也不该继承旧的退避。

**“提交后才执行”是怎么做到的：**`enqueue` 在调用方的事务里写行，然后 `kick()` 用 `queueMicrotask` 调度分发。SQLite 驱动（`node:sqlite` 的 `DatabaseSync`）和 `atomic` 都是同步的，微任务只可能在最外层事务提交之后运行；如果事务回滚，这一行根本不存在，分发器什么也读不到。代价是隐含了一个不变量：**事务里不能有 await**。

**分发**（`effects.ts:103-122`）：

- **有序的 kind**（relay、notice）：永远只取最老的一行，即使它还在退避中，也挡住后面所有行（队头阻塞）。主 Session 需要按因果顺序看到报告和通知。
- **无序的 kind**：取已到期的行，最多 `16 + 运行中数量`，排除正在运行的 key（同一 key 不会并发执行），并发数由 `limit()` 控制。limit 为 0 就是暂停（暂停处理、工位已满）。
- **执行结果**：成功时按 `kind + key + seq + generation` 精确删除自己那一行；关闭过程中失败的不动，下次启动重跑；抛 `Later` 只推后 due，不计失败；其他错误按 `min(上限, 初值 × 2^attempts)` 退避。

**Effects 没有自己的定时器**：退避到期的行，要等 Passes 每 2 秒一次的 `kick()` 才会被捡起，所以退避的精度大约是 2 秒。

六种 effect 的配置：

| kind | 注册者 | key | 有序 | 并发 | 退避 |
|---|---|---|---|---|---|
| `assessment` | MasterAssessor | evalId | 否 | 暂停时 0，否则 4 | 2 s → 60 s |
| `launch` | DshExecutor | assignmentId | 否 | 暂停时 0，否则等于空闲工位数 | 5 s → 300 s |
| `delivery` | DshExecutor | ticketId | 否 | 暂停时 0，否则 4；失败不报警（显示在工单上） | 2 s → 60 s |
| `relay` | Sessions | requestId | **是** | 1，**暂停时也照常** | 2 s → 60 s |
| `notice` | Sessions | 通知名 | **是** | 1，**暂停时也照常** | 2 s → 60 s |
| `reply` | Outbox | 消息 id | 否 | 暂停时 0，否则 1 | 5 s → 300 s |

relay 和 notice 在暂停时也照常投递，因为主 Session 需要收到“已暂停”这类通知。

### 8.3 Outbox：回执、保序与丢弃

一条出站消息的回执状态机：

```
（无）──send：插入回执 + 排 reply effect（同一事务）──▶ pending
pending ──Source 发送成功──▶ sent（终态）
pending ──Source 回答会话已不存在──▶ dropped: channelGone（终态）
pending ──Source 不再监听这个会话──▶ dropped: channelIgnored（终态）
pending ──withdraw：删除回执 + 取消 effect（同一事务）──▶（无）
```

- **回执永久保留。**同一个 id 再次 `send` 是空操作，这是 Engine 确定性消息 id 能够幂等的前提（§7.10）。
- **同工单、同会话保序**，而不是全局保序（`store.ts:259-261` 的 `behind` 查询）：只要存在一条分组键相同、更早、且失败不到 3 次的待发消息，这条就抛 `Later` 让路。其他工单、其他会话不受影响；前面那条失败 3 次之后，不再挡住后面的。这比把 reply 设成有序 kind 更精细：一条卡住的消息不会挡住所有人。
- **至少一次投递，靠渠道去重。**`sources.send` 和 `settle` 不在一个事务里：崩溃发生在“发出去之后、记成已发之前”，重启后会再发一次。去重靠渠道自己：飞书用 `--idempotency-key <消息 id>`（`feishu-channel.ts:23`）；GitHub 在评论里嵌入 `<!-- dsh-superagent:<id> -->`，发送前先扫描是否已经发过（`github-channel.ts:34`）。新接入的 Source 必须遵守这个幂等约定（`kind.ts` 写明了 `send` 的幂等要求）。

### 8.4 Passes：每 2 秒一轮，同一时刻只跑一轮

```ts
async pump(): Promise<void> {
  if (!this.host.configured() || this.host.closing()) return
  if (!this.pumping) return this.pumping = this.pumpOnce().finally(() => { this.pumping = undefined })
  const next = () => { this.pumpAgain = undefined; return this.pump() }
  return this.pumpAgain ??= this.pumping.then(next, next)
}
```

一轮正在跑时，无论来多少次请求，都只合并成**一次**后续的 pass；当前这轮失败了，后续 pass 照样执行（`then(next, next)`）。每轮先 `effects.kick()`，再 `engine.evaluateDue()`。请求来源有：每 2 秒的定时器、`receive` 存下新 Trigger、Settings、工具和管理 API。

### 8.5 notices：唤醒主 Session，还是只让它知道

给主 Session 的消息分两种，区别在于**会不会让它开始一个新的 turn**：

- **relay**（steer，会唤醒）：执行器的报告（ready、blocked、failed、dropped、waiting，以及带回答的 progress）；带问题的报告（`worker_attention`）；人在工单线程上的后续消息（`follow_up`）。候选卡片也是 steer，但走的是 assessment effect。
- **notice**（inject，不唤醒，等下次被唤醒时一起带上）：工单结束、Trigger 被重新排队、订阅被取消、执行器关注了某个线程、配置或提示词被保存。
- **普通的 progress 报告不 relay。**它只更新工单记录，主 Session 完全不知道（`notices.ts:121-129`）。这是“注意力预算”在代码层面的体现：连路由 LLM 的注意力也要节省。
- **不把主 Session 自己的操作再通知给它。**Runtime 维护一个 `quiet` 计数，主 Session 的工具执行 Engine 命令时，由此引发的领域事件不再变成通知（`runtime.ts:139-146`）。

### 8.6 AgentDelivery：每条消息只投递一次，进程重启后也成立

投递给 DSH 会话的每条消息，都用 `dsh-superagent:<requestId>` 作为确定性的消息 id（`delivery.ts:16`）。投递前检查三处：已经消费过的（从会话持久化日志里读出来，并订阅实时事件补充）、还在 inbox 里排队的、刚刚 inject 过的。三处都没有才投递，然后 `sessionPersistence.flush()` 作为持久化屏障（`delivery.ts:66-85`）。effect 行要等投递返回后才删除，所以：

- 崩溃在 flush 之前：重投时会被日志或 inbox 去重；
- 崩溃在 flush 之后、删行之前：重投时直接跳过。

### 8.7 Runtime：组合根的启动与关闭

- **启动顺序**（`runtime.ts:88-217`）：
  1. 锁住状态目录（`proper-lockfile`），**在打开数据库之前**，防止两个进程同时迁移；
  2. 打开 Store；
  3. 按依赖顺序构造 Setup、Prompts、Commands、Effects、SourceRegistry、Sessions、Settings、DshExecutor、MasterAssessor、Outbox、Engine、Passes；DshExecutor 和 MasterAssessor 通过 `get engine()` 拿到 Engine，打破循环依赖；
  4. 先注册管理 API，所以 setup 没完成时面板也能用；
  5. 跑 Setup，失败就停在“等待配置”，不启动任何循环；
  6. 安装会话 hook，确保主 Session 存在，同步名单；
  7. 恢复中断的派工，启动 Passes 和 Effects。

  在 `effects.start()` 之前，持久化的待办只是安静地躺在表里。
- **关闭顺序**（`runtime.ts:229-253`）：同一个同步片段里先 `abort`，再 `drain`，让因 abort 失败的 handler 不计入失败次数；取消主 Session 和所有执行器的 turn，但保留 inbox；等 Passes、Effects、agent 都空闲；先停执行器的后台 job 和 subagent，免得它们结束时再把执行器唤醒；**先关数据库，再释放锁。**

### 8.8 迁移：只支持“上一版”

`migration.ts` 在 Store 构造的同一事务里，把 schema 10 的记录原地改写成 schema 11 的形状。任何一步失败，整体回滚，数据库保持 schema 10 不变。更老的库直接拒绝，提示换一个新的状态目录。文件头注释说明了目的：**让运行时的读取方永远不需要处理旧形状。**代价是跨多个版本升级的用户必须重建状态。

### 8.9 本节的问题与局限

- **（确认）内容相同的配置通知会被永久去重。**notice 的投递 id 是 `notice:<名字>:<内容哈希>`。暂停、恢复、再暂停时，“已暂停”通知的内容完全一样，同一个主 Session 生命周期内，第二次就不会再 inject 了，effect 却按成功删除（`sessions.ts:180-186`、`notices.ts:22-24`）。
- **（确认）有序 kind 没有死信队列。**一条永久失败的 relay 会一直挡住后续所有 relay，只能通过健康状态发现。
- **（推断）锁失效后后台循环仍在运行。**`onCompromised` 只把健康状态置为 error 并 abort，没有停 Passes，也没有 drain Effects。
- **（确认）`SourceContext.state` 不按 Source 隔离。**Source 可以读写任何 KV 键，包括 schema 版本和配置快照。
- **（确认）数据只增不减**：回执、Trigger、Decision、设置历史、Session 用过的提示词版本，都没有清理逻辑。

## 9. Source 层：把外部世界翻译成 Trigger

**一句话：**Source 负责两件事。入站：轮询或订阅外部事件，翻译成 Trigger 交给 Engine；出站：通过自己的 Channel 把消息写回原对话。它只报告外部事实，不做任何领域决定。

### 9.1 插件协议：SourceKind 与 SourceContext

**`SourceKind`（`src/sources/kind.ts`）的成员：**

| 成员 | 必选 | 谁调用、用来做什么 |
|---|---|---|
| `kind`、`title`、`description` | 是 | 注册键与面板显示 |
| `schema`、`fields`、`facts` | 是 | 校验配置（校验失败就禁用该实例，不崩溃）、生成面板表单 |
| `health` | 是 | 注册时在健康板上认领条目；`report` 只能写声明过的条目 |
| `scope` | 否 | **最关键的一个**：Intake 用它过滤，Gate 用它判断“全收”和准入表的行，Mandate 重验证也用它 |
| `channel`（`send`、`allowed`、`link`） | 否 | Outbox 发消息；`allowed` 为假时回执记为 `channelIgnored` |
| `threadContext` | 否 | 研判窗口的读取器 |
| `find`、`repliesTo` | 否 | 搜索；从正文里的引用找到相关工单 |
| `policy`（`schema`、`sync`） | 否 | 这个 Source 的策略部分（名单、bot 名单），以及名单同步 |
| `environment`、`refreshLabels` | 否 | 执行器的环境变量；刷新显示名 |

**`SourceContext` 刻意给得很窄**（`src/app/source-context.ts`）：Source 能拿到配置、命令、`receive`、KV 状态、线程的读取与 meta 写入、工单的**只读**查询，以及工单方面唯一的两个写操作：`bindThread` 和 `threadClosed`。它拿不到 `report`、`end`、`handback`，也拿不到 Policy 和 Session。

原因是“PR 合并了，工单该不该结束”要看 `followsDone`、`unfinished`、是否被人叫停，这些都是核心层的知识。Source 只知道外部事实：“这个线程关闭了，结果是 merged，时间是 T”。

（确认）这道隔离并不彻底：`state.set` 可以写任意 KV 键，没有命名空间。注释写着“Source-scoped”，实际只是约定。

### 9.2 GitHub：一次轮询回合

`watch` 循环（`github.ts:127-146`）每轮调用 `pollRound`，然后等 `max(GitHub 要求的间隔, 配置的间隔)`，至少 60 秒。连续失败若干轮才报 error，成功一轮就清零。一轮依次做四件事，任何一件失败都不影响后面的：

1. **通知**（`pollNotifications`，`github.ts:298-343`）：
   - 调仓库级通知端点（`Commands.path` 自动加 `repos/{repo}/` 前缀）；读 `X-Poll-Interval`。
   - 待读队列 = 上一轮失败的条目 + 本轮通知（按时间正序）。
   - 逐个读 Issue 或 PR：Issue 正文只在**新开**时读一次；评论按 `since` 过滤；跳过 agent 自己发的。
   - 读失败的条目单独记重试：404 重试（可能是暂时失去了私有仓库的访问权）；410 丢弃（已删除）；连续失败满 24 小时放弃。
   - 本轮结束时，在一个事务里把全局游标推进到“本轮开始时间减 5 秒”，同时写回重试表。
2. **全收作用域的仓库流**（`readStream`，`github.ts:203-249`）：按开关读新 Issue、新 PR、所有评论。
   - 游标推进到“最新条目时间减 5 秒”，只进不退；最远只回看 14 天，并向下取整到整点。
   - 只有游标不变时才带 `If-None-Match` 发送 ETag，取整到整点正是为了让 URL 在一小时内保持不变，从而命中 304、不消耗 rate limit。
3. **未结束工单的 PR 与 CI**：读每个工单还在跟随的 PR（顺带执行 `observePull`，可能触发关闭流程）；CI 结果做成快照指纹，指纹每变一次产生一个 CI Trigger。默认分支 CI 只在**失败集合**变化时触发，检查还在运行时沿用旧的失败集合，所以只有全部检查通过才报“恢复”。
4. **订阅线程**：有活工单在订阅，或者在最后一条消息、工单结束之后 14 天内的线程，继续按各自的游标读评论（`poll.ts:8-14`）。

**为什么游标回退 5 秒？**远端事件可能晚到，本轮进行中也可能有新的变更。回退造成的重复读取由确定性的 Trigger id 吸收：重复的 Trigger 在 Intake 返回 `duplicate`。这是“至少一次读取 + 幂等写入”的经典组合。

### 9.3 Trigger 的构造规则

| | GitHub | 飞书 |
|---|---|---|
| **id（幂等的关键）** | 评论或 review：`key('github-comment', 链接, 评论 id, updated_at)`，**编辑后会成为新的 Trigger**；Issue 或 PR 正文：`key('github-issue', 编号)`；CI：`key('github-ci', 编号或分支, 修订号)` | `key('feishu', message_id)`；实时事件、历史页、话题页三条路径算出的 id 完全一样 |
| **线程** | `github:<编号>`；默认分支 CI 是 `github:ci:<分支>` | `<群 id>:<话题根消息 id>` |
| **是不是人** | 登录名不以 `[bot]` 结尾、类型是 User、且不是 agent 自己 | `sender_type === 'user'` |
| **是否 @ 了机器人** | 正文里有 `@<agent 登录名>`（大小写不敏感，邮箱地址不算） | 私聊一律是 `direct`；群里 @ 到机器人是 `mention` |
| **自己发的** | 在读取时跳过 | 在翻译时丢弃 |

两点观察：

- **（确认）两个 Source 产出的 Trigger 都只属于一个线程。**领域模型支持“一条 Trigger 属于多个线程”，以及由此产生的线程关联（Correlation），但目前没有任何 Source 用到。跨线程的关联走的是另外几条路：正文里的 `#123` 或链接被解析成相关工单；工单通过 `bindThread` 跟随自己的 PR。
- **飞书的“一条翻译规则”。**实时事件、历史补偿、话题轮询三条路径，都先把输入适配成同一种事件结构，再走同一个 `feishuTrigger` 函数。测试断言：话题页产出的 Trigger 与实时事件产出的**完全相等**（`tests/sources.spec.ts`）。

### 9.4 PR 归属：分支命名约定

**`claimPull`（`github-pull.ts:35-50`）**认定一个 PR 属于某个工单，需要满足两个条件之一：

- 它就是工单起源的那个 PR；
- 它是 agent 账号开的、来自本仓库（不是 fork）、并且**分支名的最后一段等于工单短名**。

此外，它不能已经被另一个未结束的工单跟随。执行器开 PR 后调 `orchestrator_report(pr_number)` 主动认领。

**`observePull`**是兜底：轮询发现 agent 的 PR，并且分支名恰好匹配一个未结束的工单时，自动绑定。

PR 关闭或合并时，Source 只调用 `threadClosed(pr, 'merged' 或 'closed', 时间)`，工单是否结束由核心层判断（§7.5）。来自飞书的工单会收到一条结束通知。它的 id 由 PR 编号、渠道和结果决定，所以工单复活后再次结束也不会重复发送。

这个设计的代价是：依赖短名不冲突；fork 的 PR 永远绑不上；起源 PR 不是 agent 开的，只能靠执行器汇报来绑定。

### 9.5 写回：两种渠道的幂等方式

- **GitHub**（`github-channel.ts:18-45`）：REST API 没有幂等键，所以在正文末尾嵌入 `<!-- dsh-superagent:<消息 id> -->`，发送前分页扫描目标位置已有的评论，找到 agent 自己发的、带这个标记的，就视为已发送。回复 review 线程时，如果原评论已被删除，就退回到 PR 会话里发一条顶层评论。只有 410 和“会话已锁定”判定为永久失败（gone）；404、5xx 都重试。
- **飞书**（`feishu-channel.ts:19-30`）：用 `lark-cli im +messages-reply --reply-in-thread --idempotency-key <消息 id>` 在原消息所在的话题里回复；Markdown 交给 lark-cli 转换。消息被撤回或找不到判定为 gone；超时、进程崩溃没有错误码，就重试。

### 9.6 飞书的三个循环

`runFeishu` 并行跑三个循环（`feishu-loops.ts:18-19`）：

1. **实时事件**：启动 `lark-cli event consume im.message.receive_v1 --as bot` 子进程，逐行翻译、接收。45 秒内没就绪就报错；进程退出后重启，按指数退避等待，上限 60 秒。
2. **历史补偿**：每 60 秒一轮，按时间窗口拉取配置的群和私聊的历史。第一次遇到某个群只记录游标，不回读历史。顺带扫描最近 14 天内活跃的话题。
3. **群名刷新**：每 60 秒一次，供面板显示。

**（确认）问题：**三个循环用 `Promise.allSettled` 并行、插件入口也没有 catch。某个循环如果因为异常退出，它就静默停止了，健康状态也不更新。GitHub 侧对此有显式处理。

### 9.7 安全边界：所有 GitHub 调用都限定在本仓库

`Commands.path`（`commands.ts:94-98`）给每个 GitHub API 路径强制加上 `repos/{repo}/` 前缀，并拒绝绝对路径、带 scheme 的地址和 `..`：

```ts
private path(path: string): string {
  const outside = /^\/|^[a-z]+:|(^|\/)\.\.?(\/|\?|$)/i.test(path)
  if (outside) throw new Error('GitHub calls are restricted to the configured repository')
  return path ? `repos/${this.repository}/${path}` : `repos/${this.repository}`
}
```

连通知都改用仓库级端点。这是最小权限：即使模型被提示注入，宿主侧代码也读不到别的仓库。注意这只约束宿主自己的调用；执行器在 shell 里直接用 `gh`，不受这个限制（§12）。命令出错时，`CliError` 只携带错误码，**永不回显命令参数**，免得凭据或长文本泄露进日志。

### 9.8 本节的问题与局限

- **（确认）放弃重试的通知，之后不会被补读。**注释和 `docs/003` 都说“游标保留，之后的通知会补读漏掉的内容”。但通知路径的起点是 `max(该线程的游标, 全局游标)`，全局游标每轮都在前进，所以失败期间的评论读不回来了。只有当这个线程被订阅、且仍在 14 天窗口内时，线程轮询才会从旧游标补读。
- **（确认行为，风险是推断）**评论被编辑后会成为新的 Trigger，actor 仍是原作者。GitHub 允许有写权限的协作者编辑别人的评论，这样加进去的 @ 会被记在原作者名下。这与 Issue 正文“不读编辑”的理由相矛盾。
- **（确认）**review 类型的 Trigger 在研判窗口里定位不到原消息：会话页只包含普通评论，窗口退化为只显示前 5 条。
- **（确认）**飞书的历史补偿不覆盖“其他群只收 @”的群，服务停机期间那些群里 @ 机器人的消息会丢失。
- **（推断）**仓库流每页 100 条、不翻页。如果有超过 100 条更新集中在 5 秒之内（例如批量编辑），游标可能卡住不动。
- **（确认）效率：**GitHub 每次发送前都要扫描目标位置的全部评论；review 列表每轮全量拉取；飞书“已见消息”集合按线程无限增长。

## 10. 策略与配置：policy.ts、Settings、Setup

**一句话：**这一层回答“谁能让 agent 干活、凭什么”。核心是三个纯函数：`gate`（准入）、`revalidate`（重验证）、`policyEffects`（变更影响）；外加一个保证并发安全的配置保存流程。

### 10.1 配置结构与默认值

| 块 | 字段 | 默认值与约束 |
|---|---|---|
| `policy.rules` | `{id, name, enabled, instruction}[]` | 两条默认规则：`copy`（文案修正）、`visual`（局部视觉修正）；名称 ≤100 字符，说明 ≤2,000 字符，id 不可重复 |
| `policy.sources.<id>` | `allowed`（名单）、`admission`（准入表）、其余由 Source 自己定义（如飞书的 `syncChat`、`bots`） | 名单为空、准入表为空 |
| `eval` | `mode`、`timeoutSeconds`、`model` | `observe`；超时 120 秒（30–600）；模型为空时用默认 |
| `executor` | `model`、`maxWorkers`、`directory`、`prepare`、`blockedCommands` | 3 个工位（1–16）；目录默认 `<stateDir>/work`；prepare 命令 ≤4,000 字符；`gh`、`git`、`lark-cli` 不能被屏蔽 |
| `sources.<id>` | `enabled`、`config` | 由各 Source 的 schema 校验 |
| `paused` | 布尔 | `false` |

**来源规则：**每次保存都必须写明 origin（`panel`、`tool`、`sync`）。执行环境（`directory`、`prepare`、`blockedCommands`）和账号字段（GitHub 的仓库与 CLI、飞书的 CLI）**只能从面板改**，否则抛 `panelOnlyEnvironment` 或 `panelOnlyAccounts`（`settings.ts:24-51`）。原因是防止一段被注入的消息让主 Session 改掉 shell 路径或换掉账号。

但要注意，`policy` 和 `eval` 两个块**没有这个限制**：主 Session 的 `orchestrator_settings_save` 可以扩大名单、修改准入表、把模式切到 auto。按文档这是设计如此（应人的明确要求可以由主 Session 保存），唯一的约束是提示词。授权配置本身可以被 LLM 修改，这是一个值得意识到的风险面。

### 10.2 Gate：完整的决策表

![图 4：一条消息的两条路](img/l1-routing.svg)

**图 4：已有工单的后续消息由该工单的执行器自己裁决，不经 Eval；其余消息先过纯代码的 Gate 得到强度，再交给路由 LLM。LLM 拿不到 Gate 之外的任何授权；主动发现的工作在 observe 模式下只记录，不动手。**

**人的消息**（`policy.ts:191-211`），自上而下第一条命中即为结果：

| # | 条件 | 结果 | `basis.via` |
|---|---|---|---|
| H1 | 发送者在名单上，且 @ 了机器人或私聊 | **explicit** | `addressed` |
| H2 | 准入表对应格子设为 explicit（只有“名单外但 @ 了”这一格可以） | **explicit** | `admission` |
| H3 | 线程上已有工单（不算只为上下文关注的），且不是名单外的人私聊，且（发送者在名单上，或模式不是 off） | 名单内为 **explicit**，名单外为 **inferred** | `followup` |
| H4 | 准入表格子为 inferred，且模式不是 off | **inferred** | `admission` + `observation` |
| H5–H7 | 其他 | 拒绝：`not_a_member`、`admission_ignored`、`no_mention` | — |

**机器与 bot 的消息**（`machineGate`，`policy.ts:171-188`）：不在 bot 名单上的 bot 一律 `not_human`；名单上的 bot 最多 **inferred**（off 模式下拒绝），**永远不能是 explicit，也不能叫停工单**；没有发送者的事件（如 CI）只能通过“名单外、未 @”那一格成为 inferred 候选。

四个细节：

1. **没 @ 机器人的消息，只有在 Source“全收”这个作用域时才查准入表**，否则直接视为 ignore（`receivable`，`policy.ts:200-201`）。准入表写了 inferred，在“只收 @”的群里也不起作用。
2. **explicit 格子排在 follow-up 之前。**注释原话：“so a thread's tickets never weaken an explicit request to a follow-up”。如果反过来，线程上一旦有了工单，陌生人在客服群里的明确请求就会被降级成 inferred；off 模式下甚至会被直接拒绝。
3. **私聊线程只属于一个人。**名单外的人私聊时，不能借“线程上已有工单”获得 follow-up 资格。
4. **follow-up 不受准入表限制。**名单内的人在工单线程上不 @ 也算 explicit。工单线程本来就会被轮询，不依赖群是否全收。

### 10.3 revalidate：工单复活前重新检查授权依据

只在两种情况下调用：已结束的工单被人的消息唤醒时，以及把消息 attach 到已结束的工单时。**活着的工单不会因为配置变化被撤销。**规则（`policy.ts:213-240`）：

0. Source 已经不再监听原线程（被禁用、插件卸载、群被移除）→ 失效。
1. `followup`：explicit 永远有效（**即使发送者已被移出名单**）；inferred 要求模式是 auto，且命中的规则仍启用。
2. `bot`：bot 已不在 bot 名单 → 失效；否则同第 5 条。
3. `admission`：按**当前**的准入表和**当前**的名单重新查格子；查到的强度低于原 Mandate 的强度 → 失效。发送者后来被加进名单，会被视为“升级”。
4. `addressed`：发送者不在名单 → 失效。
5. 带 `observation`（主动发现）的 → 要求模式是 auto，且命中的规则仍启用。

举例：

| 配置变化 | 哪些工单的 Mandate 会失效 |
|---|---|
| 把 Alice 从飞书名单移除 | Alice @ 机器人开的工单、Alice 在全收群里没 @ 开的工单；Alice 的 explicit follow-up 工单**不受影响** |
| 模式从 auto 改成 observe | 所有主动发现的工单（admission、bot 带 observation 的），以及 inferred 的 follow-up 工单；explicit 工单不受影响 |
| 客服群“名单外、@ 了”这格从 explicit 改成 inferred | 陌生人在那里开的 explicit 工单（强度从 2 降到 1）；之后把这个人加进名单，又会恢复有效 |

### 10.4 assessmentOutcome：模型给判断，代码出结论

（`policy.ts:40-61`，§11.3 已介绍上下游）校验顺序：

1. 每条启用规则必须**恰好**表态一次，否则拒收；
2. 不是 accept 的决定原样返回，附上审计信息；
3. 方向未定（`implementation_requested` 为假、`target_basis` 是 undetermined、或需要设计决策）→ ignore；
4. 没有命中的规则 → ignore；
5. accept 缺 `intent_evidence` → 拒收；
6. inferred 且模式不是 auto → ignore，附 `would_accept: true` 和 `proposal`；
7. 否则 accept，带上命中的规则。

Engine 在 `decide` 时还会再检查一次 `admitsInferred`，即使有人绕过这个函数直接调 Engine，也接不了不该接的单。

### 10.5 policyEffects：配置变了，告诉谁、怎么说

对每个**未结束的**工单计算影响（`policy.ts:144-165`）：

- **规则层面：**它命中的规则被禁用，或文本被修改（附新文本）。
- **授权层面：**旧配置下 `revalidate` 成立、新配置下不成立时，构造一个**反事实**：新配置，但把模式和规则换回旧的（`asBefore`）。在反事实下仍不成立，说明是“人或作用域”的问题，报 `requesterAllowed: false`；在反事实下成立，并且是 auto 切换到非 auto，报 `proactive: false`。

为什么用反事实？因为 `revalidate` 只是一个布尔值，混合了名单、准入表、作用域、模式和规则多个因素。把其中两个固定住，就能把失败原因拆开。执行器收到的提示因此不同：“你的请求人已无权发起工作”和“主动工作不再被接受”，后续做法也不同（原文见 §12）。代价是两个因素同时变化时，只报告前一个。

**策略版本**只哈希规则、模式和各 Source 的策略部分（`policy.ts:116-118`）。模型、并发数、暂停、轮询间隔都不影响“谁能发起工作”，改了也不会中断进行中的评估。代价是：Source 配置的变化（例如把某个群移出监听）不改变版本，**不会产生任何通知**，但相关工单下次复活时会在 revalidate 第 0 条失效。

### 10.6 保存流程：两次冲突检查加一个事务

`Settings.save`（`settings.ts:148-196`）：

1. 按**块**检查修订号，不用全局修订号。面板上的多张配置卡、主 Session、名单同步会同时改不同的块。
2. 校验 schema，检查模型是否可用，检查来源限制，账号字段变化时探测账号。这几步有 await。
3. **再检查一次修订号**，因为上一步的 await 期间可能有人先保存了。
4. 重新读取当前配置，只合并本块。
5. 没有变化就直接返回，不产生新修订。
6. 计算前后策略版本；版本变了就计算 `policyEffects`。
7. **一个事务里**完成：写快照并推进指针；通知主 Session（工具发起的保存除外）；版本变了就调 `engine.policyChanged`，给受影响的工单投 `policy-changed` 输入，并作废所有进行中的评估、重新排队。
8. 账号字段变化，或 setup 还没完成时，**整体重启插件**。原因是部署信息散布在闭包、命令 shim、git 配置和 `process.env` 里，热更新要逐个替换。重启用 `setTimeout(…, 0)` 延后，保证调用方先拿到“正在重启”的响应。

第 4 到 7 步之间没有 await，在单线程的 Node 里，“检查后写入”这一段是原子的。

**名单同步（例如从飞书群同步成员）主动让路：**同步前记下 policy 块的修订号，等外部名单返回后发现修订号变了，就放弃这次同步，按 1 分钟起、上限 1 小时的指数退避重试。人在面板上的编辑始终优先于后台同步。

### 10.7 Setup：像执行器的 shell 一样去探测

- **探测方式：**`workerProbe`（`deployment.ts:72-76`）用 DSH 的子进程服务、去掉凭据类环境变量、把 worker 的命令 shim 移出 PATH，再去执行 `gh` 和 `lark-cli`。这样探测看到的，就是执行器真正会看到的环境。
- **GitHub 侧的检查顺序：**仓库已配置 → CLI 存在 → 已登录 → 能读仓库 → **有 push 权限**。仓库名取 GitHub 返回的 `full_name`，改名后的仓库也能对上通知。
- **飞书是可选的：**飞书检查失败只降级为一条警告，不阻塞启动。
- **服务脚本**（`scripts/dsh-superagent`）的退出码：0 正常；1 没在跑、跑了两份或运行时报错（重启可能修好）；2 某个部件报错（重启修不好，要人处理）；3 等待配置。只有人为的 `stop` 会写“手动停止”标记，`restart` 不写。另一个 profile 里每小时运行的看门狗用 `--respect-stop restart` 处理退出码 1，人为停止会被尊重。

### 10.8 本节的问题与局限

- **（确认）**名单内的人发起的 explicit follow-up 工单，在这个人被移出名单后**仍然有效**，也不会收到通知。这与 `addressed` 的处理不对称。
- **（确认）**只是显示名变了（飞书同步会原地更新成员名字）也会改变策略版本，从而作废所有进行中的评估。
- **（确认）**账号 shim 的重写发生在事务提交之后，失败时配置已保存，调用方却收到错误。
- **（推断，较高置信）**保存时写进历史记录的策略版本，用的是**未经 schema 规范化**的合并结果；运行中的 Policy 用的是规范化后的配置。如果规范化改变了 JSON 的键序（Source 策略部分的字段顺序可能不同），两个版本号就对不上，面板按工单的策略版本去历史里查会查不到。没有运行验证。
- **（确认）**设置历史不裁剪，`Versions.list` 每次读出全部快照再排序。

## 11. 主 Session：工具面、研判与提示词系统

**一句话：**主 Session 是一个常驻的 LLM 会话，只做分诊。它能做什么，由 14 个工具定义；它怎么想，由一套版本化的提示词约束；它能看到多少，由“卡片”和“研判窗口”两种有界投影控制。

**涉及文件：**`src/app/tools/main.ts`（工具）、`src/app/assessment.ts`（MasterAssessor）、`src/sources/admission-context.ts`（研判窗口）、`src/app/prompts.ts`（提示词拼装与版本）、`src/app/resources.ts`（`orchestrator_read` 的资源树）、`src/app/cards.ts`、`src/app/links.ts`（给模型看的卡片），以及 `prompts/*.md`。

### 11.1 14 个工具，以及它们落到 Engine 的哪里

| 工具 | 调用 | 被拒的典型情况 |
|---|---|---|
| `orchestrator_read` | 读资源树（不经 Engine） | 未知资源；执行器去读主 Session 专属的资源 |
| `orchestrator_route` | `decide`（accept、stop、ignore）、`attach` | 见 §11.2 |
| `orchestrator_ticket_retry` | `retryTicket` | 不可重试（`canRetry` 为假） |
| `orchestrator_ticket_request_status` | `requestStatus` | 执行器最近一次报告不是 `waiting`，就不发 |
| `orchestrator_ticket_note` | `note`（≤100 词，不带任何授权） | 工单已结束 |
| `orchestrator_trigger_requeue` | 先检查 `followupOwners`，再 `reevaluate` | 消息仍归某工单裁决（`workerDecides`）——这个动作只能在面板上做 |
| `orchestrator_eval`、`_eval_read`、`_eval_decide` | MasterAssessor（研判三件套） | 见 §11.3 |
| `orchestrator_reply` | `reply`（answer、result、attention） | `Replies` 的用途规则，见 §7.7；飞书消息再过一遍 100 词上限 |
| `orchestrator_thread_unsubscribe` | `unsubscribe` | — |
| `orchestrator_settings_save`、`_policy_sync`、`_prompts_save` | Settings、Prompts | 版本冲突；工具来源不能改执行环境和账号字段 |

另外，主 Session 还有 DSH 自带的 `skill`、`bash`、`read`、`read_image`、`job_*`。“不读代码”“研判只读”这两条只是**提示词约束**，工具层面没有拦（`sessions.ts:263` 放行了 `bash` 和 `read`）；真正由工具层强制的是 `noWait` 守卫（§12.6）。

### 11.2 `orchestrator_route`：一次调用路由多张卡片，逐条独立

主 Session 一轮里常常同时积压几张候选卡片。路由工具允许一次传 1–20 条决定（`main.ts:125-153`），每条单独作为一个 Engine 事务执行：

```ts
const results = decisions.map(item => {
  const head = { trigger_id: item.trigger_id, kind: item.kind }
  const problem = invalid(item)
  if (problem) return { ...head, refused: problem }
  try {
    const result = route(host, item, decided)
    ...
  } catch (error) {
    if (error instanceof UserError) return { ...head, refused: error.message }
    return { ...head, refused: `Not recorded: ${...}; retry this item` }
```

几个设计点：

- **schema 只约束形状，字段要求按 kind 在运行期检查。**`FIELDS` 表（`main.ts:61-68`）声明每个字段“哪些 kind 会读、哪些 kind 必填”：accept 必填 `title`、`worktree_name`、`acknowledgement`；attach 与 stop 必填 `ticket_id`；ignore 必填 `reason`。不属于本 kind 的字段写成 null 也会被忽略。原因是 JSON schema 表达不了“某 kind 需要某字段”，而要求模型为不相关的字段填值只会增加出错机会。
- **同一调用内的批次互斥。**前一条 accept 了某条消息，它所在的整批都已被那个 Decision 带走，后面再 ignore 同一条消息会被拒绝（`main.ts:92-93`）。
- **accept 的幂等提示。**如果 Decision 的 id 与调用前去向记录里的一样，说明是重放，返回“已接受，标题和确认语不变”（`main.ts:101-103`），而不是报错。
- **路由器写不了范围。**显式 accept 时 `scope` 直接取 `title`（≤200 字符），备注 `note` ≤30 词，派工文本里会标明“Routing note (not the source request)”。结构上就让路由器没地方改写需求。
- **推断型候选不能用 route 接受。**route 不传 rules，`admitsInferred([])` 恒为假，Engine 抛 `notExplicit`，错误信息直接告诉模型去用 `orchestrator_eval_decide`。
- **一个小的不一致：**`attach`、`requeue`、`unsubscribe` 都包在 `quietly` 里，避免把主 Session 自己的操作再通知回自己；`stop` 没包，所以主 Session 会收到自己触发的 `ticket_ended` 通知。测试里显式过滤了这类通知（`tests/tools.spec.ts`），可能是有意保留的回执。另外，schema 层面的错误（缺 `trigger_id`、多余字段、超过 20 条）仍会让整次调用失败，与工具描述里“a refused item leaves the others”不完全一致。

### 11.3 研判：模型给判断，代码给结论

研判只用于 inferred 候选（主动发现的工作）。完整链路：

1. **投递后才计时。**`MasterAssessor` 注册的 `assessment` effect 处理器（`assessment.ts:94-102`）先确认 Eval 仍是当前那个，再以 steer 方式把候选卡片投进主 Session，**投递成功后**才调 `engine.candidateDelivered`。主 Session 是串行的，忙的时候卡片排队不消耗研判时间。代价是：没投递出去的 Eval 永不过期，线程会一直被占着，这依赖 effect 队列的持久化和幂等投递（同一 `candidate:<evalId>` 只投一次）。
2. **候选卡片**（`links.ts:55-71`）包含：主消息（正文 ≤600 字符，超出给读取指针）、回复的上一条、强度与接单模式、同批其他消息、被交回的理由、本线程的相关工单（≤8，未结束的在前）、已经送达了哪些工单、正文里提到的 `#123` 或 URL 对应的工单（≤4），以及一句“下一步该做什么”（显式和推断两种文案）。
3. **`orchestrator_eval` 返回有界窗口**（`assessment.ts:183-219`）：
   - 正文 ≤4,000 字符；
   - 线程的前 5 条，加触发消息前后各 5 条；
   - 前后各两条近邻给 1,200 字符，窗口内其他消息给 400 字符；
   - 窗口外已加载的消息只给 50 字符的存根；
   - 触发消息本身只标 `trigger:true`，不重复正文；
   - 图片只给 id，要读就调 `eval_read`。
   - 线程按页加载，默认最多 10 页，并按 evalId 缓存，Eval 结束时 `release`。
4. **`orchestrator_eval_read`** 只认窗口里出现过的消息 id、发放过的游标和图片 id，还能读仓库文件和目录。模型没法编造一个 id 去读任意东西。
5. **`orchestrator_eval_decide`** 的 schema（`assessment.ts:21-34`）：`decision`、每条启用规则的 `rule_assessments[{rule_id, matched, reason}]`、`direction{implementation_requested, target_basis, requires_design_decision, evidence}`，以及 `intent_evidence`、`acknowledgement`、`worktree_name`、`title`、`scope`、`reason`。
6. **`assessmentOutcome`**（`src/policy.ts:40-61`，§10.4 详述）把这些判断转成结论：规则没覆盖全就拒收；方向未定转成 ignore（`directionUnsettled`）；没有规则命中转成 ignore（`noRuleMatched`）；observe 模式下转成 `ignore + would_accept + proposal`。
7. **`Engine.decide`** 再查一遍 evalId、幂等、裁决归属、`admitsInferred` 和策略版本（§7.4）。

这条链路的取舍：**主 Session 自己研判，不另起评估 LLM**。提示词原文是“do not delegate it to another Session or evaluator”。好处是复用常驻上下文和缓存，研判规则也只写在一处（`prompts/admission.md`；`orchestrator_eval` 返回的是上下文，不重复说明）。代价是主 Session 的上下文要装下窗口和图片，所以才需要窗口、存根、缓存与 release 这一整套。另一个代价是 ignore 和 unavailable 也必须填完整的 direction 与 rule_assessments，token 开销不小。

> **一个风险（推断）：**窗口外的消息都会生成存根。GitHub 每页 100 条、默认 10 页，大线程可能产生上千条存根；测试只覆盖了 30 条消息的情形。

### 11.4 系统提示词怎么拼

`promptParts`（`prompts.ts:102-116`）按固定顺序拼出两种角色的系统提示词：

| 顺序 | 主 Session（orchestrator） | 执行器（worker） |
|---|---|---|
| 1 | `runtime/shared`、`runtime/orchestrator` | `runtime/shared`、`runtime/worker` |
| 2 | 团队段：Communication、Ticket scope、Delivery requirements | 同左 |
| 3 | `runtime/subscriptions`、`runtime/replies`、`runtime/dispatch` | `runtime/subscriptions`、`runtime/replies`、`runtime/work` |
| 4 | `runtime/admission`（研判规则） | — |
| 5 | `runtime/handoff` + `handoff/orchestrator` | `runtime/handoff` + `handoff/worker` |
| 6 | `runtime/environment`（部署级变量） | `runtime/environment`（本工单的变量） |
| 7 | `runtime/policy`（**每次请求实时生成，放在最后**） | — |

再由 `Sessions` 包一层（`sessions.ts:244-249`）：开头加 `[dsh-superagent prompt revision N]`，执行器末尾再加 `mandateInstructions`。主 Session 的段标了 `complete: true`，意思是完全替换 DSH 的默认提示词；执行器则保留 DSH 的编码提示词和仓库的 `AGENTS.md`。

几个细节：

- **Policy 段为什么放最后。**它每次请求都从当前配置重新生成，规则一改就生效；放在最后，修改不会动到前面的文本。注释原话是“which nothing follows, and whose change leaves the rest unchanged”。推断：这也让提示词前缀保持稳定，便于前缀缓存。每条规则被压成单行 `- name (id): instruction`，规则内容没法伪造出新的标题。
- **模板占位符只替换一遍**，并且用 `Object.hasOwn` 查值（`prompts.ts:80-81`）。规则文本里出现 `{mode}` 也不会被二次替换，原型链上的名字也填不进去。
- **版本与会话绑定。**每次模型请求都读取最新版本，并把用到的版本号记进 `KV.sessions.prompts(id)`。执行器只能读当前版本和自己用过的版本（`mayRead`）。
- **“Default updated”。**团队覆盖某个内置部件时，记下内置原文的哈希作为 `base`；升级后内置文本变了，面板会提示“默认内容已更新”，并给出新旧对比，由团队选择保留自己的文本还是改用新默认。覆盖文本与内置文本相同时自动删除，重新跟随默认值。

### 11.5 提示词在防什么：八条关键约束

| 约束（原文摘录） | 防止的失败模式 | 代码里的配合 |
|---|---|---|
| “Routing does not authorize rewriting the request”（`shared.md`） | 传话失真，路由器替人编需求 | scope 取 title；备注 ≤30 词并标注“not the source request” |
| “never wait for them with sleep, polling or job waits; end the turn when nothing is left to do”（`orchestrator.md`） | 主 Session 被阻塞，新候选积压 | `noWait` 工具守卫 |
| “Attach only when the message points at that ticket explicitly … a wrong attachment subscribes the ticket to the whole thread and stays”（`dispatch.md`） | 误挂，而且误挂的副作用不可逆 | `attach` 拒绝挂回交回它的工单 |
| “act on a follow_up only to stop the work … never accept, attach or requeue it”（`dispatch.md`） | 同一条消息被两方处理 | `leftToWorkers`、`workerDecides` |
| “establish three independent facts … When intent remains uncertain, ignore quietly”（`admission.md`） | 主动越权，或打扰人确认闲聊 | 研判 schema 强制 direction 与 intent_evidence |
| “Include negative judgments as matched=false”（`admission.md`） | 理由说不适用，却标 matched=true | 必须对每条启用规则恰好表态一次 |
| “Unsolicited IM must require human attention … if nothing, send nothing”（团队 `communication.md`） | 进度刷屏 | 回复用途的前置条件；100 词上限 |
| “Address at most three review rounds per PR from each automated reviewer”（团队 `communication.md`） | 无休止地追着 bot 的建议改 | 轮数记在报告里，跨重启、跨执行器延续 |

### 11.6 测试怎么守住“模型看到的东西”

这是整个仓库最有特色的测试之一。`tests/model-surface.spec.ts` 用一个假的 DSH 宿主，把完整业务流程跑一遍，收集两种角色看到的**所有**文本：提示词段、工具描述、schema 描述、skill、错误信息、输入卡片。然后检查：

- **体积预算：**提示词加工具定义，主 Session 不超过 59,000 字符，执行器不超过 40,500 字符。注释写着“只在有意时调高，并在提交信息里说明原因”。
- **只引用存在的东西：**提到的 `orchestrator_*` 工具必须已注册；`name=value` 里的 value 必须是 schema 的枚举值；`DSH_SUPERAGENT_*` 环境变量必须真实存在；资源路径必须能匹配 `RESOURCES`。
- **术语一致：**禁用旧术语（reopen、task/ 等）；描述工单状态必须用卡片状态的封闭集合。
- **跨角色引用：**只有一种角色会读到的文本，提到另一角色的工具时，同一句里必须点名那个角色。
- **数字一致：**文中写“N words”必须是 100，提到 routing 的必须是 30。

`tests/cards.spec.ts` 还有约 50 条“某个提示词文件必须包含某句原话、不得出现某句旧话”的断言，并要求主 Session 和执行器各自的角色说明分别短于 1,400 和 1,000 字符。

**本节小结：**主 Session 的设计思路是“让 LLM 只做语义判断，其余都用结构约束”：工具 schema 加运行期字段表限制它能说什么，有界卡片和窗口限制它能看什么，代码推导结论限制它能决定什么，测试守护它读到的每一个字。代价是提示词和工具描述高度耦合，改一个词都可能牵动 `model-surface` 和 `cards` 两组测试。

## 12. 执行器：一工单、一会话、一目录

**一句话：**DshExecutor 把每个工单映射成一个常驻的 DSH 会话（`worker-<ticketId>`）和一个工作目录。它管理工位并发、后台工作、瞬时故障重试和重启恢复，并把 DSH 的会话事件翻译成 Engine 能理解的事实。

**涉及文件：**`src/executor/dsh.ts`（673 行）、`src/executor/background.ts`、`src/app/sessions.ts`、`src/app/delivery.ts`、`src/app/tools/executor.ts`、`src/infra/environment.ts`、`src/app/role-models.ts`，以及 `cordis.patch.yml` 里的两个 preset。

### 12.1 工位：算出来的，不是加锁锁出来的

`WorkerSlots`（`dsh.ts:171-199`）自己只保存一样东西：“正在启动”的预留，类型是 `Map<ticketId, 'launch' | 'wake' | 'person'>`。“正在运行”不存储，而是每次实时计算：在所有未结束的工单里，有运行中的派工，并且会话正在跑、或者正在等后台工作结束的，就算占着工位。

```ts
used(): number { return new Set([...this.running(), ...this.starting.keys()]).size }
launches(): number {
  const wakes = [...this.starting].flatMap(([ticketId, by]) => by !== 'launch' ? [ticketId] : [])
  return this.max() - new Set([...this.running(), ...wakes]).size
}
reserve(ticketId: string, by: 'launch' | 'wake' | 'person'): void {
  if (this.starting.has(ticketId)) return
  if (this.used() >= this.max() + (by === 'person' ? 1 : 0)) throw new Later('No worker slot is free')
  this.starting.set(ticketId, by)
}
```

- **为什么不用信号量？**一个工位要覆盖三种状态：turn 在跑；turn 结束了但后台 job 或 subagent 还在跑；正在启动。这三种状态之间要跨越多次 await，还要经得起插件在进程内重启。实时推导不会泄漏工位，代价是每次分发都要扫一遍未结束的工单。
- **给人留一个额外工位。**人的消息唤醒执行器时，可以用到第 max+1 个工位。全局只有这一个额外工位，因为它本身也计入 `used()`。新工单的启动（launch）拿不到它，所以新工作永远挤不掉“人在推进一个已有工单”。
- **防止两个启动抢最后一个工位：**`launch`、`delivery` 和瞬时故障续跑这三个入口，都在第一个 await **之前**同步调用 `reserve`。Node 是单线程的，检查和写入天然原子；工位满了就抛 `Later`，Effects 推迟重试，不计失败。

### 12.2 启动：每个 await 之后都重新确认“我还是当前派工吗”

`launch()`（`dsh.ts:424-470`）的步骤：

1. 定义 `current()`：插件正在关闭就抛错；处理暂停就抛 `Later`；否则返回“工单还活着，且这次派工仍是最新的”。**之后每个 await 之后都调一次 `if (!current()) return`**，所以人在启动途中叫停或重试，启动会自动放弃。
2. 目录不存在时跑 prepare 命令：用 `sh -c`，带工单的环境变量，限时 10 分钟。失败时删掉空目录，并标记为不自动重试（`PrepareFailed`）。成功后检查目录的 git 配置（§12.6）。
3. 确保会话存在（`sessions.ensure`，使用执行器 preset 和执行器模型）。
4. **等待旧派工的后台工作停止。**停止最后会取消 turn，并清空 inbox；不先等它，就会把这次的派工消息一起清掉。
5. **从 inbox 里移除残留的副本**（`dsh.ts:446-448`）。重启时取消 turn 保留了 inbox，崩溃后持久化的 inbox 也会被恢复，里面可能还躺着上一次投递的输入；它们会被 `requeueUntaken` 重新投递一次（§7.5）。先删掉旧的，**重新投进去的那一份就是唯一的一份**。
6. 发派工消息。首次派工发完整 JSON（标题、≤1,500 字符的派工文本、原始 Trigger 卡片、同一批的其他消息）；之后的派工只发 `{ticket_id, assignment: 序号}`，前提是会话里还保留着首条消息。用 queue 方式发送，开一个新 turn。
7. 消息进入 inbox 的同一时刻（`accepted` 回调），调 `assignmentStarted` 把派工标成 running。这样工位交接没有空档；如果第一个 turn 结束得非常快，idle 事件也不会因为“没有运行中的派工”而被吞掉。

### 12.3 投递与“读到”

- **投递**（`deliverQueued`，`dsh.ts:290-328`）：一次投一条，以 steer 方式发给执行器。执行器空闲时，steer 会开一个 turn；正在跑时，消息会在**下一个 step 边界**被吸收。每条输入渲染成卡片；正文超过 1,500 字符时只给前面一段和读取指针。执行器空闲时先预留工位：待投递的输入里只要有一条是人写的，就用给人留的那个额外工位。
- **“读到”以写入历史为准**（`dsh.ts:495-504`）：只有 DSH 的 `session/event` 里出现 `user/message`，才调 `inputsRead`。为什么不看 inbox 的“已认领”？因为 step 认领了消息后可能被取消或失败，消息并没有进入模型历史。这样，“投递”和“读到”就分成了两件事（§7.6）。

### 12.4 turn 结束之后：后台工作、交接与 30 分钟上限

DSH 的 turn 结束（`agent/status: idle`）不代表工作结束。`workerIdle`（`dsh.ts:602-672`）分两种情况：

- **交接给人**：最新报告是 ready、有一个待人回答的问题、或者工单在“等人”（blocked、failed）。这时只停掉 subagent，**保留 shell job**（例如给人试用的 dev server），并且这些 job 结束时不再唤醒执行器；然后**立即释放工位**。
- **还有后台工作在跑**：保留工位，等 job 和 subagent 都结束（`untilQuiet`）。它们结束时，DSH 会唤醒执行器继续工作。等待最长 30 分钟，超时就杀掉 job，并报告 waiting，附上“哪些 job 被杀了”的说明。subagent 不停，因为停它会把结果送回来，又唤醒刚被报告为 waiting 的执行器。

执行器 preset 的 shell 设置（`cordis.patch.yml`）是这一切的前提：**前台命令超过 5 秒自动转成后台 job，等待 job 最多 5 秒**。长命令不会卡住 step，人的新消息就能在下一个 step 边界进来。subagent 用 DSH 的 standard preset（60 秒），因为 subagent 结束时它的 job 会被杀掉。

### 12.5 失败、重试与恢复

- **瞬时故障**（`transient`，`dsh.ts:50-55`）：HTTP 408、429、5xx，以及连接被拒、重置、超时、DNS 失败等。启动失败走 Effects 的退避（5 秒起，上限 300 秒）；turn 失败则在 15、30、60 秒后发一条“继续”消息。每种最多 3 次，第 4 次记为 failed。
- **非瞬时失败**：在一个事务里写 failed 报告并转达给主 Session。工单进入“等人”，CI 和 bot 消息只记录、不投递（§7.5 的 `addInput`）；工具仍然可用，人可以直接进会话接手。
- **重启恢复**（`resume`，`dsh.ts:524-537`）：按工单创建时间从早到晚，对每个运行中的派工取消 turn（保留 inbox）、停掉后台工作，然后调 `assignmentInterrupted`（§7.6）。恢复说明的原文：

  > “The host restarted during this ticket. Inspect the current worktree, git history and GitHub state before continuing. Do not repeat an already completed external operation. Finish the original assignment and report its status.”

- **停止**（`stop`，`dsh.ts:357-370`）挂在 `afterCommit` 上：结束工单的事务如果回滚，执行器不受影响。执行器自己用 ready 或 dropped 结束工单时，不取消它当前的 turn。兜底的是工具守卫：工单一结束，执行器的**所有**工具调用都被拒绝（“Ticket ended; new work needs a new authorized ticket”）。

### 12.6 执行器看到的世界：提示词、工具与环境

**它收到的几类输入文本**（`inputMessage`，`dsh.ts:156-164`）：

- **Trigger 卡片**：JSON；只为上下文而关注的线程上的消息带 `context: true`。
- **工单复活时**（`RESUMED`）：

  > “This ticket ended earlier and the message that follows, on its thread, resumed it under the same mandate. If it asks for more within the original scope, inspect the worktree, git history and PR state and continue; if it asks for work beyond that scope or for another ticket, hand it back … and report dropped; if it needs nothing, such as thanks, report dropped without a message.”

- **策略变更**（`policyMessage`）：按影响分别说明。规则被禁用：“finish only a change within it, or report dropped”；主动工作不再被接受：“finish only what is in progress … take on nothing new”；请求人失去权限：“take no new requests from them on it”。
- **主 Session 的便条**：前缀写明 “Note from the orchestrator, for context only”。
- **主动发现的工单**，system prompt 末尾追加 `mandateInstructions`：

  > “This ticket has limited proactive authorization. Independently verify that it is a small change internal to one module within the accepted rule scope. If it requires public-interface or behavior changes, new interaction, product decisions or clarification, stop and report dropped; do not expand scope.”

**工具：**6 个应用工具（`read`、`ack`、`report`、`thread_unrelated`、`thread_follow`、`trigger_handback`），加上 DSH 工具的白名单 `WORK_TOOLS`（文件、搜索、shell、job、web 搜索、skill、todo、subagent）。不包括向人提问、展示界面、改 profile、调度、goal、workflow 这类工具。`orchestrator_report` 在一个事务里完成三件事：认领 PR、记录报告、转达给主 Session。PR 认领被拒绝，整个调用失败，不会留下半截状态。

**主 Session 的防阻塞守卫**（`noWait`，`sessions.ts:77-86`）：拒绝出现在命令位置的 `sleep`、超过 30 秒的 shell 超时、超过 5 秒的 job 等待。理由写在 `docs/004`：“a blocked step delays new candidates”。正则是启发式的（`bash -c "sleep 60"` 这类写法能绕过），真正的兜底是路由 preset 的 5 秒 shell 上限。

**进程环境**（`src/infra/environment.ts`）：

- 清掉 `GH_TOKEN` 和 `GITHUB_TOKEN`；
- git 使用隔离的全局配置：凭据只通过 `gh auth git-credential` 获取；SSH 地址改写为 HTTPS；不允许 git 猜测提交身份；
- `gh` 和 `lark-cli` 被换成 shim，绑定 agent 的账号；
- 屏蔽的命令也做成 shim；
- 每个工单目录都检查仓库本地配置里没有 `credential.*` 和 `http.*extraheader`（`verifyDirectory`）。

**模型选择**（`role-models.ts`）：给新会话追加一条 `model/selection` 事件，而不是调用 DSH 的 `selectModel`，因为后者会改全局默认。所以改执行器模型只对新建的会话生效。

### 12.7 本节的问题与局限

- **（确认，安全相关）重试会绕过工作目录的 git 配置检查。**`prepareDirectory` 先建目录，再 `verifyDirectory`（`dsh.ts:259-261`）。检查失败抛的是普通错误，目录已经存在；人点“重试”后，`launch` 看到目录存在就**跳过 prepare 和检查**（`dsh.ts:435-438`）。检查的本意是拒绝仓库本地配置里的 `credential.*` 和 `http.*extraheader`，第二次启动就放行了。
- **（确认，配置）subagent 能用 `web_fetch`，执行器本身不能。**`WORK_TOOLS` 包含 `web_fetch`，执行器 preset 关掉了 fetch；而 subagent 跑在 DSH 的 standard preset 上，那里 fetch 是开的，白名单按 subagent 自己的父作用域过滤，于是放行了。
- **（确认）subagent 的限制有覆盖缺口。**`limitSubagent` 最多向上找 8 层祖先，并要求中间的父 agent 都已加载；找不到执行器祖先，就完全不做工具限制。
- **（确认）**执行器自己报告 ready 结束工单时，本意是不打断它当前的 turn；但只要还有 subagent 活着，停止 subagent 的流程会取消这个 turn。
- **（确认，设计如此）**执行器以 `danger-full-access` 运行，不切换 OS 用户，不隔离文件系统和网络；HOME 不变，`~/.ssh` 可读。shim 和屏蔽命令可以用绝对路径绕过。宿主侧“只能访问配置仓库”的限制，不约束执行器在 shell 里直接用 `gh`。
- **（推断）**flush 失败时，派工已经被标成 running、turn 也已经在跑，但启动流程会把工单记为 failed。

## 13. 管理 API 与面板

**一句话：**一个只监听本机的管理 API，加一个注入 DSH 侧栏的 React 面板。面板只回答三个问题：现在有什么需要我？系统健康吗？某项工作的来龙去脉是什么？

### 13.1 管理 API：25 条路由，类型在两端共享

- **类型契约**（`src/app/api.ts`）：以 `'VERB path'` 为键，响应类型直接写成 `ReturnType<typeof 视图函数>`，路径参数用模板字面量类型从路径里抽出来。面板只 `import type`，不会把服务端代码打进浏览器包。改了视图函数，面板的类型会立刻跟着变。代价是没有运行时的响应校验。
- **安全模型**（`http.ts:75-79、231`），三层：
  1. 只挂在 `127.0.0.1` 的 Web 服务器上，否则注册时直接抛错；
  2. `Host` 头必须精确等于 `127.0.0.1:<端口>` 或 `localhost:<端口>`，防 DNS rebinding；
  3. 必须带自定义请求头 `x-orchestrator-request: 1`，Origin 要么缺省、要么同源。跨源网页加不了自定义头，因为需要预检，而服务端不回 CORS 头。服务脚本只需要带这一个头。

  信任边界因此就是“本机”。请求体默认上限 128 KiB；响应带 `no-store` 和 `nosniff`。
- **错误映射：**参数校验错误 400（附字段路径）；setup 错误 400（附问题码）；`UserError` 中 `missing` 类 404、其余 400（附 code，面板据此本地化）；修订冲突 409；其他 500。

### 13.2 视图：计数必须等于它链接的列表长度

- **“需要人处理”** = 未结束工单中 `ticketStage` 为 person 的那些。这是 §7.9 那 10 行函数的直接应用。
- **“系统问题”**依次是：setup 未完成、没有可用的 Source、名单为空、Source 报错、处理已暂停、运行时报错、评估超时、评估 unavailable（每个线程只取最新一条、未被忽略、且线程不在排队或评估中）、发送失败的消息。
- **24 小时计数**统一按 **Trigger 的到达时间**计算：概览展示的是“同一批到达的消息”从接收、过滤、评估到接单的漏斗，所以“已接单”这个计数能和它链接到的决策列表对上（两边用同一种 SQL 口径）。
- **问题（确认）：**被过滤消息的清单每个 Source 最多显示 5 条，而概览的计数没有上限。严格说做到的是“计数等于各 Source 计数之和”，而不是“等于列表长度”。“加载更多”会卡在服务端的上限：决策最多 200 条，工单和线程最多 1,000 条。

### 13.3 面板的几个工程细节

- **轮询**（`client/api.ts:67-98`）：请求串行，不重叠；数据没变就不触发渲染；出错时保留旧数据；标签页隐藏时间隔乘 6。默认 5 秒一次；插件重启期间切到每秒一次，直到服务恢复或 30 秒超时。
- **空 body 的 404 表示“服务没在跑”。**DSH 重启时会卸载路由，返回空 404；dsh-superagent 自己的响应总是带 JSON body。所以重启期间不会误报“工单不存在”。
- **配置草稿的三方合并**（`client/draft.tsx`）：
  - 服务端用块级修订号做严格的乐观锁；
  - 客户端收到 409，或轮询发现修订号变了，就用 `merge3(base, 我的, 服务端的)` 把草稿 rebase 上去；
  - 带 `id` 的列表（规则、名单、飞书群）按元素合并；
  - 冲突时保留“我的”，并标出冲突路径。

  理由是主 Session 的工具、名单同步、多个浏览器标签页都可能同时写同一块配置。
- **保存后先等修订号回读。**保存成功后不立刻显示服务端的值，而是继续显示刚提交的值，直到轮询读到的修订号不小于这次保存的修订号。保存可能触发重启，重启期间读取会失败很久；测试要求“重启期间刚保存的 CLI 仍留在屏幕上”。
- **Markdown 安全。**面板里没有 `dangerouslySetInnerHTML`；Markdown 子集全部渲染成 React 文本节点；链接只接受 `http(s)://`。服务端对问题和输入文本先去掉 HTML（去两遍，中间解一次实体）。残余风险（推断，低）：少数直接使用服务端 URL 的链接组件没有做协议过滤，目前这些 URL 都来自 GitHub 和飞书。

## 14. 测试体系：59 个测试文件守护的是什么

测试代码约 1.5 万行，和源码量相当。按“守护什么”分成六类：

| 类别 | 代表文件 | 守护的性质 |
|---|---|---|
| 领域规则 | `core-engine.spec.ts`、`stage.spec.ts`、`followups.spec.ts`、`policy.spec.ts` | 在内存仓储上验证全部领域规则；阶段表逐行；准入重验证用 288 种组合对照一个参考函数 |
| 持久化与副作用 | `effects.spec.ts`、`store.spec.ts`、`outbox.spec.ts`、`delivery.spec.ts`、`migration.spec.ts` | 回滚后不执行；同 key 合并；退避序列；同工单同会话保序；重启后只投一次；迁移失败数据库不变 |
| 并发与竞态 | `runtime-concurrency.spec.ts`、`passes.spec.ts`、`settings.spec.ts` | 工位与给人的额外工位；pass 合并；配置保存与账号检查、名单同步之间的竞态；停止只在提交后生效 |
| 模型可见面 | `model-surface.spec.ts`、`cards.spec.ts`、`prompts.spec.ts`、`tools.spec.ts` | 体积预算；引用的工具、资源、环境变量真实存在；术语一致；约 50 条“必须出现或不得出现”的原句断言 |
| 集成 | `runtime.spec.ts`、`sources.spec.ts`、`commands.spec.ts` | 在模拟的 DSH 宿主上走通工具和 HTTP；Source 的翻译、游标、PR 归属；CLI 调用限定在配置的仓库 |
| 文档与代码卫生 | `docs.spec.ts`、`class-docs.spec.ts`、`locales.spec.ts`、`controls.spec.tsx` | 编号文档只能引用排在前面的；架构文档的模块表与测试表必须列全；每个类都有 JSDoc；中英文案的占位符一致 |

**两个值得借鉴的做法：**

- **文档也被测试。**`tests/docs.spec.ts` 要求 `docs/007-architecture.md` 的模块表和测试表覆盖每一个源文件和测试文件，编号文档只能引用编号更小的文档。文档和代码因此不会脱节，读者也可以按编号顺序阅读而不遇到未定义的概念。
- **内存仓储与 SQLite 的语义对齐被测试。**`store.spec.ts` 断言两者的工作集和排序完全一致，所以 `core-engine.spec.ts` 可以放心地在内存里测试领域规则，不用担心与生产实现有偏差。

---

# 第三部分　流程精讲

第二部分是“每个零件长什么样”，这一部分是“零件怎么咬合”。每条流程都落到具体的函数和事务。

## 15. 流程一：一条明确请求的一生

![图 5：端到端时序](img/d2-sequence.svg)

**图 5：9 个编号的 SQLite 事务把整个流程切成 9 段。每个事务只写状态和待办；发确认、启动执行器、投递输入、转达结果这些对外动作，都由 Effects 在事务提交后执行，结果再作为“事实”回到 Engine。**

场景：名单内的 alice 在 Issue #42 下评论“@agent 把设置页的 Save changes 按钮改成 Save”。

| # | 事务 | 由谁触发 | 写了什么 | 提交后发生什么 |
|---|---|---|---|---|
| 1 | `Engine.receive` | GitHub Source 的轮询 | Trigger；去向记录；Thread `github:42` 的 pending | Passes 被请求补一轮 |
| 2 | `Evals.start`（在 `evaluateDue` 里） | Passes | Evaluation（explicit，`via: addressed`）；`assessment` effect 行 | 卡片以 steer 方式进入主 Session，然后 `candidateDelivered` 开始计时 |
| 3 | `Engine.decide`（accept） | 主 Session 调 `orchestrator_route` | Decision；Mandate；Ticket；订阅 `github:42`；pending 的派工；确认回执；`launch` 和 `reply` effect | 发出确认；DshExecutor 占一个工位，跑 prepare，建会话，以 queue 方式发派工消息 |
| 4 | `assignmentStarted` | DshExecutor | 派工 pending → running；有待投递输入就排 delivery | — |
| 5 | `report(progress, pr=57)` + `bindThread` | 执行器调 `orchestrator_report` | 报告；工单跟随 `github:57`（own）；订阅 | 不 relay（普通进度不打扰主 Session） |
| 6 | `Engine.receive` | PR #57 上的 CI 失败、review 评论 | 扇出给工单的 Input（**不经 Eval**）；`delivery` effect | 以 steer 方式投给执行器；会话写入历史后 `inputsRead` |
| 7 | `report(ready)` | 执行器 | 报告；`relay` effect | 主 Session 被唤醒，看到 `worker_report` |
| 8 | `Engine.reply(result)` | 主 Session 调 `orchestrator_reply` | 结果回执；`reply` effect | 结果发到 Issue #42（工单的起源对话） |
| 9 | `threadClosed(github:57, merged)` | GitHub Source 观察到 PR 合并 | 工单 `ended: merged`；未读的人类消息回流 Eval | `afterCommit` 停掉执行器会话的 turn |

**值得注意的三处：**

- **第 3 步是全流程最关键的事务**：一次 accept 把 Decision、Mandate、Ticket、订阅、派工、确认回执、两个 effect 一起提交。半截状态在结构上就不可能出现：不会有“工单建了但没派工”，也不会有“派工了但没订阅线程”。
- **第 5 步的 progress 报告不打扰任何人**，包括主 Session。从第 6 步起，PR 上的所有反馈都直达执行器，主 Session 不参与，所以它的上下文只承载分诊。
- **第 9 步之后工单并没有“消失”**：订阅保留、Mandate 有效。alice 一周后在 PR #57 下说“标题也改一下”，第 1 步会直接把这条消息扇出给这个已结束的工单，`Tickets.wake` 重验证 Mandate 后复活它，不需要新的 Eval。

## 16. 流程二：主动发现，从影子模式到放权

场景：一个设为“全收”的飞书群里，bob（名单内）发截图说“这个按钮文案有点怪”，carol 回复“就叫 Save 吧”，没人 @ 机器人。

1. **Gate**（§10.2）：bob 在名单上、没 @、作用域全收，准入表“名单内、未 @”这一格默认是 inferred，结果是 **inferred**，`basis = {via: 'admission', cell: 'listedUnaddressed', observation: {mode: 'observe'}}`。两条消息同在一个话题线程，合成一批。
2. **卡片**：`next_action` 告诉主 Session 这是主动候选。卡片能确定与任何规则无关时（比如闲聊），可以直接 ignore，Decision 会带上 `ignoredFromCard`，和“逐条研判后忽略”区分开。这里可能命中“文案修正”规则，所以要研判。
3. **研判**（§11.3）：`orchestrator_eval` 拿到线程窗口和截图 id，`orchestrator_eval_read` 读入截图，`orchestrator_eval_decide` 填写：
   - `copy` 命中，`visual` 未命中；
   - 方向：已请求实现，目标是明确的“Save”，不需要设计决策；
   - `intent_evidence` 引用两人原话。按提示词，这正是“简短同意也能确认一个具体提案”的例子。
4. **代码出结论**（§10.4）：形式检查全部通过，但模式是 observe，结果是 `ignore + would_accept + proposal`。**群里一个字都没有。**
5. **运维审阅**：面板 Eval 页列出了一周的 would_accept 记录，看起来靠谱，于是把模式切到 auto。策略版本随之变化，进行中的评估全部作废、重新排队（`Evals.policyChanged`）。
6. **放权之后**：下一次类似讨论直接 accept。Mandate 的强度是 inferred，`basis.rules = ['copy']`。执行器多收到两段约束：
   - 派工文本里写明“Proactively accepted under copy … Matched rules … take precedence”；
   - system prompt 末尾的 `mandateInstructions`（原文见 §12）：只允许单模块内的小改动，涉及公共接口、行为变化或产品决策就报告 dropped。
7. **再收权**：运维关掉 `copy` 规则。`policyEffects` 算出这个工单受影响，`Tickets.policyChanged` 投一条 `policy-changed` 输入，执行器读到的是：

   > “Rule Copy fixes (copy), which this ticket was accepted under, no longer applies: finish only a change within it, or report dropped.”

   工单不会被强行取消。如果它此时已经结束，下次被人的消息唤醒时会在 `revalidate` 失效，拒收消息，消息回流 Eval。

**这条流程体现的取舍：**主动行为默认“只看不做”，放权要人显式操作；收权不打断进行中的工作，只把“规则变了”作为一条输入交给执行器自己收尾。代价是收权有延迟：执行器可能在本轮里继续做完。

## 17. 流程三：后续消息归谁？扇出、交回与回流

![图 6：三层状态机](img/d3-state-machines.svg)

**图 6：工单只有 open 和 ended 两态，阶段由规则派生；一个工单一生可以有多次派工；输入分“未投递、已投递未读、已读”三态。重试或重启会把已投递但未读的输入放回队首；工单结束时，未读、也没被处理的人类消息还给 Eval。**

这是代码里规则最密集的地方，用一个场景串起来。工单 T 正在处理 PR #57，dave（名单内）在 PR 下评论“顺便把 header 颜色改成蓝色”。

1. **扇出**（`Intake.receive`）：`github:57` 被 T 订阅，消息扇出给 T（Input id = `key('input', T, 'trigger', triggerId)`）。`followupOwners` = `[T]`，所以**不排进 Eval**。
2. **通知主 Session**：Runtime 在同一个事务里把这件事转成一条 `follow_up` relay，主 Session 知道了，但对这条消息只能发起 stop（`leftToWorkers`）。
3. **执行器裁决**：执行器被唤醒，这条消息走的是“给人预留的那个额外工位”，所以即使工位满了也不会排在新工单后面。它判断改颜色超出“改按钮文案”的范围，调 `orchestrator_trigger_handback(reason)`：
   - 去向记录里加一条 `handbacks`；
   - T 不再拥有这条消息，也没有别的工单拥有它，于是排进 Eval；
   - 主 Session 看到的卡片上带着 `handed_back: [{ticket_id: T, reason}]`；
   - 主 Session 可以开新工单、挂到别的工单或忽略，但**不能再挂回 T**（`handedBackHere`）。
4. **另一种结局**：执行器在写 ready 报告时，dave 的消息恰好到了。工单因为 ready 结束，执行器没机会读这条消息，也没机会交回。`Tickets.end` 计算出这条消息“归 T 裁决、在 `readUntil` 之前没被读、也没被确认或交回”，`letGo` 把它从 T 的扇出列表拿掉并排进 Eval。**这条消息没有丢。**
5. **叫停**：dave 说“这个工单先停下”。消息同样先扇出给 T；主 Session 收到 follow_up 后用 `orchestrator_route` 发起 stop。stop 要过 Gate，强度必须是 explicit（dave 在名单内，他的后续消息强度就是 explicit）。工单以 `stopped` 结束，Mandate 撤回，T 从此收不到任何消息；这条“停下”的消息本身不再回流 Eval（`letGo` 里对 stop 的特判）。

**为什么不让主 Session 统一裁决所有后续消息？**执行器手里有这个工单的完整上下文，判断“这条消息是不是我的活”最准；主 Session 只需知道、只保留叫停权，它的上下文就不会被每个工单的细节撑大。代价是执行器要承担范围判断；交回要绕一圈；一条消息被多个工单拥有时，要等每个工单都交回或结束。

## 18. 流程四：PR 的生命周期与工单何时结束

工单与 PR 的关系由 `ticket.follows[]` 记录，每项是 `{线程, own（是否自己开的）, closed?}`：

- **建立：**执行器开 PR 后 `orchestrator_report(pr_number)` → `claimPull` → `bindThread(own = true)`；也可能由 `observePull` 按分支名自动绑定（§9.4）。起源就是 PR 的工单，跟随的是那个 PR（`own = false`）。
- **ready 不等于结束：**有 follows 时，ready 报告只把阶段变成“等待评审”（`person / review`）；`Assignments.report` 只有在 `followsDone` 成立时才结束工单。
- **PR 关闭时：**`threadClosed` 标记 follow 已关闭。`followsDone` 的规则是：如果工单自己开过 PR，就看“自己开的是否都关了”；没开过，就看“跟随的是否都关了”。结局取最后一次关闭的结果。
- **关闭了但工作没完：**`unfinished` 检查两种情况，满足任一就暂不结束，等下一份 ready 或 dropped 报告：执行器在最近一次非 progress 报告之后接手过人的后续消息；或者 PR 关闭之后复活的派工还没报告 ready。
- **重开：**PR 被重新打开，`claimPull` 允许重新绑定（之前的 follow 已关闭）。
- **结束通知：**来自飞书的工单收到一条结束通知，id 由 PR 编号、渠道和结果决定，工单复活后再结束也不会重复发；GitHub 上 PR 本身就显示了结果，不再重复。
- **合并之后：**订阅保留，PR 下人的新评论可以复活工单（重验证 Mandate）。bot 和 CI 的消息不会唤醒已结束的工单。

## 19. 流程五：在任何一点崩溃会怎样

这是全系统设计的“验收题”。下表按时间顺序列出崩溃点，以及系统靠什么恢复：

| 崩溃发生在 | 结果 | 靠什么恢复 |
|---|---|---|
| Engine 命令事务提交前 | 全部回滚。Source 下一轮会重读（游标还没动）；工具调用失败，模型可以重试 | SAVEPOINT；确定性 id 保证重试幂等 |
| 提交后、effect 执行前 | effect 行在表里，重启后 `effects.start()` 照常分发 | 事务性的 effect 队列（§8.2） |
| assessment：卡片投递后、`candidateDelivered` 前 | 重启后 effect 重跑：先确认 Eval 仍是当前那个；投递按 `candidate:<evalId>` 去重；再补上 `candidateDelivered` | AgentDelivery 的“只投一次”（§8.6） |
| launch 进行中 | effect 未删除，重启后重新 launch；已有的会话和目录会被复用 | launch 的每一步都先检查“这次派工是否仍是当前派工” |
| 执行器正在工作 | `resume` 对每个运行中的派工调 `assignmentInterrupted`：停在稳定报告上就保留会话；否则结束旧派工，开新派工并附恢复说明，投给旧派工但没读的输入放回队首 | `Assignments.interrupted`、`requeueUntaken` |
| 输入投递后、会话读入历史前 | 视为未读：重启或重试时重新投递，并先清掉会话里残留的副本，保证只有一份 | `readAt === null` 与 inbox 清理（§12） |
| reply：已经发出、还没记成已发送 | 重发，由渠道去重：GitHub 扫标记，飞书用幂等键 | 至少一次 + 渠道幂等（§9.5） |
| reply：已记成已发送、effect 行还没删 | 重跑时发现回执不是 pending，直接返回 | 回执状态机（§8.3） |
| 进程在迁移中 | 事务回滚，数据库保持 schema 10 | 单事务迁移（§8.8） |
| 进程挂了没人管 | 另一个 profile 里的定时 agent 每小时跑 `status`，退出码 1 就 `--respect-stop restart` | 外部看门狗（§10.7） |

**一句话总结：**Engine 的决定与它要引发的副作用同事务落盘；所有副作用至少执行一次；每一层用确定性 id 把“至少一次”收敛成“效果上恰好一次”。唯一依赖外部配合的环节是渠道发送的幂等，作者为 GitHub 和飞书分别实现了，新增 Source 时必须遵守。

## 20. 流程六：注意力预算是怎么执行的

从执行器写下一份报告，到人看到一句话，中间有 5 道闸：

1. **报告分级**（`reportRelay`，`notices.ts:121-129`）：普通的 progress 报告只写进工单记录，**连主 Session 都不唤醒**；带问题的报告变成 `worker_attention`；ready、blocked、failed、dropped，以及带回答的 progress，变成 `worker_report`。
2. **提问必须结构化**：执行器只能通过 `orchestrator_report` 的 `human_action` 字段提问，一次一个问题；不能把问题写在报告摘要里等人发现。“是否需要人”从不从自由文本里推断（`docs/001-domain.md` 的 Rejected concepts 表）。
3. **发言有用途、有前置条件**（§7.7）：主 Session 发 result 前工单必须 ready；发 attention 时文本必须与当前问题逐字一致；发 answer 时执行器必须报告过答案，且这个权限只能用一次。
4. **长度硬上限**：IM 消息在传输层按词数检查（中文先分词），超过 100 词直接拒绝（`checkedImText`）；路由备注 ≤30 词；给执行器的便条 ≤100 词。
5. **默认沉默**：团队沟通段的默认文本规定“没有人能据此行动的内容就不发”，并给出好坏对照示例；只有 ready 的结果、需要人做的决定、确定无法完成这三种情况才主动发 IM。

**为什么要做到这么细？**`docs/000-philosophy.md` 的判断是：一个能代表整个团队行动的 agent，也能浪费整个团队的注意力。前两道闸由代码强制，后三道由代码和提示词共同约束。这里的经验是：凡是能由代码强制的，都没有只留给提示词。

---

# 第四部分　亮点、取舍与风险

## 21. 设计决策矩阵

| # | 问题 | 直觉做法 | 它的选择 | 理由 | 代价 |
|---|---|---|---|---|---|
| 1 | 协调状态放哪 | orchestrator LLM 记在上下文里 | 同步、单写者的领域内核 + SQLite（§7） | 常驻系统一定会经历上下文压缩、重启、人中途插手 | 概念多，代码量大，规则之间耦合紧 |
| 2 | 副作用何时执行 | 在业务代码里直接调 API | 与状态同事务入队，提交后由分发器执行（§8.2） | 崩溃时要么都没发生，要么决定和待办都在 | 事务里不能有 await；分发靠 2 秒一轮的 kick |
| 3 | 如何不重复 | 先查后写、分布式锁 | 确定性 id + 回执永久保留 + 渠道幂等（§7.10、§8.3） | 把“至少一次”收敛成“效果上恰好一次” | 每个对象的 id 规则都要精心设计；回执表只增不减 |
| 4 | 谁能指挥 agent | system prompt 写“只听谁的” | 代码 Gate + 不可变 Mandate + 重验证（§10.2–10.3） | 评论和文档都是不可信输入，权限不能存在于文本通道 | 配置面大：名单、准入表三格 × 作用域 × 模式 |
| 5 | 主动行为如何上线 | 直接开 | observe 影子模式，只记 would_accept（§16） | 先看它会接什么，再放权 | 多一步运维操作 |
| 6 | 配置变了怎么办 | 撤销受影响的工单 | 投 `policy-changed` 输入，复活时再重验证（§10.5） | 中途撤销会留下半成品分支和 PR | 收权有延迟 |
| 7 | 后续消息谁裁决 | 全部回到 orchestrator | 订阅扇出直达执行器，越界的交回（§17） | 上下文局部性，路由器保持精简 | 执行器要判断范围；交回要绕一圈 |
| 8 | 路由器能写需求吗 | 改写成计划再派发 | 禁止：scope 取 title，备注 ≤30 词，执行器读原文（§11.2） | 避免传话失真和凭空加需求 | 执行器每次都要从原文重建上下文 |
| 9 | 研判谁来做 | 另起一个评估 LLM | 主 Session 自己做，有界窗口，代码出结论（§11.3） | 复用常驻上下文和缓存，规则只写一处 | 主 Session 上下文压力大；ignore 也要填完整的 schema |
| 10 | 工单何时结束 | 模型说 ready 就结束 | 有 PR 时看 PR 关闭这一外部事实，并检查 `unfinished`（§18） | “模型认为做完了”不等于“团队收到了结果” | 规则复杂（followsDone、unfinished、复活） |
| 11 | 怎么接外部事件 | webhook | 轮询（GitHub）+ CLI 长连接（飞书），游标回退 5 秒（§9.2） | 不需要公网入口；身份就是 agent 账号；重启安全 | 分钟级延迟；消耗 rate limit；一整套游标逻辑 |
| 12 | 模型能看到多少 | 把整个线程塞进去 | 卡片、窗口、存根、读取指针，每种都有字符上限（§11.3） | 控制上下文体积，强迫模型按需读取 | 模型可能漏看关键信息，要靠 `eval_read` 补 |
| 13 | 提示词怎么防腐 | 靠人 review | `model-surface` 与 `cards` 测试守护模型看到的每一个字（§11.6） | 提示词就是接口，要像接口一样测试 | 改一个词可能要改测试 |
| 14 | 长命令怎么办 | 等它跑完 | 前台超过 5 秒转后台 job，完成后以消息唤醒（§12） | 人的新消息能在下一个 step 进来 | 执行器要会处理异步 job 的结果 |

## 22. 风险与可疑点汇总

下面合并了逐模块精读中发现的问题。每条都标了性质：“确认”是对照代码直接成立的行为；“推断”是没有运行验证的判断。按影响从大到小排列。

**安全与授权：**

1. **（确认，设计如此）执行器不在安全边界内。**两个 preset 都以运维者的 OS 用户、`danger-full-access` 运行，执行器跑命令不需要审批；命令 shim 只决定默认凭据，可以用绝对路径绕过（`docs/006-deployment.md` 原文承认）。要隔离，只能给 agent 单独开 OS 账户。
2. **（确认）授权配置本身可以被主 Session 修改。**`policy` 和 `eval` 块没有来源限制，主 Session 可以扩大名单、改准入表、切 auto 模式；只有执行环境和账号字段是“仅面板”。主 Session 一旦被注入，影响面就是授权面（§10.1）。
3. **（确认）explicit follow-up 的 Mandate 不随名单收回。**名单内的人发起的后续工单，在这个人被移出名单后仍然有效，也不会收到通知（§10.8）。
4. **（确认行为，风险是推断）GitHub 评论被编辑后成为新的 Trigger，actor 仍记为原作者。**有写权限的人编辑别人的评论加上 @，会被记在原作者名下（§9.8）。
5. **（确认）重试会绕过工作目录的 git 配置检查。**目录检查失败时目录已经建好，重试时 launch 看到目录存在就跳过检查，仓库本地配置里的 `credential.*` 和 `http.*extraheader` 会在第二次启动时被放行（§12.7）。这是本次精读中**唯一一个与安全直接相关的确认 bug**，修复也简单：检查失败时删掉目录，或者每次启动都检查。
6. **（确认）subagent 的工具限制比执行器宽：**它能用 `web_fetch`；找不到执行器祖先时，完全不受白名单约束（§12.7）。

**正确性：**

7. **（确认）放弃重试的 GitHub 通知不会被补读**，与注释和文档不一致（§9.8）。
8. **（确认）内容相同的配置通知会被永久去重**：同一主 Session 生命周期内，第二次“已暂停”通知不会送达（§8.9）。
9. **（推断，较高置信）写进历史记录的策略版本可能与运行时不一致**，原因是一个没有经过 schema 规范化，另一个经过了（§10.8）。
10. **（确认）飞书的三个循环出错后静默停止**，健康状态不更新（§9.6）。
11. **（推断）锁失效后后台循环仍在运行**（§8.9）。

**运维与规模：**

12. **（确认）数据只增不减**：Trigger、Decision、回执、设置与提示词历史、飞书“已见消息”集合都没有清理逻辑（`docs/007-architecture.md` 原文承认）。
13. **（确认）有序 effect 没有死信队列**：一条永久失败的 relay 会一直挡住后续所有 relay（§8.9）。
14. **（推断）研判窗口的存根数量没有上界**：大线程可能产生上千条存根（§11.3）。
15. **（确认）主 Session 是串行单点**：所有候选都要经过它；默认 3 个执行器工位；GitHub 轮询至少 60 秒。这是一个面向“一个仓库、一个团队”的设计，不是平台。

## 23. 可以直接借鉴的做法

如果你也在做“常驻、多会话、对外有副作用”的 agent 系统，下面这些可以几乎原样搬走：

1. **事务性副作用队列。**每张 `effect` 表每个 `(kind, key)` 一行，与状态同事务 upsert，提交后用 microtask 分发；用 `seq + generation` 精确删除自己那一行；`Later` 推迟但不计失败。约 160 行（`src/app/effects.ts`）。
2. **按角色裁剪的领域接口。**`Pick<Engine, …>` 给每类调用方一个子集，让权限边界由编译器检查；请求用祈使句，事实用过去时。
3. **确定性 id 表。**为每个对象写下 id 公式（§7.10），幂等就有了，而不是靠先查后写。
4. **读时计算的状态。**不存 `state` 字段，用一个按优先级排列的纯函数从事实推出阶段（`ticketStage` 只有 10 行），面板、卡片、按钮共用它。
5. **“投递”与“读到”分开记录**，再加上“结束时把未读的人类消息还回去”（`letGo`），人的消息就不会丢。
6. **错误码即纠偏提示。**给模型的拒绝信息直接写清“下一步该做什么”。
7. **模型可见面的测试。**把模型能看到的每一段文本收集起来，检查体积预算、引用的工具和资源是否存在、术语是否一致（`tests/model-surface.spec.ts`）。
8. **影子模式。**主动行为先只记录“会做什么”，人审阅之后再放权。
9. **防阻塞守卫。**在工具层拒绝 `sleep` 和长等待，长命令自动转后台，让 agent 始终可以被新消息打断。

## 24. 方法说明与证据

- **怎么读的：**`src/core` 的 2,088 行由我逐行精读（§7）。其余 6 组模块（持久化与副作用、Source、策略与配置、主 Session 与提示词、执行器与会话、面板与管理 API）由 6 个子 agent 并行逐文件精读，各自返回带行号和原样代码摘录的报告。我对每份报告抽查了 1–6 条关键结论，共约 20 条，包括本文列出的每一个“确认”类问题，都与代码一致。有两处我修正了定性：研判窗口的 near 判定与 `docs/007` 一致，不算 bug；Effects 没有自己的定时器，第一版正文写错了，已改正。
- **没有做的：**没有运行测试（依赖 DSH 的 21 个 peer 包，未安装），所有行为描述都来自读代码，不是运行观察；“推断”类问题都没有用实验验证。
- **行号：**基于 commit `6337fc8`，文件名后的数字是行号。
