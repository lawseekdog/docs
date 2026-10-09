---
title: 小简前端嵌入契约
parent: 架构
nav_order: 7
---

# 小简前端嵌入契约（Front ↔ 官方 DSH Client）

本契约定义**宿主前端（Front）**与**官方 DSH Web Client（iframe 内的 `xiaojian` 插件）**之间
的边界。它回答两个问题：两端可以说什么；嵌入方可以画什么、不可以画什么。

> **状态：已实现**，逐条标注例外（`未规定` = 契约对该情形无任何规定，不等于允许）。
> 本文以代码为准。权威来源：
> - 官方侧：`ai-engine-v2/packages/xiaojian/src/client.ts`（`parseFrontCommand`、`hostedWorkspaceNavigation`、事件发布点）、`surface-wire.ts`、`composer-notice.ts`、`surface-panel.ts`（Front 通知与面板的线格式和呈现），及其测试 `ai-engine-v2/packages/xiaojian/tests/*.spec.ts`
> - 宿主侧：`frontend/src/features/assistant/xiaojianBridgeMessages.ts`、`xiaojianSurfaceWire.ts`、`xiaojianSurfaceView.ts`、`useXiaojianSurfaceSync.ts`、`XiaojianBusinessSessionGate.ts`、`DshConversationSurface.tsx`、`useDshSurfaceConnection.ts`、`dshWebToken.ts`
> - 锁定测试：`frontend/tests/unit/xiaojian-page-mode-mounted.test.mjs`、`xiaojian-surface-no-cover-mounted.test.mjs` 等
>
> 核对基线：2026-10-09 的 `ai-engine-v2` main（官方 DSH `0.2.0-rc.2`，含 ai-engine-v2#562）与 `frontend` main（含 frontend#388）；§4 发送结果按 2026-10-08 的 ai-engine-v2#519 与 frontend#353。
> 本文只写 bridge 的线协议与嵌入现状；小简的业务规则（入口、不自动发送、布局）以共享规范
> G「Front、Xiaojian 与文书」为准，本文不重复。**引用代码时以符号名为准。**

## 1. 所有权

官方 DSH 是唯一的 agent 控制平面（见 `ai-engine-v2/AGENTS.md`）。

| 真值 | 归属 |
| --- | --- |
| Session / Workspace / Conversation 状态 | 官方 Client |
| 决策（Approval / Question）及其状态 | 官方 DSH Approval |
| 发送的受理、观察与未确认 | 官方 Client，经事件发布 |
| 业务数据 | 各 Owner 服务 |
| 入口卡片的呈现 | 官方 Client：桌面在空白会话的 hero 内（只切换该空白会话的业务模式），移动端在「工作」页（经 `mobile-navigation.v1` 交给 Front 启动器） |
| 入口启动、材料选择；Front 通知与面板的内容和业务逻辑 | Front（呈现归官方 Client，见 §6.2） |

**推论性规则：** 凡属官方侧的真值，Front 不得自建第二来源。注意这条推论的**限定范围**：
Front 现在那个 25 秒提示之所以不构成违规，是因为它并不宣称任何状态（只说"仍在等待"），
而不是因为它在原则上是允许的——它之所以存在，是因为官方侧在该窗口内是沉默的（§7.1）。
**正确的修法是给官方侧补上界，不是让 Front 把这段沉默讲得更清楚。**

## 2. 传输

**唯一通道是 `postMessage`。** 已实现，且无例外：

- 没有任何代码读写小简 iframe 的 `contentDocument` / `contentWindow.document`
  （`contentDocument` 只出现在文书预览 `SourceDocxDocumentView.tsx` 自己的 iframe）；
- 没有任何代码 monkey-patch 小简 iframe 的 window；
- 没有任何 Front 的 CSS 指向 iframe 内部。

这不只是洁癖：官方 Client 用选择器判定自身 UI 是否就绪（composer **必须唯一**匹配
`[data-composer-input]`，`surface-ready` 在该元素存在前不发出）。**向该文档注入任何元素都可能
破坏官方就绪判定**，因此注入属禁止项，而非风格问题。

