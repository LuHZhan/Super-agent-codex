# Codex Super-agent

面向个人的主动式工作助手。这里记录讨论中已经明确的领域用语，不规定技术实现。

## Language

**Codex Super-agent**：
本项目参考 dsh-superagent-prototype 的实现构建的 Codex 版本，简称 SAC。目标是在预设授权范围内处理工作，逐步具备主动发现、启动和持续推进任务的能力。
_Avoid_：将其仅理解为一次请求中的多 Agent 并行执行。

**SAC**：
Super-agent 的 Codex 版本，本项目 Codex Super-agent 的统一简称。
_Avoid_：将 SAC 与参考项目 SAD 混称；本项目后续不使用 SA 作为简称。

**Router**：
路由与协调者，负责理解事件、判断是否需要行动以及协调任务的 Agent 角色。
_Avoid_：将 Router 与保存状态、执行调度的全部基础设施混为一谈。

**Worker**：
执行者，承接具体任务并汇报结果或阻塞情况的 Agent 角色。
_Avoid_：将 Worker 的一次执行等同于任务的完整生命周期。

**SAD**：
Super-agent 的 DeepSeek 版本，特指参考项目 dsh-superagent-prototype。
_Avoid_：将 SAD 与计划构建的 Codex Super-agent 混称。

**Source（事件来源）**：
SAC 接收外部变化的来源，例如 CODING 中 BUG 单及评论的变化；其适配器把平台数据转换为系统可理解的事件。
_Avoid_：将 Source 当作代码仓库，或把收到事件等同于获得执行授权。

**Thread（事项会话）**：
围绕同一个外部事项形成的持续会话；当前场景是一张 CODING BUG 单及其相关评论。
_Avoid_：与操作系统线程、Codex 执行会话混淆，或每收到一条评论就当作新事项。

**Outbox（待发箱）**：
SAC 对外消息的发送记录集合，记录哪些结果待发送、已发送或仍需核对。
_Avoid_：将进入 Outbox 当作已经送达平台。

**Channel（回传通道）**：
向外部事项会话写入消息的通道；当前场景用于把处理结果回复到原 CODING BUG 单。
_Avoid_：与覆盖接收和回传两端的广义“信道闭环”混淆。

**开发主分支（Development base branch）**：
用户持续开发新需求并整合修改的个人开发分支；SAC 修复工作的分支基线及 MR 目标应指向它。
_Avoid_：默认等同于仓库默认分支、main、master 或发布分支；具体分支名称与基线版本另行配置。
