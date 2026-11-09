# SAC 接入腾讯 CODING BUG 单

状态：Open API 轮询方向已确认，配套细节继续澄清，未批准实施。CODING 调查日期为 2026-11-02，Codex 接入与条款核验更新于 2026-11-05。人类视图见 `overview.md`，范围见 `proposal.md`，领域名称定义见根目录 `GLOSSARY.md`。

已确认 CHANNEL-01：腾讯 CODING 是首个接入平台；CONN-01：Open API 轮询起步，Webhook 后置；REPO-01：代码仓库也在 CODING；CONN-03：指定账号的 @SAC 评论口令触发受理。其余适配、可靠性、权限与认证细节继续澄清；当前未授权实际运行或远端操作，没有调用用户团队 API。

## 1. 官方能力与证据

### Open API

官方提供 HTTPS POST、Action 和 JSON 请求体形式的 Open API。概要列出 `https://e.coding.net/open-api`，认证示例也使用团队域名。具体操作页的 URL 展示不完全一致，实际联调需选择并验证同一调用形式，不能直接机械拼接所有示例。

| 能力 | 官方 Action | 已核对内容与使用建议 |
| --- | --- | --- |
| 缺陷列表 | `DescribeIssueListWithPage` | `ProjectName`、`IssueType=DEFECT`、页码和每页数量；支持 `UPDATED_AT`、标签、处理人、状态等筛选，支持更新时间排序。页面示例 `Conditions` 为数组，字段展示为 object，需要联调核对 |
| 单张详情 | `DescribeIssue` | 以 `ProjectName + IssueCode` 定位；返回描述、创建人、处理人、状态、标签、自定义字段和附件；可请求描述图片下载地址 |
| 评论列表 | `DescribeIssueCommentList` | 以 `ProjectName + IssueCode` 定位；示例包含 CommentId、ParentId、RawContent、CreatorId、CreatedAt、UpdatedAt。所读请求参数未列分页，不虚构分页参数，完整性需验证 |
| 创建评论 | `CreateIssueComment` | Content、IssueCode、ProjectName、ParentId；文档将 ParentId 标为必填，顶层评论取值需验证，不直接假设可省略；未列原生幂等键 |
| 修改事项 | `ModifyIssue` | 可改 StatusId、AssigneeId、标签等；能力存在不表示第一版应开放给 Worker |
| 读取项目配置 | `DescribeProjectIssueTypeList`、`DescribeProjectIssueStatusList`、`DescribeProjectIssueFieldList` | 官方目录列出，可用于核对自定义事项类型、工作流及属性；不硬编码每个项目的状态 ID |
| 附件 | `DescribeIssueFileUrl`、`DescribeIssueAttachmentPreSignedUrl` | 官方目录列出下载地址和预签名信息能力；具体字段、有效期及权限待联调 |
| 活动记录 | `DescribeIssueLogList`、`DescribeIssueStatusChangeLogList` | 官方目录列出，可用于核对操作人与状态变化；不把“仅按状态更新时间查询”误作所有变化的增量接口 |