双向身份校验：Front 侧只接受 `event.origin === 约定 origin && event.source === iframe.contentWindow`
的消息（`xiaojianBridgeMessages.ts` 的监听器）；官方侧由 `requireParentOrigin` 从自身
`window.location.origin` 推导父 origin，**从不使用 `'*'`**，且仅在确实被嵌套时启用桥。

## 3. 命令信封（Front → 官方 Client）

```
{ contract_version: 'lawseekdog.front.command.v1', command: <name>, nonce: <int > 0>, ... }
```

**硬规则（已实现）：**

1. `nonce` 必须是 `> 0` 的安全整数；
2. **键集必须精确匹配**——多一个键即整条命令被拒；
3. `contract_version` 不匹配 → 解析返回 `null`；官方 Client 另经 `frontCommandRejection`
   归类错误码（契约前缀可识别但版本不符 → `front-command-contract-unsupported`；
   命令未知或键集不符 → `front-command-invalid`；第三方 `contract_version` → 静默忽略）。

**命令全集（13 条）。** 「精确键集」除 `command` / `contract_version` / `nonce` 外：

| 命令 | 精确键集 | 作用 |
| --- | --- | --- |
| `set_mobile_view` | `view` (`home`\|`conversation`) | 选择移动端呈现 |
| `report_surface_state` | `mobile` (boolean) | 请求回报 `surface-state.v1`（回显 nonce） |
| `set_surface_presentation` | `presentation` (`page`\|`overlay`) | 设定根节点 `data-xiaojian-presentation`；`page` 显示官方侧栏 |
| `set_history_scope` | `scope` (`HistoryScope` \| `null`) | 限制官方会话列表范围；`null` 列出全部 |
| `focus_composer` | — | 聚焦官方 composer |
| `set_composer_notice` | `notice`（`null` 或 §3.1 的精确形状） | 在当前显示会话（空白会话与已开始会话皆然）的输入区上方显示**至多一条** Front 通知；新命令替换，`null` 移除；点击其操作**不**清除通知（由 Front 决定），只发 `front-notice-action.v1` |
| `set_surface_panel` | `panel`（`null` 或 §3.1 的精确形状） | 在主区**取代会话与输入区**显示一个面板（桌面主页面、覆盖层与移动端同），其后的会话既不可见也不可达；一次一个，新命令替换，`null` 移除后会话重新显示；**只由 Front 清除**，官方侧不自行撤下 |
| `insert_document_reference` | `session_id`, `reference` | 仅写入官方输入层；**不改草稿、不发送**；nonce 去重防重放 |
| `open_session` | `session_id` | 经官方导航状态打开该 Session |
| `start_new_session` | `intent`, `references`, `sessionTitle`, `workspaceKey`, `workspaceTitle`；另可带入口键 `composer_hint` / `initial_draft` / `initial_materials` / `library_template`（可选；`library_template` 仅限 `document_draft`） | **唯一能创建 Session 的命令**，经 Host Remote 铸造预置绑定会话；挂起期间官方界面显示忙碌面板（§4 发送结果） |
| `preselect_entry` | 同 `start_new_session` | 点击入口时先复用**本页为同一业务范围**（`workspaceKey` 与 `references` 相同）建立、仍空白且未绑定模板的当前会话，经官方选择切换其业务模式，不再堆积空会话；无可复用会话时等同 `start_new_session` |
| `set_session_search` | `open`；或 `open`, `query`（两种精确键集） | 开合官方会话搜索 |
| `submit_message` | `intent`, `expected_agent_preset`, `text`, `materials`，**至多一个** target：`library_template` / `contract_review_decision_target` / `contract_review_batch_decision_target` / `document_action_target` / `document_revision_decision_target` | 经官方会话服务发送 prompt 与材料 |

**`submit_message` 的附加约束（已实现）：** `text` 非空且 `text === text.trim()`（首尾空白即拒）；
`expected_agent_preset` 必须属于 `AGENT_PRESET_BY_INTENT` 的取值集合。

