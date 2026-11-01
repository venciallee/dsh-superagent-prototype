# dsh-superagent 分析

两篇文档，建议先读概念篇，再读源码篇。HTML 版是单文件，图已内嵌，直接用浏览器打开即可。

| 文档 | 内容 | 文件 |
|---|---|---|
| 概念篇：《dsh-superagent 深度拆解》 | 它是什么、解决什么问题、核心设计与亮点；与 DeepSeek Harness、ReAct、multi-agent、Agent Team 的对比 | `dsh-superagent-explainer.md` / `.html` |
| 源码篇：《dsh-superagent 源码精读》 | 高层架构；逐模块精读（核心、持久化、Source、策略、主 Session、执行器、面板、测试）；六条端到端流程；设计决策矩阵与风险汇总 | `dsh-superagent-source-deep-dive.md` / `.html` |

图（`img/`，明暗两色自适应）：

| 文件 | 回答的问题 |
|---|---|
| `l0-system.svg` | 系统长什么样，LLM 在哪两处出现 |
| `d1-hexagon.svg` | 领域核心如何通过 6 个端口与外界隔离 |
| `l1-durable-kernel.svg` | 为什么崩溃不丢、不重：决定与副作用同事务落盘 |
| `l1-routing.svg` | 一条消息的两条路：执行器裁决，或经 Gate 给路由 LLM |
| `d2-sequence.svg` | 一条请求从 @ 到合并，经过哪 9 个事务 |
| `d3-state-machines.svg` | 工单、派工、输入三层状态如何迁移 |
| `l1-spectrum.svg` | ReAct、Subagent、Agent Team、superagent 在“触发、授权、状态、执行”四个轴上的差别 |