来源：[列表](https://coding.net/help/openapi#/operations/DescribeIssueListWithPage)、[详情](https://coding.net/help/openapi#/operations/DescribeIssue)、[评论读取](https://coding.net/help/openapi#/operations/DescribeIssueCommentList)、[评论创建](https://coding.net/help/openapi#/operations/CreateIssueComment)、[修改事项](https://coding.net/help/openapi#/operations/ModifyIssue)。操作页经浏览器实际展开核对，未发送其 API 测试请求。

### 认证与配额

官方列出 OAuth 2.0、个人访问令牌、项目令牌。个人令牌使用 `Authorization: token ...`，项目令牌以用户名/密码组成 Basic 凭据；项目协同对应项目令牌 scope 为 `project_issue_rw`，个人/OAuth 对应 `collaboration:issue:ro/rw`。文档标明单团队单接口每秒最多 30 次请求，设计应远低于该上限并处理限流。

来源：[Open API 认证说明](https://coding.net/help/openapi)。项目令牌的实际接口覆盖及回评身份仍需在目标团队验证。

### Webhook / Service Hook

采用当前 Service Hook 文档，不混用旧 WebHook 的事件名及投递约定。

- 事项事件列出 ISSUE_CREATED、ISSUE_DELETED、ISSUE_STATUS_UPDATED、ISSUE_ASSIGNEE_CHANGED、ISSUE_ITERATION_CHANGED、ISSUE_RELATIONSHIP_CHANGED、ISSUE_UPDATED。
- 请求头包括 `X-Coding-Service-Hook-Event`、`X-Coding-Service-Hook-Id`、`X-Coding-Service-Hook-Action`、`X-Coding-Delivery`；消息体包含团队、项目、事项和发送者信息。
- Webhook 服务支持向指定地址发送 HTTP POST，配置签名令牌、HTTP 认证、自定义头和模板。签名说明为 HMAC-SHA1 十六进制值加 `sha1=` 前缀。
- 所读事件表未单列事项评论创建/修改事件；不能假设评论变化一定发出 ISSUE_UPDATED，也不能假设评论一定改变事项 UpdatedAt。
- 所读 Service Hook 页面没有明确保证重试次数、顺序、离线保留时长及签名所在请求头；这些需要实际投递核验。不要把旧 WebHook 页的超时信息直接当成 Service Hook 保证。

来源：[事件](https://coding.net/help/docs/project-settings/service-hook/event.html)、[Webhook 服务与签名配置](https://coding.net/help/docs/project-settings/service-hook/service.html)、[Service Hook 简介](https://coding.net/help/docs/project-settings/service-hook/intro.html)、[升级说明](https://coding.net/help/docs/project-settings/service-hook/upgrade.html)。

### SDK

未在本次查阅的官方资料中找到可确认覆盖当前事项接口的官方 Node.js/TypeScript 或 Python SDK。这是本次调查范围内的结论，不是断言所有语言均无官方 SDK。

找到社区 Go 项目 `juanjiTech/coding-open-api-sdk`，其作者 README 明确标为实验性且多数内容未测试，因此不作为 SAC 首版的核心依赖。官方文档的多语言 HTTP 示例也不等于提供受维护 SDK。

来源：[社区 SDK 原仓库](https://github.com/juanjiTech/coding-open-api-sdk)。建议用薄 HTTP Client 封装少量必需 Action，后续可在相同接口下替换 SDK。

## 2. CONN-01：连接方案比较与已选方向

用户已选择 Open API 轮询 + API 回评，Webhook 后置。具体轮询参数尚未定稿。

| 方案 | 收益 | 代价 | 建议 |
| --- | --- | --- | --- |
| Open API 轮询 + API 回评 | 本地只需出站 HTTPS；容易补读和验证 | 有轮询延迟，需分页和限流 | 首版推荐 |
| Service Hook + API + 补偿轮询 | 事项变化通知更及时 | 要有平台可达接收端、验签及重复投递处理；评论通知覆盖未确认 | 后续增强 |
| 单独依赖 Webhook | 接收路径看似简单 | 缺详情补读与回写、无法假定无遗漏 | 不采用 |

D2 已确认初始轮询间隔约 60 秒。实现建议加入抖动，按项目规模和调用量校准，展示最近成功扫描时间。实际发现延迟还取决于扫描耗时和失败退避。

若以后引入 Webhook：平台 → 可达接收端 → 验证来源 → 持久化接收 → 快速响应 → 后台补读 API。不要在回调请求内等待 Codex 完成任务。采用中继会增加部署组件和数据流转范围，届时另行决定。

## 3. CONN-02：参考 SAD 的职责分配

```text
CODING Open API
  → CodingClient（认证、HTTP、限流、响应解析）
  → CodingSource（轮询、快照差异、Trigger、Thread 映射）
  → 持久化接收记录 + 确定性受理规则
  → Codex Router → Codex Worker → 结果及验证证据
  → Outbox → CodingChannel → CreateIssueComment → 原 BUG 单
```

| SAD 实现 | SAC 对应建议 |
| --- | --- |
| `src/sources/github.ts` 的轮询和游标 | CodingSource 的事项增量读取与评论轮询 |
| `src/infra/commands.ts` 的 gh 封装 | CodingClient 直接调用 CODING API，无需模拟 gh |
| `src/sources/kind.ts` 的 scope、threadContext、channel | 项目/事项过滤、详情与评论读取、结果回写能力 |
| Trigger 与 Thread | 平台事件规范化、同一 BUG 的持续上下文 |
| Policy 与 Ticket | 可执行动作的规则约束及一张 BUG 的处理记录 |
| `src/app/outbox.ts` | 发送记录持久化、重试、回执核对 |
| `src/sources/github-channel.ts` 的发送去重 | 评论中稳定的 SAC 发送标识 + 本地发送记录 + 远端核对 |

Codex 负责理解和执行；游标、鉴权、去重、权限拦截及发送队列由普通程序负责。Codex 的具体进程/协议仍是独立待决策项。

## 4. CONN-03：身份、会话与受理

- Thread 建议以 `coding:<teamId>:<projectId>:<issueCode>` 标识，避免不同团队/项目同号 BUG 冲突。API 调用另外保存 ProjectName；项目改名时更新路由，不改变业务身份。
- Channel 地址保存团队、项目、IssueCode，以及需要楼中楼回复时的 ParentId；默认建议使用 BUG 顶层回评，顶层 ParentId 值需验证。
- 同一 BUG 的补充评论进入原处理记录，不因每次消息新建任务；是否允许结束后重新受理另行明确。
- 接收范围与执行授权分开。已确认由 指定的 CODING 用户 在 CODING BUG 下评论 `@SAC xxxx` 作为受理信号；用户纠正了此前的口令拼写，并接受评论对同事可见。
- 身份匹配建议使用目标团队内经验证的用户 ID，与评论 `CreatorId` 对比，不依赖可变显示名或正文中的自称。授权账号的实际身份须在本地联调时核验。
- **POLL-02（已确认）**　新任务发现范围为指定项目中 `AssigneeId` 命中两名指定处理人之一的 BUG。有效 `@SAC` 评论的 `CreatorId` 必须匹配用户本人经验证的 ID。联调时分别核对两名处理人的数字 ID。使用本人凭据时可通过 `DescribeCodingCurrentUser` 核对自己的 `Id`、`Name` 和 `GlobalKey`。已受理任务转出处理人范围时按已确认的 D1 暂停、保留并继续跟踪。[CODING 官方响应示例](https://coding.net/help/docs/member/tokens.html)
- 口令建议匹配去掉首尾空白后正文起始位置的独立 `@SAC`，其后须有空白和非空指令。不因普通文字中包含该词、引用或代码块出现该词就触发。优先检查 API 的 RawContent；富文本 mention 的表示需用真实样例验证，不从 HTML 不加区分地提取整段文字。
- `@SAC` 可以作为纯文本口令，解析规则不依赖存在同名机器人账号。它不重命名 SAC，也不代表自动建立了 CODING 原生 mention 关系；是否选择原生 mention 另行核对。
- 受理条件为配置范围内的 DEFECT + 指定 CreatorId + 合法口令。评论 ID 作为请求来源，同一评论重复读取时按 INTAKE-02 去重。除 MR-02 外，其他已结束任务的重开规则待定。
- **INTAKE-02（已确认）**　受理时保存触发评论 ID、作者和当时的指令正文。Router 分派、Worker 执行和重启恢复都以保存的正文为授权依据。轮询再次读到同一评论 ID 时，即使正文或 `UpdatedAt` 已变化，也不覆盖已受理指令、不重新分派。通过 CODING 调整范围时，须以原 BUG 下新发的 `@SAC` 评论为来源，重新校验作者、口令及动作权限。
- **INTAKE-03 指令语义（已确认）**　Router 读取受理时保存的指令正文及当前任务，识别目标、明确限制和有歧义的动作。受控执行组件按当前授权逐项核验。Worker 先做已明确允许的部分，暂缓有歧义的动作。仅当下一步取决于澄清时，按 ATT-01 选择一个渠道提问。推送受限时按 PUSH-01 执行。
- **PUSH-01 推送限制的生效时机（已确认）**　按同一任务串行处理新指令受理和推送发起。受理「先别推送」时，已入队但尚未发出的推送失去执行授权，保留本地改动。若推送已发出，根据推送回执和 CODING 远端修复分支的提交核对结果。证据不足时面板显示「推送结果待核对」，保持推送暂停，核实后更新面板。在这条限制生效期间阻止创建 MR，包括已排队的创建重试。
- **PUSH-02 恢复推送（已确认）**　将推送限制保存在任务状态中。受理“继续修”时只恢复本地修复和验证。Router 识别到明确允许推送的意思后，受控执行组件还须核对指令来源和现有交付条件，才能推送。
- **UI-02 日常沟通（已确认）**　localhost 对应 BUG 的 Codex 任务对话可独立承载 PUSH-02 的授权。受控执行组件核对用户身份及对话绑定的 BUG 任务，记录本次明确许可，仅解除该任务的推送限制，确认待推送提交已通过要求的验证后执行。无需等待 CODING 评论。CODING 的任务触发、回评和 ATT-01/02 阻塞问答仍关联同一任务。本地身份核验方式待设计。
- **INTAKE-01（已确认）** 在线发现或离线补读到修复指令后，在创建任务、发送受理确认之前核对 CODING 中的 BUG 当前状态。确认已关闭时，按触发评论 ID 保存跳过结果，在本地面板显示「受理前 BUG 已关闭」。以后重读该评论时沿用此结果。BUG 重新打开后，仅对新发的有效 `@SAC` 指令重新检查。状态读取失败时按读取失败处理。
- 受理前记录团队、项目、IssueCode、CommentId、评论版本及授权依据，关联原 BUG 会话。评论正文只能描述任务，不能自行修改权限规则。修复后的推送、MR 和正常结果回评权限遵循 AUTH-04；其他动作另行明确。
- 用户自己发表评论后，SAC 通过 API 读取即可受理，不需要再改标签或处理人。用户接受评论共享可见，不以完全私密或无通知为约束；明确修复请求通过检查并保存后，按 ACK-01 回评一次。
- 列表快照的 CreatorId 不代表最近修改者。若规则依赖“谁发起了操作”，从已核对的活动记录或 Webhook sender 获取；无法确认时不伪造 actor。
- REPO-01 已确认代码托管在 CODING。显式配置 CODING 项目/BUG 到仓库及本地工作目录的映射，不默认一个项目只有一个仓库。Git 克隆、拉取与推送所用凭据，以及 CODING MR 接口与权限后续核对；本轮不将其视为已打通。

## 5. CONN-04：读取、去重和发送恢复

### 读取

1. **POLL-01（已确认）** 首版只对用户指定的一个 CODING 项目轮询 BUG 和评论。首次开始监听时，先保存该项目的监听起点。按评论创建时间区分历史指令和新指令，默认跳过起点之前的指令。初次扫描过程中发出的新指令也要进入受理检查。恢复时沿用已保存的起点和读取进度，在 POLL-02 范围内补读离线期间仍可读取的新指令，包括旧 BUG 单上的新评论。重叠扫描按评论 ID 去重。
2. 按配置项目查询 DEFECT，使用更新时间区间、升序和分页；固定本轮上界，保留重叠窗口。
3. 记录事件后才推进游标；部分页失败保持可重读，不跳过失败页。重叠窗口、稳定事件键和周期性核对共同降低分页期间数据变化带来的遗漏，不能声称轮询还原了每一次瞬时修改。
4. 对 POLL-02 范围内的候选 BUG（包括尚未受理者）轮询评论，活跃任务可提高频率。不能只读取已受理 BUG，也不依赖评论必然推动事项 UpdatedAt；扫描范围、分页完整性与发现延迟需按实际规模评估。离线补读同样受处理人筛选限制，离线期间已转出范围且尚未受理的 BUG 上的指令可能无法发现，用户接受此边界。已受理任务按 D1 继续跟踪。
5. 事项事件键建议包含业务身份、更新时间和关键字段摘要；评论事件键包含 CommentId、UpdatedAt 和内容摘要。SDK/平台若提供可靠版本号，则优先用版本号。
6. 评论已有状态和附件元数据分开保存。删除或撤权导致的不可读不能一概当作 BUG 已关闭。

### 首轮调度与运行决策

- **D1（已确认）** 已受理 BUG 转给两名指定处理人之外的人时，暂停执行、保留任务和已有状态，在 localhost 提醒用户决定是否继续。用已保存的 BUG 身份持续读取状态，候选列表筛选不能删除已有任务的跟踪记录。具体停止和恢复协议仍需验证。
- **D4（已确认）** 首版手动启动 SAC 后台，网页关闭后后台继续运行。主动退出后台后，下次启动核对持久化状态与实际执行状态再恢复。
- **D5（已确认）** 故障集中显示在 localhost，短暂网络故障自动退避重试，权限失效或需要人工处理时明确显示阻塞。执行恢复须核对任务和已发生的动作，发送结果不明时遵循 CONN-04。
- **D3（已确认）** 默认 3 个 BUG Worker 执行名额、上限可配置、超额排队，另为人工消息唤醒已有任务预留 1 个额外名额，人工续办优先于新任务。等待答复且无后台命令或子 Agent 运行时才释放名额，启动和唤醒前预留名额以避免超限。Router 和子 Agent 的资源另行计量。

SAD 本地源码依据为 `src/policy.ts` 的 `maxWorkers` 默认 3，`src/limits.ts` 允许 1–16，`src/executor/dsh.ts` 的 `WorkerSlots` 在启动和唤醒前预留名额，允许人工唤醒多占 1 个名额。活跃 turn、后台 shell 或子 Agent 运行期间持有工单名额。SAC 已确认采用默认 3 个普通名额和 1 个人工续办额外名额，默认最多同时执行 4 张 BUG。额外名额用于人工消息唤醒已有任务。Codex 上的实际后台状态和资源计量需验证。

### 发送

- 先原子保存 Outbox 记录和稳定 messageId，同一 BUG 的发送串行执行。
- 回评加入可被回读识别的 `SAC messageId` 标识，成功后保存远端 CommentId（若响应未返回，则读取核对）。隐藏标记是否保留需要验证，不能照抄 GitHub HTML 注释就假定生效。
- 超时不等于发送失败。先查询原 BUG 评论核对 messageId；找到则确认成功，无法确定时保持“发送待核对”，避免盲目重复 POST。
- 该方案不保证远端 exactly-once。没有服务端幂等支持时，应暴露不确定状态，不能把它当作已发送或丢弃。
- 自己产生的评论按已记录的远端 ID/标识过滤，避免再次创建任务。如果使用个人令牌，不应把该账号所有人工评论一并过滤。
- 网络错误和限流退避；认证/权限错误停止对应连接并报告。错误记录保留 Action、RequestId 和脱敏错误，不输出令牌。

## 6. CONN-05 与 CONN-06：回写权限和凭据

AUTH-01 已确认：明确要求修复的指令可由 SAC 自动排查、修复、验证；分析指令仅分析。AUTH-04 已确认验证通过后的自动推送、MR 和正常结果回评。分支隔离方式见已确认的 BRANCH-04；自动修改事项状态未获默认授权，关闭 BUG、合并与发布由人工处理。当前不编写产品实现代码。

### BRANCH-01：开发主分支与 MR 方向

- 已确认：用户开发主分支持续承接新需求，修复工作的基线和 MR 目标必须指向该分支，不能回退到仓库默认分支。
- 每个仓库需明确配置目标开发分支；缺少配置时不猜测为 main/master。名称尚未提供。
- BRANCH-04 已确认使用用户另行准备的 BUG 专用 clone，每个 BUG 取开发主分支的明确提交为基线，在该 clone 下创建独立修复分支与 worktree；MR 指向同一开发主分支。已确认设计，不表示本轮创建工作目录或实施。
- BRANCH-02 已确认以 CODING 远端已推送版本为基线，记录取得时的 commit；不自动吸收用户工作目录内未提交或未推送的新需求修改。
- 开发主分支在修复期间会推进，按 BRANCH-03 区分首次交付前同步与 MR 后冻结；冲突通过追加 merge commit 解决，不强推用户开发主分支。
- MR 的 base 必须显式设置为已配置的开发主分支，不能依赖托管平台默认值。BUG 项目到仓库/分支的映射继续待定。

### BRANCH-03：交付前同步与 MR 后冻结（已确认）

用户明确：任务修复分支仅由该 Worker 及其子 Agent 开发；推送并进入 MR 状态后，除非出现代码冲突，否则不再改动，冲突通过补充提交处理。此前持续跟随目标分支 rebase 以及 force-with-lease 的建议撤回，不作为默认权限或流程。

首次交付前流程：在未发布的任务分支完成修改，停止并行写入、保存原 head 后 fetch，将本任务修复提交 rebase 到明确的远端开发主分支提交；比较补丁和差异，重跑 BUG 回归与项目检查，保存 target SHA 与 fix SHA，首次推送前按 PAUSE-01 核对 BUG 当前状态，通过后普通 push。创建或重试创建 MR 前按 PAUSE-02 核对 BUG 和 MR 当前状态。首次推送后若 MR 创建失败，重试 MR 创建时沿用已推送 head，不因服务中断改写已发布历史；期间如需同步，按已发布分支规则处理。

**PAUSE-01（已确认）**　从受理到首次推送前，一旦确认 CODING BUG 已关闭，暂停该任务的 Worker 和子 Agent，保留原分支、worktree、未提交改动及已有提交。面板显示「修复中 BUG 已关闭，改动已保留」。重新打开后收到用户新发的有效 `@SAC` 指令时，关联原暂停任务，核对 BUG 当前要求、工作区差异及已有提交，再继续修复。首次交付仍按 BRANCH-03 同步和验证。

**PAUSE-02（已确认）**　已推送分支后，在首次创建或重试创建 MR 前核对 BUG 状态。若上次请求结果不明，先按源分支和目标分支核对远端是否已有对应 MR。确认 BUG 已关闭且 MR 尚未创建时，暂停创建，保留远端修复分支和本地 worktree，面板显示「BUG 已关闭，修复分支已推送，MR 未创建」。BUG 重新打开且收到新的有效 `@SAC` 指令后，恢复原任务，核对分支差异和验证结果是否仍适用，必要时重新验证，再创建 MR。

**MR-01 评审中关闭 MR（已确认）**　受控执行组件根据任务保存的仓库和 MR 标识，核对 BUG 已关闭、对应 MR 仍开放且未合并后，执行关闭。若 MR 已关闭或已合并，记录远端实际状态。关闭请求结果不明时先重读 MR 状态，再决定是否重试。接入验证需覆盖关闭接口及令牌权限。

**MR-02 重开 BUG 后新建 MR（已确认）**　前一轮 MR 因 BUG 关闭而关闭且未合并时，以重新打开 BUG 后收到的新有效 `@SAC` 修复评论为触发。核对原修复分支相对当前远端开发主分支的差异，按本轮指令修复并重新执行验证。验证通过后新建指向该开发主分支的 MR，保存本轮指令与新 MR 的关联，保留旧 MR 的关闭记录。创建请求结果不明时核对本轮是否已有新 MR，按评论 ID 去重。

进入 MR 后已确认：

- 无代码冲突时冻结源分支；目标分支仅有新提交不是重新 rebase、补提交或强推的触发条件。普通上下文评论也不自动解除冻结，新的明确人工修改指令另行受理。
- 发现代码冲突时进入冲突处理，保留已有提交，追加解决结果后更新同一 MR。采用 merge 最新目标到修复分支、处理冲突、重新验证并普通 push；用户已接受生成 merge commit，保留原修复提交历史。
- 不 rebase 或强推开发主分支，不自动 rebase 已发布修复分支；普通推送被拒绝时先核对，不升级成强推。Worker 与子 Agent 对同一 worktree 的 Git 操作仍需串行协调。
- 机械冲突可在既定 BUG 范围内处理；新需求改变预期行为时提出具体问题。目标删除或历史发生非快进变化时暂停自动整合并核对。
- 验证证据继续绑定明确的目标与修复提交。目标推进不抹除历史验证，也不自动代表最新组合验证通过；只读组合验证及其失败后的处理另行设计，不擅自改写冻结分支。

取舍：MR 后追加 merge commit 可以保留已评审的提交历史并使用普通推送，但不保证开发主分支最终只有线性的修复提交。最终人工合并可按项目支持选择 merge 或 squash 等方式；目标在 MR 后推进时，fast-forward 条件可能不再成立，不能为满足线性历史而自动解除冻结。CODING 的实际合并方式、保护规则、冲突检测及评审失效规则待核验。

依据：[Git rebase](https://git-scm.com/docs/git-rebase)、[Git merge](https://git-scm.com/docs/git-merge)。以上为设计讨论，不执行 Git 操作。

### CONN-05：对外评论与工程证据

用户已确认：正常结果回评采用 SAD 风格的一句结果、链接与必要下一步，实质限制才补第二句；不描述技术内部。正常交付顺序为修复与验证 → MR → BUG 评论，MR 目标遵循 BRANCH-01。自动交付权限遵循已确认的 AUTH-04。

核对的 SAD 依据：

- `dsh-superagent-prototype/prompts/team/communication.md`：结果应回到原会话；默认只写简短用户可见结果、链接和必要下一步；技术实现、文件路径、commit、测试命令和工作过程留在 PR/内部报告。
- 同文件强调：PR 创建不意味着已准备好、已修复或已合并；只有自审和验证后可供人评审/使用，才发送正常结果通知。进行中的常规工作不反复对外播报。
- `dsh-superagent-prototype/prompts/work.md`：区分用户可见结果和技术证据；达到可评审状态后提供真实 PR URL 和准确状态，不必等待合并。

上述短通知规则在 SAD 中主要用于 IM。SAC 拟将其表达原则用于 CODING BUG 评论，这是适配建议，不宣称 SAD 已有 CODING 评论模板，也不照搬其字数限制。

建议结果通知保持状态准确，例如区分“修复已验证，MR 待合并”和“已上线”；不暴露内部 API、实现细节、命令或执行流水。若验证缺口影响使用或评审，应以普通语言说明实际限制。需要用户决定或补充材料时，按 ATT-01 单独提问。

CONN-05 正常结果回评（已确认）：参考 SAD，用一句话表达用户可见结果、实际 MR 链接和必要下一步，实质限制才补第二句，与 overview 示例一致。此前“处理结果 / 当前状态 / MR”三项格式是 SAC 自拟候选，不是 SAD 的固定模板。文字从已核对的验证与 MR 状态生成，不填造链接或把创建 MR 当作修复上线，不机械输出“验证通过”或“尚未上线”。技术证据留在 MR/报告；受理确认遵循 ACK-01。除 ATT-01 外，异常及冲突处理通知规则待定。

**MSG-01 评论署名（已确认）**　CODING Channel 将 `SAC：`（全角冒号）作为每条发往 BUG 单的评论正文开头，重试沿用同一正文。前缀用于读者识别，受理指令仍校验 `CreatorId` 和 `@SAC`，直接答复按 ATT-02 关联，自回评按已发送评论 ID 识别。

本轮补充核对 SAD 的评论设计（以下路径均位于参考仓库）：

- `prompts/replies.md` 和 `src/core/replies.ts`：acknowledgement、answer、result、attention 四类用途；受理确认关联人类请求，答问关联原问题且受已报告答案约束，attention 匹配当前结构化问题。result 的代码准入覆盖 ready、completed、merged、dropped 等状态；准入不代表每个状态都必须发送，仍受沟通策略约束。
- `prompts/team/communication.md`：普通进度不主动通知；完成自审、验证并可供评审后才通知结果；IM 默认一句，实质阻塞或限制才加第二句；常规测试无需提及。GitHub 评论、review 回复及 PR 描述在提示词中要求自动 Agent 身份声明，IM 不加该声明；这不是由 github-channel 自动拼接的字段。
- `src/app/outbox.ts` 与 `src/sources/github-channel.ts`：保存发送回执和重试，GitHub 发送前按账号与消息隐藏标识查重，发送到 Issue/PR 或相应 review 讨论。发送器使用提供的 text，并未强制渲染固定三行正文；该去重机制也不能直接等同于 CODING exactly-once 保证。
- `prompts/replies.md` 的 100 words 上限是 IM 规则，`src/app/outbox.ts` 对 feishu reply 调用长度校验；不将其表述成所有 GitHub/CODING 评论通用限制。

以上区分提示词表达规则、程序发送门槛和平台投递机制；SAC 采纳范围继续按已确认规则推进，不自动继承 SAD 的通知权限或 review 自动处理行为。

优先验证项目令牌的 `project_issue_rw`：范围便于限制在项目，但它是读写范围，并非仅评论权限。因此 CodingClient 还需要 Action 白名单。若项目令牌不覆盖所需接口或身份不符合预期，再评估受限账号个人令牌；多用户产品化时再考虑 OAuth。

平台凭据只由本地 Client 使用，不传给 Router/Worker 的提示词。实际 BOT/个人身份、撤权和轮换需要联调验证。附件获取只使用已核对的 API 返回地址，不对 BUG 文本中的任意 URL 附带凭据。

## 7. 文档中已发现的歧义及联调要求

**ACK-01 受理确认（已确认）**　对指定账号发出的明确修复请求，在确定性受理和动作授权通过、任务持久化后，将具体受理确认送入 Outbox，并回复原 BUG。建议消息身份绑定平台、项目、BUG 和触发评论 ID，避免重启、轮询与传输重试重复入队；发送不明时按 CONN-04 核对，不承诺 exactly-once。文字只表明已接下具体请求，不根据任务已创建就声称 Worker 已开始运行。此规则不授权为无效口令、其他作者、普通讨论或每次内部重试发评论；其他请求类型另行明确。示例见 overview。当前阶段仅维护设计文档。

**ATT-01 人工提问（已确认）**　先核对 BUG 描述、评论和已有材料。确需用户决定或补充材料时，按 BUG 保存用户指定的提问渠道，按任务和阻塞标识保存唯一待答问题及实际投递渠道。默认在 localhost 对应任务对话写入一条提问，仅在用户明确为该 BUG 指定 CODING 时选择原 BUG 单并经 Outbox 发送。关闭页面、轮询、重启或迟迟未答时复用已有记录，避免在同一渠道或另一渠道重复发问。CODING 发送结果不明时按 CONN-04 核对。有效答复关联同一问题并恢复原任务。

**ATT-03 切换提问渠道（已确认）**　任务页点选和经核验的用户指令更新同一张 BUG 的提问设置。创建待答问题时保存当时选定的渠道。切换后，已有问题的发送、发送结果核对及答复关联仍使用保存的渠道。验收时在本地提问后切到 CODING，原问题只在本地出现且仍可答复，下一条新问题只发到原 CODING 单。

**ATT-02 直接回复（已确认）**　在 CODING 提问时，用已绑定的 `CreatorId` 核对答复者，用团队、项目、BUG 和待答问题的评论 ID 核对回复对象。回复属于该问题且足以继续时，将答复归入原任务，按回复评论 ID 去重，在已有授权范围内恢复一次。关联不明时保持等待。接入验证需用一条直接回复和一条同单普通评论对照，确认 `DescribeIssueCommentList` 返回的 `ParentId` 能否准确指向提问的 `CommentId`。

| 精确证据 | 原因/未知点 | 可能影响 | 验证与预防 |
| --- | --- | --- | --- |
| 列表页字段显示 `Conditions object`，请求示例为 `Conditions: []` | 文档 schema 与示例不一致，真实接受类型未实测 | 请求失败或过滤不符合预期 | 在测试项目验证，再固定请求契约并保留响应样例 |
| CreateIssueComment 的 ParentId 标为 required，样例为 1 | 页面未说明顶层评论约定 | 回评失败或挂错父评论 | 单独验证顶层与子评论，不使用示例的 1 作为默认 |
| 评论创建响应示例为 `{}`，字段仅见 RequestId | 未确认是否返回新 CommentId | 无法直接建立发送回执 | 发送后回读识别标记，验证读写一致性 |
| Service Hook 事项事件表没有单列评论事件 | 公开契约未确定评论通知覆盖 | 只依赖 Webhook 可能漏追问 | 独立轮询评论；测试创建/编辑评论的通知行为 |
| 未列评论幂等键 | 无法从文档确认服务端幂等保证 | 超时重发可能重复评论 | Outbox、稳定标识、串行发送及远端核对，不承诺 exactly-once |

还需核对：目标团队版本、项目类型、自定义工作流、令牌权限、时区和时间字段精度、评论完整性、附件下载权限。没有据此认定 CODING 存在产品故障，只记录接口文档的验证缺口。

## 8. 后续验证顺序（非本轮执行）

**ACCEPT-01 第一轮通过条件（已确认，尚未实测）**　用户在联调时提供验收单链接。监听开始后，指定账号在一张已有、尚未关闭的真实 CODING BUG 下发出 `@SAC <排查要求>`。以 BUG 编号和触发评论 ID 核对 localhost 上唯一对应的任务，记下任务 ID。遇到影响下一步排查且现有材料无法回答的问题时，核对它只在默认 localhost 任务对话里出现一次。用户在本地回答后，以同一任务 ID 核对后续排查记录确实采用了答复。若未发生真实提问，将问答续办记为待验证。代码修改、推送及 MR 交付在第二轮验收，CODING 评论与直接回复的关联另行验证。

1. 只读契约验证：认证、指定项目 DEFECT 列表、详情、评论及附件元数据。
2. 经授权在测试 BUG 上验证顶层评论、子回复、中文/Markdown、发送后回读与身份。
3. 用重复事件、重启、评论追问、超时及权限失效验证接收与发送闭环。
4. 接入 Codex Router/Worker，执行一张目标明确且已授权的 BUG 单，回传结果与证据。
5. 再评估是否需要 Service Hook；若采用，验证签名头、投递 ID、重试、重复及乱序。

## 9. 首轮信道工程方案与 Codex 核验

核验日期为 2026-11-05。本节给出工程建议和核验结果，产品规则沿用已确认条目。证据分为官方文档、本机 CLI/schema 检查和端到端实测。当前前两类已完成，真实推理与进程重启后的续办待联调。

### 9.1 ENG-01 接入路径与官方适用范围

建议 SAC 以本地 App Server 的 stdio JSON-RPC 接入 Codex，通过独立持久化线程承担 Router 与 Worker。官方平台文章将持久对话、事件和审批列为 App Server 的用途，SDK 更适合程序化任务封装。该划分与本项目的 localhost 任务对话相符。[官方平台架构说明](https://developers.openai.com/blog/codex-as-a-platform)

官方仍将 App Server 标为实验性，明确提示生产负载支持限制。首轮按个人本地原型验证，版本与协议一起锁定。旧 codex mcp-server 集成已被官方移除，模型调用入口采用 App Server，业务工具可使用 Codex 支持的外部 MCP。[官方迁移说明](https://learn.chatgpt.com/docs/mcp-server)

**AUTH-05 工程建议**　新建 SAC 优先验证 Sign in with ChatGPT 的 ChatGPT plan usage 流程。官方专题覆盖开源及本地托管应用，提供本人授权后使用订阅额度的路径。SAC 的个人、本地、本人发起编码任务场景与该文档描述相符，这是基于公开资料的适用性判断。具体账号和地区准入、授权 scope、模型访问需要实际验证。[订阅接入适用范围](https://developers.openai.com/siwc/token-sharing-open-source)

旧的 App Server 托管 ChatGPT 登录可继续用于已有本地或开源应用，官方同时建议迁移到 Sign in with ChatGPT，并限制其用于商业或托管服务。新 SAC 的认证设计优先采用当前推荐路径；现有 CLI 登录已用于本机恢复实测，但不替代 SAC 独立授权的核验。[App Server 认证](https://learn.chatgpt.com/docs/app-server#auth-endpoints)

公开资料支持官方客户端集成和订阅授权路径，但没有为 SAC 或任何指定并发数提供账号免封保证。条款要求遵守使用政策，禁止共享账号凭据、绕过限制等行为，官方也会因滥用或异常使用采取措施。SAC 绑定本人身份与授权，只供本人本地使用，额度不足时保留任务并等待恢复，权限或政策拒绝时显示原始原因与请求标识并停止受影响的执行。具体限流错误与恢复时点以服务返回为准。[使用条款](https://openai.com/policies/terms-of-use/) · [账号警告说明](https://help.openai.com/en/articles/10562178-why-did-i-receive-a-warning-about-my-account) · [订阅错误与恢复](https://developers.openai.com/siwc/token-sharing-open-source/errors-and-recovery)

### 9.2 AUTH-05 认证与额度处理

**官方同类实现核对（2026-11-05）**　检索官方插件目录及 OpenAI 维护的公开项目，未安装插件或改变认证选型。

| 参照 | 实际证据 | 对 SAC 的含义与限制 |
| --- | --- | --- |
| [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) | README 的 Requirements 明确允许 ChatGPT subscription 或 API key，并计入 Codex usage limits；Codex Integration/FAQ 明确包装本机 App Server、复用 CLI 登录，提供后台任务、状态、取消和 resume | 这是安装于 Claude Code 的官方插件，证明外部 Agent 委派本地 Codex 并使用订阅有官方先例；不证明任意部署或并发数获得豁免 |
| [openai/symphony](https://github.com/openai/symphony/blob/main/elixir/README.md) | 官方原型轮询 tracker，为工单创建 workspace、启动 Codex App Server，并持续推进；当前适配器包括 Linear、GitHub Issues、Jira Cloud、Asana、GitLab | 是独立编排器，工单执行结构接近 SAC；README 标明仅供评估，未列 CODING，不能直接当作完整 SAC 或订阅政策许可 |
| [Sign in with ChatGPT 官方示例](https://developers.openai.com/cookbook/articles/sign-in-with-chatgpt) | Usage policy and terms 明确覆盖本地运行的开源工具和个人项目；收费或远程托管应用需先申请接入 | 直接支持 SAC 本地个人新应用的候选授权方向；仍须满足具体账号、scope、模型准入和服务条款 |

**2026-11-06 源码对照与选型建议**

codex-plugin-cc 的 getCodexAuthStatusFromClient 调用 account/read（refreshToken=false）及 config/read 来识别登录和 provider。插件通过 thread/start 或 thread/resume 分派任务。refreshToken=false 仅用于状态检查，CLI 托管登录的正常续期由 Codex 执行。Symphony 的本地 start_port 在工单工作目录启动配置的 Codex 命令，并从子进程环境清除 tracker 声明的敏感变量。SAC 可分别借鉴前者的登录检查和后者对工单凭据的处理。[插件源码](https://github.com/openai/codex-plugin-cc/blob/db52e28f4d9ded852ab3942cea316258ae4ef346/plugins/codex/scripts/lib/codex.mjs#L868-L877) · [Symphony 源码](https://github.com/openai/symphony/blob/be10a1b79df723d6d7612b5651c8522704dafb2e/elixir/lib/symphony_elixir/codex/app_server.ex#L192-L244) · [CLI 认证与续期](https://learn.chatgpt.com/docs/auth)

| 候选路径 | 谁保存和刷新凭据 | 对桌面及采用条件的影响 |
| --- | --- | --- |
| 共用桌面默认 CLI 登录 | Codex | 设置最少，但 Windows 原生 CLI 默认与桌面共用状态目录，测试会话、配置和登录操作都可能相互影响。与本项目隔离偏好相冲突 |
| 专用状态目录，使用官方 CLI 登录同一账号 | Codex 管理这次登录，Router 和 Worker 共用 SAC 的授权 | 技术可行，隔离性优于共用桌面目录。已有应用继续使用的官方说明对新 SAC 的适用性仍待确认。作为技术候选保留 |
| SAC 使用 Sign in with ChatGPT 请求 plan usage | SAC 管理本应用的授权和续期，App Server 执行任务 | 官方材料直接覆盖个人本地新项目。凭据与 App Server 使用专用目录，须验证 scope、模型准入和续期。建议作为新应用首选 |

三种路径下，同一个人账号的用量仍计入对应订阅。独立目录只分开本机状态。账号风险判断与部署范围见第 9.1 节。[Windows 状态目录](https://learn.chatgpt.com/docs/windows/windows-app) · [本地新应用订阅授权示例](https://developers.openai.com/cookbook/articles/sign-in-with-chatgpt#usage-policy-and-terms)

**专用 CLI 登录候选的隔离与验收**（设计，待实测）

2026-11-08 的案例补充。OpenAI 将 T3 列入开源集成，并把账户身份与 ChatGPT plan usage 定义为独立权限。T3 的应用内订阅连接与手动 CLI 实例可分别配置。对 CLI 候选，T3 建议用完全独立的 CODEX_HOME 分开配置和会话，shadow home 则用于需要共享会话的场景。Zed 分别提供原生 ChatGPT Subscription provider、受支持环境中的 ACP 外部 Codex 和终端 CLI。两者为设计参照，SAC 的付费或远程托管部署需按开发者入口另行申请。下列验收只针对专用 CLI 候选。[OpenAI 用户说明](https://learn.chatgpt.com/docs/sign-in-with-chatgpt) · [T3 Code](https://github.com/pingdotgg/t3code/blob/main/docs/user/providers-codex.md) · [Zed](https://github.com/zed-industries/zed/blob/main/docs/src/ai/use-an-existing-subscription.md) · [应用接入范围](https://developers.openai.com/siwc/token-sharing-open-source)

1. 登录命令与后续 App Server 都使用锁定版本的 npm CLI。只给这些子进程配置同一 SAC 专用 CODEX_HOME / CODEX_SQLITE_HOME，例如用户本机仓库外的 %LOCALAPPDATA%\SAC\codex。确认有效 sqlite_home 指向专用目录。使用最小配置、本机 stdio 和独立子进程，保留桌面进程自己的环境与 daemon。
2. 在专用目录中由本人完成一次官方 codex login 浏览器授权。若策略允许，显式选择 file 存储并限制目录 ACL，明确 auth.json 的落点。使用 keyring 时另验凭据条目的隔离。由 Codex 自行写入和续期，SAC 只保存业务映射。排除 API key 覆盖、桌面凭据复制和向 Worker 暴露认证文件，文件读取边界另按 SEC-01 验证。[官方凭据存储说明](https://learn.chatgpt.com/docs/auth) · [状态目录与覆盖优先级](https://learn.chatgpt.com/docs/config-file/environment-variables)
3. 先确认 provider 和 account/read 返回的 chatgpt 登录，再做短请求、关闭 SAC 自己的进程、按原 threadId 续办。另在正常续期后验证后续请求，观察桌面仍可继续原对话。比对两端状态位置和配置，区分桌面自身的正常写入与 SAC 的写入。失去授权时在 localhost 提示本人重新授权并保留任务。该候选的自然续期和双端并行尚无本机实测结论。

**账号限制边界**　[使用条款](https://openai.com/policies/terms-of-use/) 限制账号共享、程序化抽取及绕过保护/用量限制；[Pro 说明](https://help.openai.com/en/articles/9793128-about-chatgpt-pro-tiers) 列明禁止共享凭据、转售访问和自动抽取；具体本地集成应遵循上列官方授权文档，不能从一般条款推出所有自动化都被禁止，也不能从技术接口可调用推出所有自动化都被允许。[账号警告说明](https://help.openai.com/en/articles/10562178-why-did-i-receive-a-warning-about-my-account) 说明异常/过度使用等可能引起合规关注，警告不等于立即封禁，持续违规可能进一步限制。没有公开资料保证 SAC 或 D3 的 3+1 永不触发账号审查，也没有依据量化封号概率。当前仅本人发起和授权、他人不能通过 SAC 使用本人账户的边界应持续保持；达到额度时等待，禁止通过更换身份或规避限制继续调用。

建议将认证作为 SAC 后台内的独立模块，维护本应用的授权与凭据引用。初次注册使用真实 SAC 应用名和持久 host ID，按官方 OAuth/OIDC + PKCE 流程验证返回身份、state、nonce 及 scope。后续登录沿用注册的 client ID 与 host ID。只有身份登录成功而缺少 chatgpt.tokens.use.direct 时，将订阅推理标记为不可用。[注册与登录](https://developers.openai.com/siwc/token-sharing-open-source/sign-in) · [授权缺失处理](https://developers.openai.com/siwc/token-sharing-open-source/errors-and-recovery)

按官方文档，为 App Server 子进程配置 ChatGPT plan Responses provider，将 SAC 自己授权取得的短期 access token 传给该子进程。该路径向 api.openai.com/v1 发请求并使用订阅授权，实际计费依据是凭据与授权路径。应用标识使用稳定的 SAC 名称和版本。model/list 可供显示候选模型，模型授权须由一次真实推理确认。[App Server 订阅接入](https://developers.openai.com/siwc/token-sharing-open-source/codex-app-server)

SAC 负责新流程的令牌续期。采用官方 env_key 配置时，续期后需用新 token 重启 App Server，再恢复持久化 thread。应协调所有受影响会话，在可恢复边界轮换并保留发送中的输入状态。官方说明 store=false 下本地历史和 thread/resume 仍可使用；该流程支持本地 shell、MCP 与子 Agent，但托管工具存在限制。[预览限制与续期行为](https://developers.openai.com/siwc/token-sharing-open-source/preview-limitations)

额度按同一账号共享，D3 的 3+1 是 SAC 调度参数。普通网络故障与额度不足、认证过期、政策拒绝分开处理。确认额度不足后保留队列并显示可用的恢复时间；用户新增消费或更换计费路径需明确选择。托管 ChatGPT 认证的 account/rateLimits/read 在本机 schema 存在，新订阅 provider 的限额观测方式需联调确认。[Codex 订阅说明](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan) · [额度说明](https://help.openai.com/en/articles/9793128-about-chatgpt-pro-tiers)

### 9.3 ENG-02 本机核验与兼容问题

| 项目 | 本轮证据 | 结论 |
| --- | --- | --- |
| CLI | 2026-11-05 按用户授权通过官方 npm registry 将 @openai/codex 从 0.130.0 更新到 0.166.0；入口返回 codex-cli 0.166.0 | 本轮协议与恢复验证基线为 0.166.0；后续升级重新核验 |
| 配置兼容 | 将原 config.toml 完整复制到隔离状态目录；features list、app-server --print-startup-config 及加 --strict-config 的检查均 exit 0，无 stderr | default 值无需改为 fast；严格解析通过不代表每个插件或集成均已实测 |
| 传输 | app-server --help 列出默认 stdio:// | 可按标准输入输出设计本地协议适配 |
| 核心方法 | 默认导出的 ClientRequest schema 含 thread/start、thread/resume、thread/read、turn/start、turn/steer、turn/interrupt | 方法存在，运行行为仍需实测 |
| 持久线程 | ThreadStartParams 含 ephemeral，ThreadResumeParams 必需 threadId | SAC 保存准确 threadId，并使用持久模式 |
| 结构化输出 | TurnStartParams 含 outputSchema | 可用于 Router 决定和 Worker 阶段结果，解析及校验由 SAC 承担 |
| 追加输入 | TurnSteerParams 必需 expectedTurnId、input、threadId | 适配器必须处理 turn 已结束的竞态 |
| 历史读取 | 0.166.0 默认 schema 已含 thread/turns/list、thread/items/list、resume.excludeTurns；thread/read 的 includeTurns 在本轮实际使用 | 旧版 0.130.0 的字段缺失已不适用于当前版本，分页方法运行行为仍待测 |
| 输入标识 | TurnStartParams 新增 clientUserMessageId，本轮 turn/start 携带该字段成功 | 字段可接受，不等于已证明重复提交幂等；steer 与投递去重仍须单独验证 |
| 登录与真实请求 | 隔离的现有登录缓存经 account/read 返回 chatgpt；gpt-6-astra 两轮均 turn.status = completed | 现有登录路径与该模型本轮可用；未测试剩余额度或独立 OAuth |
| 同线程恢复 | 第一进程保存校验码并完成提问，退出后第二进程 thread/resume 返回原 threadId 及 1 轮历史；续办后历史共 2 轮，准确返回原码 | REC-01 的 Codex 上下文恢复通过；未模拟运行中崩溃、丢回执、业务问题去重 |
| 新订阅授权 | 本轮仅阅读 Sign in with ChatGPT 官方文档 | SAC 独立注册、scope、令牌续期及真实请求待验证 |

0.166.0 schema 由 CLI 内置 generate-json-schema 在临时目录生成并检查。恢复测试使用 threadId <测试会话 A>，第一轮 <测试会话 B> 完成后退出进程，新进程恢复原线程；第二轮提示没有再次给出校验码，模型仍准确返回。两轮历史仅包含 userMessage/agentMessage，未调用工具。两个测试进程均 exit 0、stderr 为空。未调用 CODING，也未编写 SAC 产品代码。

**安装来源与更新边界**　终端 Get-Command 解析到 %APPDATA%\npm\codex.ps1 / codex.cmd。npm latest 与官方 changelog 当次均指向 0.166.0，实际安装明确指定 @openai/codex@0.166.0、原 npm prefix，并禁用安装脚本。升级前无运行中的 npm Codex；桌面进程位于 AppData\Local\OpenAI\Codex\bin，VS Code 进程位于扩展目录。升级后原 9 个 Codex 进程的 PID/可执行路径仍匹配，未停止或重启它们；未修改 PATH、桌面文件或 VS Code 扩展。config.toml 和 auth.json 的 SHA-256 与升级前一致。桌面内置版本仍不以 npm 版本推断。[官方版本差异说明](https://learn.chatgpt.com/docs/reference/troubleshooting) · [0.166.0 发布及安装命令](https://learn.chatgpt.com/docs/changelog)

**兼容错误记录（已解决）**　旧版 login status 返回 config.toml:3:16: unknown variant `default`, expected `fast` or `flex`，对应 service_tier = "default"。原因是 0.130.0 配置枚举不兼容，阻止配置加载。更新 0.166.0 后对原配置副本的严格启动检查通过，原配置无需修改。防止复发的措施是绑定可执行文件版本、配置模板和该版本生成的协议，升级时先在隔离目录复验。

**测试隔离与认证边界**　配置检查和会话测试分别使用临时 CODEX_HOME / CODEX_SQLITE_HOME，测试 cwd 为空目录。未指向桌面历史、数据库或 daemon；测试配置关闭插件、MCP 应用、shell、子 Agent、浏览器等能力，使用 read-only/never。实际模型沿用用户配置的 gpt-6-astra，采用低推理强度与 default 速度完成短请求。只向隔离文件缓存写入本人的现有 access/id token 和账号关联，refresh_token 留空，未使用 API key、未刷新原令牌、未登出；测试后删除临时认证文件。此方式仅用于这次短时兼容核验，不作为 SAC 的长期登录/续期设计。所有临时诊断脚本、schema 和状态在核验后清理。独立状态目录是降低桌面干扰的隔离措施，不等同于 SEC-01 的完整安全边界。[状态目录](https://learn.chatgpt.com/docs/config-file/environment-variables) · [认证缓存](https://learn.chatgpt.com/docs/auth)

**认证诊断记录（测试方式已修正）**　首次仅向 App Server 子进程设置 CODEX_ACCESS_TOKEN，配合 ephemeral 凭据存储时，account/read 的 account 为 null，随后 turn 失败并返回 `401 Unauthorized: Missing bearer or basic authentication in header`，请求地址为 https://api.openai.com/v1/responses。证据说明当次启动未建立可用认证，不能据此判为账号封禁、额度不足或令牌无效。改用隔离的标准 ChatGPT 缓存后，account/read 为 chatgpt，两轮均成功。预防措施：推理前先验证实际认证类型，认证缺失时停止请求，不依赖单个环境变量存在；不要将这次失败与 Sign in with ChatGPT 新授权方案中自定义 provider 的 ACCESS_TOKEN/env_key 混同。新授权方案尚未实测。

**环境警告记录**　旧版在受限执行环境中报告 failed to clean up stale arg0 temp dirs 和 could not update PATH，伴随 os error 5，路径位于用户 .codex/tmp/arg0；新版在可写隔离状态目录的版本、schema、配置检查及成功会话测试中未再出现。最初在受限环境查询 npm 还报告 `getaddrinfo ENOTFOUND registry.npmjs.org`；通过已授权的受限环境外查询后成功，不修改用户 DNS 或代理设置。此类执行环境问题会阻止下载或辅助目录维护，应区分于认证/账号错误；预防措施是明确可执行路径、核验目标目录权限与网络，再进行升级验证。

### 9.4 ENG-03 首轮组件与 SAD 对照

建议链路为 CODING → CodingSource → SAC 任务存储/调度 → CodexAdapter → 本地 App Server → Router/Worker，localhost 通过 SAC 后台接收任务事件并提交答复。初版建议一个后台、一个受管理的 App Server 进程承载多条线程，便于统一订阅认证与令牌轮换。进程隔离成本与恢复测试结果可作为拆分依据。

| 职责 | SAD 的实现依据 | SAC 工程建议 |
| --- | --- | --- |
| 平台读取与归一化 | sources/github.ts 的轮询、游标和 Trigger | CodingSource 负责 60 秒轮询、指定项目/处理人筛选、作者验证与评论去重 |
| 模型判断 | app/sessions.ts 的 orchestrator 角色 | 项目 Router 使用独立 Codex thread，串行接收事件快照，结构化输出分派或续办意图 |
| 任务执行 | executor/dsh.ts 的持久 Worker 会话 | 每张受理 BUG 的任务绑定独立 Worker thread，人工答复继续同一映射 |
| 调度 | WorkerSlots 和 Effects | SAC 原子预留 3 个普通名额及 1 个人工续办名额，按 D3 排队和释放 |
| 输入投递 | app/delivery.ts 的 requestId、历史和 inbox 核对 | SAC 自建输入投递记录并读取可用历史核对，结果不明时保留待核对状态 |
| 问题与回传 | 持久 Store、Effects、Outbox | localhost 问题独立保存，CODING 回写继续使用 Outbox |
| 运行恢复 | docs/003-execution.md 的重启核对 | SAC 恢复任务记录，Codex 恢复线程历史，适配器核对已发生动作 |

SAD 参考提交为 6337fc823ce20251c8de8085948165c81aaa701e。其 AgentDelivery 依靠 dsh 可指定的消息 ID、inbox 和持久事件做去重；这些内部能力在 Codex 上需要重新建立可验证契约。SAD 的 ensure 会设置 danger-full-access，SAC 权限仍遵循 AUTH-02。

建议使用本地事务数据库保存任务与待发送输入，SQLite 为当前候选。最小记录如下，字段属于实现草案。

| 记录 | 必要内容 |
| --- | --- |
| 项目读取进度 | 项目身份、首次监听起点、逐单扫描进度、处理人 ID |
| 任务 | taskId、BUG 身份、受理原文、当前授权、状态、提问渠道 |
| 执行映射 | taskId/角色、threadId、turnId、运行时版本、provider、模型、cwd |
| 输入投递 | deliveryId、来源评论/本地答复 ID、目标线程、正文摘要、发送与核对状态 |
| 待答问题 | questionId、taskId、生成它的 turn/item、正文、实际渠道、回答与消费状态 |
| 恢复与面板事件 | 执行尝试、名额持有者、最近确认状态、按任务递增的本地事件序号 |

### 9.5 REC-01 会话启动与人工答复

1. SAC 先持久化来源和任务，再让 Router 判断。执行器校验 Router 的结构化输出与已有授权，创建或复用 Worker 映射。
2. 适配器完成 initialize/initialized 后调用 thread/start，并在派发业务输入前保存返回的 threadId。持久模式和历史目录需要显式核验。创建线程回执丢失时先查可核对的线程元数据，避免同时启动重复 Worker。
3. 首轮建议 Worker 用 outputSchema 返回阶段结果，状态可表示需要信息、排查结果或执行阻塞。需要信息时由 SAC 原子保存结果和唯一待答问题，然后在 localhost 呈现。该轮完成且后台工作清空后释放执行名额。
4. 用户答复先保存到原 questionId，再排入同一 Worker 的输入队列。人工续办优先，必要时使用额外名额。空闲线程使用 turn/start；进程重启后先 thread/resume。
5. 用户在 Worker 仍执行时补充信息，可由 turn/steer 携带 expectedTurnId 追加。只有明确返回未接受且核对 turn 已结束后，才转为新 turn；发送结果不明时先恢复核对。
6. 面板读取 SAC 保存的事件和问题。模型请求完成后，再校验阶段结果与业务状态。实时 delta 可展示，恢复依赖持久化历史和业务记录。

以上方法名与参数已由本机 schema 核对。官方文档提供 thread/turn 生命周期、事件及 steer 的约束，版本差异按 ENG-02 处理。[App Server 协议](https://learn.chatgpt.com/docs/app-server)

首轮用可持久化的阶段结果承载业务提问，避免将长时间等待用户绑定到连接上的 RPC 请求 ID。原生工具输入和审批请求由适配器处理其生命周期，重启后旧请求 ID 的可用性需重新核验。授权审批与业务问题在存储中分开，交付权限继续遵循既有规则。

### 9.6 REC-02 重启与投递核对

建议为每个任务设置单一执行所有者，后台使用单实例锁和事务状态，防止两个后台同时启动同一 Worker。重启后的顺序为恢复任务库、确认旧进程状态、核对登录与授权、读取对应 thread 历史、重建待答问题和调度名额，然后恢复允许的工作。

| 中断时点 | 恢复规则 |
| --- | --- |
| 仅网页关闭 | 后台继续执行，网页重开按本地事件序号补读，待答问题沿用原记录 |
| 正在等待用户答复 | 显示同一 questionId；用户已保存的答复继续投递，保持问题关联 |
| 输入尚在本地队列 | 核对当前任务授权后投递一次 |
| turn/start 或 steer 已发出但回执丢失 | 标记投递待核对，查历史和运行状态；只有证据确认未被接收时才重发 |
| 输入已进历史但 turn 中断 | 在原线程创建恢复尝试，结合已有输出继续，核对外部动作 |
| 执行状态未知 | 保守占用对应名额，核实旧执行是否结束，避免恢复后超过 D3 |
| token 续期需更换进程 | 保存输入状态，协调停止与重启，使用同一持久目录和 provider 恢复线程 |
| thread 历史缺失或损坏 | 保留 taskId 并显示恢复阻塞，恢复路径另行核对 |

SAC 自己生成 deliveryId，并保存业务来源及正文摘要。0.166.0 的 TurnStartParams 已有 clientUserMessageId，普通调用已验证可接受，但尚未证明重复提交时去重、重启后的保留方式及与 steer 的对应契约；不能把它直接等同 SAD messageId。JSON-RPC id 仅作为连接内请求关联使用。候选方案是核验 clientUserMessageId，并在必要时加入可回读的输入投递标识，与真实 user input item 核对；其持久化和截断行为必须实测。找不到可靠证据时保留不确定状态，不能把“未查到”直接判作“未执行”。

阶段提问建议以 taskId 与产生它的 turn/item 形成去重约束。答复记录和待投递项同事务写入，避免已答复却因进程退出丢失续办。原始流式事件未证明可全部重放，面板以已持久化事件及历史核对后的状态为准。

### 9.7 SEC-01 首轮权限与本人身份

工程建议为 localhost 仅监听 loopback，使用本地会话凭证及 Origin/CSRF 校验，任务操作关联已验证的本人身份。CODING 作者身份与本地 Web 会话分别校验。

Codex 的模型角色指令承载职责，文件、工具、网络权限由运行时和执行环境约束。Router 只需读取归一化输入并返回结构化决定。首轮 Worker 读取 BUG 与明确提供的资料，文件读取范围及所需命令在联调材料明确后配置。OAuth 与 CODING 凭据由后台持有，传给 App Server 的 token 需从工具子进程环境排除，并验证 Worker 对凭据目录、SAC 状态库及其他进程信息的访问边界。read-only 文件权限本身无法证明这些隔离成立。

受控工具和权限隔离是首轮运行前的验证条件。如果目标 Codex 版本无法满足约束，应缩小工具面或采用独立执行身份后重新核验。此处给出工程约束，具体 Windows 配置尚未验证。

### 9.8 首轮核验安排

| 顺序 | 验证内容 | 通过证据 | 当前状态 |
| --- | --- | --- | --- |
| V1 | 运行时版本、配置与协议 | 启动无配置解析错误，固定版本生成核心 schema | 0.166.0 通过原配置副本严格检查、独立 App Server 启动和新版 schema 检查 |
| V2 | 订阅授权与桌面隔离 | 首选应用授权须验证 scope、真实请求与续期。若评估 CLI 托管候选，按 9.2 验证独立首次登录、同线程续办、自然续期和桌面并行 | 部分通过。已有登录的短期推理与跨进程恢复有实测证据，两条候选的独立授权和续期分别待验 |
| V3 | 同线程续办 | 独立验证线程保存一条信息，关闭进程后 resume，再次询问仍能使用该信息 | 通过：两个独立进程、同 threadId、两轮 completed、随机校验码准确恢复 |
| V4 | 待答与重启 | 问题保存后重启，只显示一次；答复继续原 taskId 和 threadId | 方案已整理，待实现联调 |
| V5 | 投递结果不明 | 模拟发送后回执丢失，核对恢复行为及同一答复的消费记录 | 待实现联调 |
| V6 | 多 Worker 与隔离 | 3 个普通任务加 1 个人工续办，排队、停止、凭据与目录访问符合约束 | D3 已确认，运行未测 |
| V7 | CODING 真实 BUG | 按 ACCEPT-01，从本人新评论到本地问答及原任务继续 | 用户联调时提供单号，待测 |

本轮完成官方资料核对、独立 CLI 升级、配置与协议检查，以及 V3 的真实跨进程会话恢复。V2 尚未完整通过，V4–V6 依赖后续实现与联调；当前结果不覆盖运行中强制崩溃、令牌续期、业务问题去重和投递结果不明。继续以独立测试状态完成这些项目，再将真实 BUG 纳入联调。首轮结果作为代码修复与 MR 阶段的输入。

## 10. HOST-01：架构选择原则与宿主能力候选映射

已确认原则：先定义职责、状态归属、权限及可靠性契约，再评估复用、适配或自建；不以复用率或最少自建代码为目标，不预先禁止自建运行时相关组件。评估语义匹配、可验证性、故障恢复、耦合及维护成本，合理复杂度可接受，极端复杂度或无法实现时调整方案。既定官方接入、个人订阅与 Codex Router/Worker 约束保留。下表仅为候选映射，不是已经批准的完整实现分工。

参考仓库事实：`src/index.ts:8` 注入 agents、sessionController、sessionPersistence、workspaceRegistry、permissionPresets、tools、systemPrompt、skills、webServer、llm、subprocess。`src/app/sessions.ts` 创建/恢复会话，注册工具和提示词，并使用工具限制与 guard；`src/executor/dsh.ts` 管理每张 Ticket 的持久化 Worker 会话及工作目录。`docs/007-architecture.md` 则表明 Engine、Store、Effects、Outbox 和任务恢复调度属于 SAD 自身。

| dsh/SAD 能力 | SAC 的复用或适配建议 | 必须保留的边界 |
| --- | --- | --- |
| agents / sessionController | Codex App Server 的 thread/start、thread/resume、turn/start | SAC 保存 BUG、Worker、threadId 映射；会话不是业务任务 |
| sessionPersistence / 上下文管理 | Codex 持久化会话历史及 thread/compact/start | SAC 另存业务状态、投递记录和验证证据，重启后核对实际副作用 |
| 编码工具 / skills | Codex 工具、Skills/MCP 为候选，可按架构适配或补充自建工具 | 工具和执行循环的归属需按契约评估；实际可用性及订阅接入约束需验证 |
| SAD 专用工具 / systemPrompt | 用角色指令与受控 MCP 工具适配；dynamicTools 可作实验性候选 | 不假设 dsh 的 section 注入、工具注册和限制接口等价 |
| permissionPresets / tools.restrict / guard | Codex 支持的 sandbox、审批协议，加 SAC 业务动作校验 | 执行权限与 BUG/MR 授权分层；MCP 校验不能阻止可绕过它的任意 shell，需要同时约束环境、网络和凭据访问 |
| inbox / 运行中追问 | 活跃 turn 尝试 turn/steer，空闲会话用 turn/start | SAC 保留队列、关联和确认状态；处理 turn 结束竞态，不能将 API 接受视为业务已完成 |
| Web 宿主 / 客户端插件注入 | 自建 SAC 本地服务与 Web 前端 | 不要求修改官方 Codex 前端 |
| 任务调度 / Effects / Outbox / Source | SAC 自己保留并适配 SAD 的业务机制 | 不由 Codex 会话历史或原生子代理自动替代 |

官方依据（2026-11-03 核对）：[App Server](https://learn.chatgpt.com/docs/app-server) 列出会话启动/恢复、turn 启动/追加输入/中断、压缩、审批请求、Skills 和工具事件；dynamicTools 为实验性。这些证据支持执行与会话层复用，不证明 dsh 宿主 API 的完整等价。具体能力应按锁定 Codex 版本生成 schema 并验证，不依赖未公开内部对象。

Router 继续由 Codex 理解请求、作出分派判断；SAC 程序验证该判断是否符合权限、保存决定并创建或恢复 Worker。业务调度与模型判断分开不改变 BASE-02。

AUTH-02 分层权限已确认：运行时命令/文件/网络权限与业务任务动作授权分别实施，不以提示词代替限制，也不将 dsh guard/restrict 原样映射为同名 Codex 能力。具体权限配置和可绕过路径仍需设计、验证。

CONN-04 回写恢复原则已确认：由 SAC 维护 CODING 评论发送记录并核对远端状态，会话 resume 不代表已恢复回写结果。第 5 节 Outbox、稳定标识和超时核对仍是具体方案建议，需接口实测；本次认可不扩大为所有可靠性参数已确认。

AUTH-03 已确认：将 push、MR、回评作为受控业务动作，Worker 提交请求，SAC 校验授权、目标、当前任务状态和适用的验证证据，再执行和记录结果。预先授权的动作可自动执行，不要求逐次人工确认。实现需要限制 Worker 对写凭据及旁路执行的访问；单纯暴露 MCP 工具不构成隔离。自动交付范围见 AUTH-04，隔离和防绕过机制尚未确定，不能把产品架构确认当作当前会话实际远端操作授权。

AUTH-04 已确认：指定账号的明确修复指令，在仓库/分支映射及隔离策略明确、授权检查和规定验证通过后，允许自动推送任务修复分支、创建或更新原 MR，并在可评审后回评用户可见结果。MR 目标为个人开发主分支；不自动合并、发布或关闭 BUG。验证不足、目标不明、授权不符或需求歧义时记录阻塞，不通过自动重试放宽权限或验证要求。分析请求保持只分析。独立分支方式另按 BRANCH-04 确认；同步策略另按 BRANCH-03 确认；正常结果回评按 CONN-05 执行；受理确认按 ACK-01 发送。需要用户决定或补充材料时按 ATT-01 提问，其他异常通知权限待定，CODING MR 接口待验证。当前会话仍仅讨论与维护文档。

BRANCH-04 已确认两层隔离：用户另行准备 BUG 修复专用 clone，与日常开发 clone 分离；SAC 以专用 clone 为 Git 管理基础，在其目录下为各 BUG 的本次修复任务建立独立分支和 linked worktree。两个 clone 使用同一 CODING 远端；不创建新远端仓库，不要求 fork。修复从 BRANCH-02 的远端开发分支提交开始，MR 指向该开发分支，不从专用 clone 当前检出的任意本地分支猜测基线。评审期间的补充反馈沿用原分支和当前 MR，关闭后重开 BUG 的处理见 MR-02。本轮不执行 clone 或创建目录。

实现建议：专用 clone 承担 fetch 和 worktree 管理，Worker 的 cwd 指向自己的任务 worktree。保存任务到远端仓库身份、专用 clone 路径、分支、worktree 和 MR 的映射；创建前核对专用 clone 的远端与配置相符，重复受理时复用已有映射。Git 管理操作需要协调，不能把多个 worktree 当作彼此独立的 Git 仓库。由于 worktree 位于专用 clone 的目录下，需避免父工作区的暂存、递归扫描或构建误包含任务子目录；具体目录命名与本地排除配置在实现前确定，不自动修改业务仓库的共享忽略规则。

worktree 隔离文件，不替代进程权限、凭据和网络限制；依赖安装、共享构建资源、跨 BUG 依赖、任务重开和清理规则仍需明确。用户尚未提供专用 clone 的路径。

独立 CLI 0.166.0 的已完成 turn 跨进程恢复已通过 ENG-02/REC-01 核验。待授权实现后的其他最小兼容验证包括：工具执行与进度事件；运行中追问及 turn 结束竞态；暂停/审批与取消后的进程状态；角色工具及权限边界；运行中断线、崩溃及重启后核对投递和外部动作。未经验证，不承诺和 dsh 相同的消息确认、停止后台进程或恢复保证。

UI-01 自建前端方向、BRANCH-02、BRANCH-04 专用 clone 与任务 worktree、HOST-01 架构选择原则、AUTH-02 分层权限、AUTH-03 动作执行入口、AUTH-04 自动交付及 CONN-04 回写恢复原则已确认。BRANCH-03 已确认首次交付前 rebase、MR 后冻结、冲突时追加 merge commit 并重新验证后普通推送。CONN-05 正常结果回评文案已确认。ACK-01 受理确认已确认。ATT-01 人工提问和 ATT-02 直接回复后继续原任务均已确认，CODING 回复关联待实测。最终 MR 合并方式及其他消息规则继续讨论，不在本轮写代码。