**现状提示（不是规范，是既成事实）：** 线上键名不统一——`session_id` 为 snake_case，而
`sessionTitle` / `workspaceKey` / `workspaceTitle` 为 camelCase。新命令请勿扩大这种不一致
（`set_composer_notice` / `set_surface_panel` 的全部键均为 snake_case）。

**已删除：** `open_home`。桌面不再有独立首页面板，空白会话的 hero 就是唯一起始画面（§6.2）；收到该命令
即与任何未知命令同样按 `front-command-invalid` 处理。

### 3.1 Front 通知与面板（`set_composer_notice` / `set_surface_panel`，已实现）

Front 保有这些状态的内容与业务逻辑，官方 Client 负责呈现（§6.2）。两个对象在**每一层**
（命令、`notice` / `panel`、动作、选项、字段、字段选项）都按精确键集解析，多或少一个键即整条命令按
`front-command-invalid` 拒收。

```
notice: { code, tone, title, detail, original_text, actions, locks_input }
panel:  { code, tone, busy, title, detail, error, original_text, materials, choices, fields, actions }
action: { action, label, primary }
choice: { choice_id, label, detail }
field:  { field_id, kind, label, value, placeholder, max_length, options: [{ value, label }] }
```

取值规则（`surface-wire.ts`、`composer-notice.ts`、`surface-panel.ts`）：

- `code` 符合 `/^[a-z][a-z0-9-]{0,63}$/`，是内容的身份：换 `code` 即另一个面板（焦点移到其标题）；`tone` 为 `info`｜`warning`｜`error`。
- `notice.title` 非空且已 trim；`panel.title` 非空。`detail` / `original_text` / `error` / `placeholder` 为字符串或 `null`（`original_text` 以可选中复制的形式显示律师未发送的原文）。
- 动作 `action` 符合 `/^[a-z][a-z0-9-]{0,31}$/` 且在同一对象内唯一，`label` 非空；通知至多 3 个，面板至多 4 个。
- `locks_input: true`：通知显示期间输入框不可输入、不可发送（经官方 composer 阻断，通知标题作为原因）。
- `busy: true`：标题旁显示加载指示，并**禁用选项与字段**（不禁用动作）。
- `materials`（未发送材料的文件名）、`choices`、字段的 `options` 各至多 50 项，文件名与各 label 非空，`choice_id` 非空且唯一，选项 `value` 唯一；`fields` 至多 3 个，`field_id` 符合 `/^[a-z][a-z0-9_]{0,31}$/` 且唯一，`label` 非空。
- `kind: 'select'` 必须有选项，`max_length` 为 `null`，`value` 须是某选项的 `value`，或在 `placeholder` 非 `null` 时为空串（表示尚待选择）；`kind: 'text'` 无选项，`max_length` 为 `null` 或 ≥ 1 的安全整数，且 `value` 不得超过它。
- 字段值是面板内的本地 UI 状态：`code` 或任一字段的初始 `value` 变化时按新 `value` 重新初始化，其余变化保留律师已编辑的内容；值只随动作上报。

## 4. 事件（官方 Client → Front）

全部为 `lawseekdog.dsh.*.v1`，经 `window.parent.postMessage(msg, parentOrigin)` 发出。
Front 侧解析器为 `xiaojianBridgeMessages.ts` 的 `parseXiaojianBridgeMessage`，同样
**按键集精确匹配**，键集不符即丢弃。

| 事件 | 载荷 | 含义 |
| --- | --- | --- |
| `surface-ready.v1` | — | 官方界面就绪；**仅在 composer 唯一存在后发出** |
| `attention.v1` | `attention` (`idle`\|`executing`\|`decision_required`\|`failed`) | 官方注意状态 |
| `submission-accepted.v1` | `session_id`, `nonce` | **准入回执**：Host 已受理该 prompt |
| `submission-returned.v1` | `session_id`, `nonce`, `reason`, `draft_restored` | **未发送定论**：原因已由官方界面显示在该会话的输入区；`draft_restored` 表示原文已放回官方输入框（`false` 时原文在官方侧的输入区通知里，见下） |
| `reference-inserted.v1` | `session_id`, `nonce` | 文档引用已插入 |
| `session-ready.v1` | `session_id` | 会话就绪 |
| `session-selected.v1` | `session_id` | 经历史来源选中会话 |
| `session-search-closed.v1` | — | 官方搜索已关闭 |
| `surface-error.v1` | `error` | 官方错误码；官方界面同时在当前会话的输入区以「会话」用语告知律师（不显示原始错误码；取消类导航不提示；尚无会话显示或面板取代会话时无处显示），Front 仍据此驱动自己的状态 |
| `surface-state.v1` | `nonce`, `surface_ready`, `session_id`, `display` | 对 `report_surface_state` 的权威回报 |
| `business-navigation.v1` | `action: 'open'`, `path` | 请求 Front 做业务导航（path 必须同源） |
| `mobile-navigation.v1` | `action: 'conversation'\|'history'`；或 `action: 'launcher'`, `intent` | 移动端导航请求 |
| `mobile-display.v1` | `source`, `isConversation`, `hasDraft` | 移动端显示投影 |
| `front-notice-action.v1` | `code`, `action` | 律师点了 Front 通知（`set_composer_notice`）上的操作；通知不会因此消失 |
| `front-panel-action.v1` | `code`, `action`, `choice_id`, `values` | 律师点了 Front 面板（`set_surface_panel`）的操作或选项：点选项时 `action` 为 `choose`、`choice_id` 为所选项，其余 `choice_id` 为 `null`；`values` 为字段的当前值（`field_id` → 字符串），无字段时 `{}` |
| `new-session-requested.v1` | — | 律师点了官方「新会话」：仅在 Front 已声明桌面呈现（`set_surface_presentation`）、非移动端、且无请求或新建进行中时发出；官方侧自己不创建会话，其余情形仍按 fail-closed 拒绝。Front 照自己「新会话」按钮的逻辑处理：无专业业务时用 `start_new_session` 开空白会话（咨询意图，会话标题「法律咨询」），专业页面内开该业务的另一段会话；移动端 Front 不响应 |

**发送结果（已实现，2026-10-08）：** 一条 `submit_message` 只会得到一个定论。
`submission-accepted.v1` 表示官方 `Session.prompt` 已受理；从此该请求只记录在官方会话中（原生
submission echo），Front 立即放手。`submission-returned.v1` 表示请求在受理前结束、未发送：Host 照官方
输入框默认 sink 的语义处理——不带专业目标且输入框未被改动时，用 `input.setDraft` 放回原文，并经
`input.notify` 把原因显示在该会话的输入框上（`draft_restored: true`）；带专业目标（由其页面重试）或输入框
已被改动时不放回，改由官方界面在该会话的输入区显示一条自有通知，写明原因并附可选中复制的原文，律师
关闭前一直保留（`draft_restored: false`）。受理后若原生 echo 未被观察到即退役，Host 同样如此处理，
不再向 Front 发事件。不存在"结果待核对"状态；
`submission-observed.v1`、`submission-unconfirmed.v1` 与 15 秒观察计时已删除。
Host 在会话绑定后立即登记原生 pending echo（`beginHostedSubmission`），律师的消息先出现在会话中，
目标绑定与材料接纳随后在其下进行，prompt 复用同一 request identity；prompt 之前的任何失败都会让 echo 退役。
启动器请求（无专业业务）从点击开始到送达，Front 不覆盖官方 surface；新会话未能打开时，Front 经
`set_surface_panel` 让官方界面在会话的位置显示 `launcher-failure` 面板（"新会话未能打开"，未发送，原文和
材料名保留），由律师选择重新开始（或重新登录）或关闭（2026-10-09）。`start_new_session` 挂起期间
（从收到命令到新会话打开或失败），官方界面自己在主区显示忙碌面板"正在新建会话…"（小简自有内容，与
`set_surface_panel` 同一面板组件），新会话打开后切到该会话；上一个会话不会在此期间露出（2026-10-09）。
`submit_message` 在入队前被 Host 拒收（`submit-requires-current-session`、`front-command-*`）时，
Host 仍发 `surface-error.v1`，Front 按确定未发送处理：原文由 Front 保留，以 `refused-submission`
通知显示在会话输入区上方，律师关闭即止（2026-10-09）。

## 5. 硬不变式

| # | 不变式 | 状态 | 锁定位置 |
| --- | --- | --- | --- |
| 1 | 导航 fail-closed：无隐式创建、无重试、无合成状态；Front 是 `start_new_session` 的唯一显式入口 | 已实现 | `hostedWorkspaceNavigation`（拒绝 `workspace-navigation-requires-explicit-front-action`）；`desktop-page.spec.ts` |
| 2 | 打开/关闭/刷新官方界面**必须只读** | 已实现 | 同上 |
| 3 | 决策由官方 Approval 拥有，Front 不合成点击/决策/会话状态 | 已实现 | `client.ts` Approval 投影注释 |
| 4 | Front 不得覆盖或隔离官方 surface：除连接中封面外，不在其上画任何层；iframe 仅在未激活或封面显示时 `inert`；Front 的状态由官方 surface 以通知与面板呈现（见 §6） | 已实现（主页面与覆盖层；ai-engine-v2#562、frontend#388） | `xiaojian-page-mode-mounted.test.mjs`、`xiaojian-surface-no-cover-mounted.test.mjs` |
| 5 | 每条提交只有一个由官方给出的定论（受理或带原因的退回），Front 不合成发送结果 | 已实现（ai-engine-v2#519、frontend#353）；准入前的上界见 §7.1 | §4 事件链；`client-lifecycle.spec.ts`、`hosted-submission-return.spec.ts`、`xiaojian-handoff-barrier.test.ts` |
| 6 | origin 精确匹配；hosted 令牌 URL 与前端同源，且不得携带前端 origin 查询参数 | 已实现 | `dshWebToken.ts`；`requireParentOrigin` |
| 7 | 打开界面**不得**自动发送 | 已实现 | `shouldAutoSendPromptOnXiaojianOpen()` 恒返回 `false` |

## 6. 嵌入方边界（本节是契约的核心）

### 6.1 主页面模式：明文禁止覆盖，且有测试锁定

官方侧明文（`client.ts`，`hostedWorkspaceNavigation` 文档注释）：

> The stock Web workspace UI owns an eager navigation policy … **forbidden for the hosted
> Xiaojian surface where opening, closing and reloading must be read only. Front is the only
> explicit entry into `start_new_session`** … Keep the official Session/Workspace Controllers
> and Conversation surface; replace only the Web navigation policy with a fail-closed carrier.
> **There is no retry, implicit create, alternate Session selection or synthetic state.**

宿主侧明文（`xiaojian-page-mode-mounted.test.mjs`）：

> The main page's session column is the official DSH sidebar inside the surface, and its blank
> Session hero is the one start screen. The Front only reports where it shows the surface and
> answers the official 新会话 by starting a blank Session; **it draws no rail and no home cover of
> its own.**

官方「新会话」在桌面只向 Front 请求一个空白会话（`requestNewSession` 发 `new-session-requested.v1`），
自身仍不创建（§4）。

该测试逐条断言主页面下：`assertNothingCoversFrame`（iframe 容器内无绝对或固定定位的覆盖层、iframe **无**
`inert`、className **不含** `invisible` / `pointer-events-none`）、`[data-xiaojian-home]` 为 `null`、
`[data-xiaojian-session-rail]` 为 `null`、无 `[data-xiaojian-new-conversation]`、无
`[aria-label="历史记录"]`、`set_history_scope` 末次为 `null`；一次 `new-session-requested.v1`
恰好得到一条咨询意图的 `start_new_session`，且不发 `preselect_entry` 与已删除的 `open_home`。

**结论：主页面模式已实现且被锁定，是合规的基准形态。**

### 6.2 覆盖层模式与 Front 状态的呈现：Front 不在官方 surface 上画任何层（**已整改**，2026-10-09）

共享规范 G「Front、Xiaojian 与文书」规定了布局：桌面端 `XiaojianOrb` 展开为**右侧覆盖
workspace**（即 `presentation: 'overlay'`），移动端 `/m/xiaojian` 全屏。所以小简整体浮在业务页上
是规定的形态。

在这个覆盖面板**内部**，Front 不得在官方 iframe 上叠自己的界面。用户 2026-10-04 明确决定：
**agent 交互的界面归官方嵌入 surface，Front 不得自行改造**；自有 UI 只能与它并列，不能盖在它上面，
也不能禁用它。2026-10-09 产品负责人确认该决定覆盖原先全部 Front 自绘层，包括业务会话选择面板
（取代 §7.2 原先的"暂不改动"）；唯一例外是连接中封面——它出现时官方界面尚未加载，没有可呈现的界面。
同一决定同步写入共享规范 G「Front、Xiaojian 与文书」。

**已实现（ai-engine-v2#562、frontend#388）：** Front 保有这些状态的内容与业务逻辑，官方 surface 负责
呈现，经 §3.1 的两条通道：

- 通知 `set_composer_notice`：至多一条，位于当前会话输入区上方，可锁定输入（`locks_input`）；
- 面板 `set_surface_panel`：取代会话与输入区，会话在其后既不可见也不可达。

Front 在 `xiaojianSurfaceView.ts` 把既有状态派生为至多一条通知和一个面板（面板优先级：新建会话 > 业务会话
选择 > 打开中 > 打开失败或绑定中 > 启动器失败；通知优先级：打开中 > 发送中断 > 入队前被拒），由
`useXiaojianSurfaceSync` 发出：内容变化时才发命令，消失时发 `null`，官方界面（重新）加载后重发；通知
先于面板出现、后于面板消失，会话在两个阻断状态之间不会短暂可达；Front 开始新 Session 时自行清掉
自己的面板。动作回到 Front 后，只认 Front 最后告知的那条通知或面板及其提供的操作与选项，`code` 或
动作不符即忽略。

| Front 状态 | 官方 surface 中的呈现 |
| --- | --- |
| 专业页面打开业务会话，或请求打开会话；目标会话尚未显示 | 面板 `opening-business` / `opening-request`（忙碌，「取消」） |
| 同上，官方 surface 已显示目标会话、业务关联尚未确认 | 通知 `opening-business` / `opening-request`（`locks_input`，「取消」）：会话可见、输入锁定，确认后通知移除 |
| 请求绑定中 | 面板 `open-binding`（忙碌，「取消」） |
| 请求打开失败 | 面板 `open-failed`（原文与材料名；重新登录、重新开始、重新打开会话或重试，按情形给出，另有「取消」） |
| 无专业业务的请求未能得到会话（未发送） | 面板 `launcher-failure`（原文与材料名；重新登录或重新开始，另有「关闭」） |
| 连接断开时请求已在官方 surface 内 | 通知 `interrupted-submission`（原文，「关闭」） |
| Host 在入队前拒收 `submit_message` | 通知 `refused-submission`（Front 保留的原文，「关闭」） |
| 业务会话选择：需选择、尚无会话、读取失败、关联不符 | 面板 `business-session`（各会话为选项，操作随情形） |
| 新建会话（专业事项会话或准备工作） | 面板 `new-session`（业务类型与会话主题字段，「开始」／「选择模板」与「取消」，校验错误在 `error`） |
| 连接中 | 封面 `[data-xiaojian-connecting]`：**唯一**的 Front 封面，iframe 此时 `inert`；已在打开的请求的「取消」放在封面上，因为界面尚未加载 |

被退回的请求与官方界面自身的错误由官方 surface 自己在输入区呈现（§4 的 `submission-returned.v1`、
`surface-error.v1`）；`surface-error.v1` 仍驱动 Front 自己的状态。iframe 仅在未激活或封面显示时 `inert`，
Front 不再用 `invisible` / `pointer-events-none` 隐藏它。取消请求会挂起连接，此时没有 surface 可呈现，
「已关闭本次会话请求」（`[data-xiaojian-open-cancelled]`）与连接错误一样在 iframe 的位置内联渲染，
不覆盖任何东西。

**起始画面（已实现）：** 桌面不再有独立首页面板，空白会话的 hero（居中，标题「今天需要处理什么？」，七个
入口卡片在输入框上方）是唯一起始画面；移动端保留「工作」页（同一标题、同一组七个入口）。`open_home` 与
`home-launch.v1` 已随首页面板一并删除。

### 6.3 移动端 home：已整改（原「未规定」）

移动端不再有 Front 自绘的启动窗口：`MobileStartComposer` 与 `MobileLauncherProvider` 已随发起窗口删除
（frontend `9e9404c3`，2026-10-09；`xiaojian-agent-first-hardcut.test.ts`、`xiaojian-mobile-pwa-contract.test.mjs`
断言其不存在）。「工作」页由官方 surface 渲染，点击入口经 `mobile-navigation.v1`（`action: 'launcher'`）
交给 Front；Front 以 `entryOnly` 派发——`start_new_session`（入口意图、标题、业务引用）后 `focus_composer`，
不发 `submit_message`——文字与材料在官方 composer 中输入。`MobileSubmissionProvider` 只保存那一条请求的
提交身份。

### 6.4 明确允许

与官方 surface **并列**、不覆盖它的自有 UI：全局顶栏、surface 头部（非主页面时位于 iframe 之前）、
iframe 之前的流内提示（如「继续原会话」）。它们不改变官方界面的可达性。原先 iframe 容器内的
`inset-x-*` 提示条与对话框已不存在，其内容由 §6.2 的通知与面板呈现；发起窗口（启动器）已删除。

## 7. 已知空白与风险

### 7.1 新建会话准入前可能永远没有回执（**已修复**：ai-engine-v2#422）

准入之前，官方侧原本没有任何计时器。

**触发条件（已用测试复现）：** `start_new_session` 创建并打开会话后，`runPending` 里的
`applyReadyGate` 要同时满足两件事：会话列表把新会话列为当前会话，并且它的 Agent-scoped
conversation 已存在。但它**只在会话列表通知时重查**。scoped conversation 由另一个官方
控制器注入（`waitForScopedConversation` 的注释写明了这一竞态），可能在列表最后一次通知
之后才出现。一旦如此：

- `actionInFlight` 永久为 `true`；
- 不发 `session-ready.v1`；
- 排队的 `submit_message` 停在 `submitPending` 的 `wait` 门；
- `submission-accepted.v1`、`surface-error.v1` **一个都不发**。

**后果：** Front 没有任何可据以失败的事件，只剩 `useXiaojianHandoff.ts` 的 25 秒提示，
而它声明"不会自动重复发送"，即无限等待。它盖住的是官方侧没有上界的一段窗口，
缺陷在于**没有原因、也没有终点**。

**修复：** [ai-engine-v2#422](https://github.com/lawseekdog/ai-engine-v2/pull/422)
（Issue #421）。新会话列为当前会话后，改用现成的有界等待 `waitForScopedConversation`
（10 秒）。

- 等到：照常完成创建，并发送排队的请求一次。
- 等不到：发 `surface-error.v1` `lawseekdog-xiaojian:conversation-scope-unavailable`，
  丢弃排队请求、不重放，并释放桥。Front 经 `launcher-failure` 面板（§6.2）显示"新会话未能打开"（本次请求尚未发送，原文和材料保留），
  只有律师点"重新开始"才再建一个会话；重新连接只恢复连接，不重建、不重发。

**状态：** 已合并。准入前的这一段有了上界和原因；准入后的定论见 §4「发送结果」。

### 7.2 覆盖层与移动端 home 的整改与条文（**已完成**，2026-10-09）

覆盖层内 Front 自绘的层已全部改由官方 surface 呈现（§6.2，ai-engine-v2#562、frontend#388）：除连接中
封面外，Front 不在官方 surface 上画任何层，也不再因这些状态使 iframe `inert` 或不可见。业务会话选择
面板原先按用户决定暂不改动；2026-10-09 产品负责人决定 2026-10-04 的决定覆盖原先全部 Front 自绘层
（含该面板），故一并改为面板呈现，该"暂不改动"已作废。该决定同步写入共享规范 G「Front、Xiaojian 与
文书」。移动端 home 的 Front 自绘启动窗口已删除（§6.3）。本节不再有**规划中**的剩余项。

### 7.3 对官方 Web Client 的补丁

迁到 DSH `0.2.0-rc.2` 后，原先改官方聊天渲染的 `dsh-assistant-settlement` 补丁已不存在。
`ai-engine-v2/pnpm-workspace.yaml` 的 `patchedDependencies` 中，只有一个补丁改官方 Web Client：
`patches/dsh-model-catalog-lazy`（改 `@deepseek-ai/dsh-client-ui-model-selection`）。它去掉冷启动、
重连和配置事件时对全局模型目录的三次预读（普通律师无权读，只会产生授权失败）；显式打开
选择器时仍走官方 `load()`。它改的是读取时机，不改界面呈现，也不在本契约的 bridge 范围内。

其余三个补丁（`dsh-history-search`、`dsh-session-ignorable-append`、`raw-tool-arguments`）作用于
服务端或模型适配层，与嵌入界面无关。

## 8. 如何验证

| 不变式 | 验证方式 |
| --- | --- |
| 主页面无覆盖、无隔离（§6.1） | `frontend/tests/unit/xiaojian-page-mode-mounted.test.mjs` |
| 命令键集与拒绝语义（§3） | `ai-engine-v2/packages/xiaojian/tests/client.spec.ts`、`mobile-bridge.spec.ts`、`desktop-page.spec.ts` |
| 通知与面板命令的精确键集与取值规则（§3.1；`open_home` 已删除） | `ai-engine-v2/packages/xiaojian/tests/surface-commands.spec.ts` |
| 面板与通知的呈现（§3.1、§6.2：忙碌禁用、字段重置、焦点、至多一条、锁定输入） | `ai-engine-v2/packages/xiaojian/tests/surface-panel.spec.ts`、`composer-notice.spec.ts` |
| 唯一起始画面、无首页面板残留（§4、§6.2） | `ai-engine-v2/packages/xiaojian/tests/start-screen.spec.ts`、`surface-panel.spec.ts` |
| 未发送定论与官方界面错误的呈现（§4） | `ai-engine-v2/packages/xiaojian/tests/hosted-submission-return.spec.ts`、`surface-failure.spec.ts` |
| 事件解析与 origin 校验（§4） | `frontend/tests/unit/xiaojian-matter-discussion-mounted.test.mjs`、`xiaojian-standalone-entries-mounted.test.mjs`；三个新事件的键集见 `xiaojian-surface-view.test.mjs` |
| Front 状态到通知与面板的映射与优先级（§6.2） | `frontend/tests/unit/xiaojian-surface-view.test.mjs` |
| Front 不在 iframe 上画任何层、只发变化的命令、动作只认当前通知或面板（§5.4、§6.2） | `frontend/tests/unit/xiaojian-surface-no-cover-mounted.test.mjs` |
| 导航 fail-closed（§5.1） | `ai-engine-v2/packages/xiaojian/tests/desktop-page.spec.ts`、`client-lifecycle.spec.ts` |
| 新建会话准入前的上界（§7.1） | `client-lifecycle.spec.ts`（#422 合并后） |
| 握手与令牌（§5.6） | `frontend/tests/unit/xiaojian-dsh-web-token.test.ts` |

## 9. 相关文档

- 共享规范 G「Front、Xiaojian 与文书」：`lawseekdog-agent-skills/workspace-guidance/guide/business-flows.md`
- [系统架构](overview.md)
- `ai-engine-v2/docs/external-integrations.md`（第三方组件准入，与本契约不同层）
