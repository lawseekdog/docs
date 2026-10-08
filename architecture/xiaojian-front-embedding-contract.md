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
> - 官方侧：`ai-engine-v2/packages/xiaojian/src/client.ts`（`parseFrontCommand`、`hostedWorkspaceNavigation`、事件发布点）及其测试 `ai-engine-v2/packages/xiaojian/tests/*.spec.ts`
> - 宿主侧：`frontend/src/features/assistant/xiaojianBridgeMessages.ts`、`DshConversationSurface.tsx`、`useDshSurfaceConnection.ts`、`dshWebToken.ts`
> - 锁定测试：`frontend/tests/unit/xiaojian-page-mode-mounted.test.mjs` 等
>
> 核对基线：2026-10-07 的 `ai-engine-v2` main（官方 DSH `0.2.0-rc.2`）与 `frontend` main；§4 发送结果按 2026-10-08 的 ai-engine-v2#519 与 frontend#353。
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
| 入口卡片、启动器、材料选择 | Front |

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

**命令全集（10 条）。** 「精确键集」除 `command` / `contract_version` / `nonce` 外：

| 命令 | 精确键集 | 作用 |
| --- | --- | --- |
| `set_mobile_view` | `view` (`home`\|`conversation`) | 选择移动端呈现 |
| `report_surface_state` | `mobile` (boolean) | 请求回报 `surface-state.v1`（回显 nonce） |
| `set_surface_presentation` | `presentation` (`page`\|`overlay`) | 设定根节点 `data-xiaojian-presentation`；`page` 显示官方侧栏 |
| `set_history_scope` | `scope` (`HistoryScope` \| `null`) | 限制官方会话列表范围；`null` 列出全部 |
| `focus_composer` | — | 聚焦官方 composer |
| `insert_document_reference` | `session_id`, `reference` | 仅写入官方输入层；**不改草稿、不发送**；nonce 去重防重放 |
| `open_session` | `session_id` | 经官方导航状态打开该 Session |
| `start_new_session` | `intent`, `references`, `sessionTitle`, `workspaceKey`, `workspaceTitle` | **唯一能创建 Session 的命令**，经 Host Remote 铸造预置绑定会话 |
| `set_session_search` | `open`；或 `open`, `query`（两种精确键集） | 开合官方会话搜索 |
| `submit_message` | `intent`, `expected_agent_preset`, `text`, `materials`，**至多一个** target：`library_template` / `contract_review_decision_target` / `contract_review_batch_decision_target` / `document_action_target` / `document_revision_decision_target` | 经官方会话服务发送 prompt 与材料 |

**`submit_message` 的附加约束（已实现）：** `text` 非空且 `text === text.trim()`（首尾空白即拒）；
`expected_agent_preset` 必须属于 `AGENT_PRESET_BY_INTENT` 的取值集合。

**现状提示（不是规范，是既成事实）：** 线上键名不统一——`session_id` 为 snake_case，而
`sessionTitle` / `workspaceKey` / `workspaceTitle` 为 camelCase。新命令请勿扩大这种不一致。

## 4. 事件（官方 Client → Front）

全部为 `lawseekdog.dsh.*.v1`，经 `window.parent.postMessage(msg, parentOrigin)` 发出。
Front 侧解析器为 `xiaojianBridgeMessages.ts` 的 `parseXiaojianBridgeMessage`，同样
**按键集精确匹配**，键集不符即丢弃。

| 事件 | 载荷 | 含义 |
| --- | --- | --- |
| `surface-ready.v1` | — | 官方界面就绪；**仅在 composer 唯一存在后发出** |
| `attention.v1` | `attention` (`idle`\|`executing`\|`decision_required`\|`failed`) | 官方注意状态 |
| `submission-accepted.v1` | `session_id`, `nonce` | **准入回执**：Host 已受理该 prompt |
| `submission-returned.v1` | `session_id`, `nonce`, `reason`, `draft_restored` | **未发送定论**：原因已显示在该会话的官方输入框；`draft_restored` 表示原文已放回官方输入框 |
| `reference-inserted.v1` | `session_id`, `nonce` | 文档引用已插入 |
| `session-ready.v1` | `session_id` | 会话就绪 |
| `session-selected.v1` | `session_id` | 经历史来源选中会话 |
| `session-search-closed.v1` | — | 官方搜索已关闭 |
| `surface-error.v1` | `error` | 官方错误码 |
| `surface-state.v1` | `nonce`, `surface_ready`, `session_id`, `display` | 对 `report_surface_state` 的权威回报 |
| `business-navigation.v1` | `action: 'open'`, `path` | 请求 Front 做业务导航（path 必须同源） |
| `mobile-navigation.v1` | `action: 'conversation'\|'history'`；或 `action: 'launcher'`, `intent` | 移动端导航请求 |
| `mobile-display.v1` | `source`, `isConversation`, `hasDraft` | 移动端显示投影 |
| `home-launch.v1` | `intent` | 主页面首页面板把业务入口交给 Front 启动器 |

**发送结果（已实现，2026-10-08）：** 一条 `submit_message` 只会得到一个定论。
`submission-accepted.v1` 表示官方 `Session.prompt` 已受理；从此该请求只记录在官方会话中（原生
submission echo），Front 立即放手。`submission-returned.v1` 表示请求在受理前结束、未发送：Host 照官方
输入框默认 sink 的语义处理——原因经 `input.notify` 显示在该会话的输入框上；不带专业目标且输入框未被
改动时，用 `input.setDraft` 放回原文（`draft_restored: true`）。受理后若原生 echo 未被观察到即退役，
Host 同样在输入框中提示并放回原文，不再向 Front 发事件。不存在"结果待核对"状态；
`submission-observed.v1`、`submission-unconfirmed.v1` 与 15 秒观察计时已删除。
Host 在会话绑定后立即登记原生 pending echo（`beginHostedSubmission`），律师的消息先出现在会话中，
目标绑定与材料接纳随后在其下进行，prompt 复用同一 request identity；prompt 之前的任何失败都会让 echo 退役。
启动器请求（无专业业务）从点击开始到送达，Front 不覆盖官方 surface；新会话未能打开时只在会话旁提示
"新会话未能打开"，由律师选择重新开始或关闭。
`submit_message` 在入队前被 Host 拒收（`submit-requires-current-session`、`front-command-*`）时，
Host 仍发 `surface-error.v1`，Front 按确定未发送处理。

## 5. 硬不变式

| # | 不变式 | 状态 | 锁定位置 |
| --- | --- | --- | --- |
| 1 | 导航 fail-closed：无隐式创建、无重试、无合成状态；Front 是 `start_new_session` 的唯一显式入口 | 已实现 | `hostedWorkspaceNavigation`（拒绝 `workspace-navigation-requires-explicit-front-action`）；`desktop-page.spec.ts` |
| 2 | 打开/关闭/刷新官方界面**必须只读** | 已实现 | 同上 |
| 3 | 决策由官方 Approval 拥有，Front 不合成点击/决策/会话状态 | 已实现 | `client.ts` Approval 投影注释 |
| 4 | 主页面不得覆盖或隔离官方 surface（见 §6） | 已实现 | `xiaojian-page-mode-mounted.test.mjs` |
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

> The main page's session column is the official DSH sidebar inside the surface … **it draws no
> rail and no home cover of its own.**

该测试逐条断言主页面下：iframe **无** `inert`、className **不含** `invisible`、
`[data-xiaojian-home]` 为 `null`、`[data-xiaojian-session-rail]` 为 `null`、无
`[data-xiaojian-new-conversation]`、无 `[aria-label="历史记录"]`、`set_history_scope` 为 `null`。

**结论：主页面模式已实现且被锁定，是合规的基准形态。**

### 6.2 覆盖层模式：布局有规定，覆盖层内的自绘层不合规（**待整改**）

共享规范 G「Front、Xiaojian 与文书」规定了布局：桌面端 `XiaojianOrb` 展开为**右侧覆盖
workspace**（即 `presentation: 'overlay'`），移动端 `/m/xiaojian` 全屏。所以小简整体浮在业务页上
是规定的形态。

未规定的是：在这个覆盖面板**内部**，Front 能不能在官方 iframe 上叠自己的界面。用户 2026-10-04
明确决定：**agent 交互的界面归官方嵌入 surface，Front 不得自行改造**。自有 UI 只能与它并列，
不能盖在它上面，也不能禁用它。这条决定尚未写入共享规范。

按这条决定，`DshConversationSurface.tsx` 现有的以下自绘层**不合规**：

| 层 | 形态 |
| --- | --- |
| 首页卡片墙 `[data-xiaojian-home]`（仅非主页面、非移动端） | `absolute inset-0 z-10`，盖在 iframe 上 |
| 业务会话打开门（`data-xiaojian-open-gate`，仅专业页面打开业务会话的 opening/failed/cancelled；启动器请求与发送期间已不覆盖，2026-10-08） | `absolute inset-0 z-[15]` |
| 新建会话对话框 | `absolute inset-0 z-30` |
| 连接中封面 `[data-xiaojian-connecting]` | `absolute inset-0 z-10` |
| `blockConversationFrame` 路径 | 给 iframe 加 `inert`，并加 `invisible pointer-events-none` |

`inset-x-3 top-3 z-20` 的两条提示条不遮挡会话本体，属 §6.4 的并列 UI。

### 6.3 移动端 home：**未规定**

移动端 home 态下 Front 自绘 `MobileStartComposer`（`frontend/src/apps/mobile/MobileLauncherProvider.tsx`），
替代官方 composer 的呈现。官方侧不存在对应的禁止条款，宿主侧也无锁定测试。按 §6.2 的决定
同样需要核对是并列还是替代。

### 6.4 明确允许

与官方 surface **并列**、不覆盖它的自有 UI：启动器、全局顶栏、`inset-x-*` 提示条、自有对话框。
它们不改变官方界面的可达性。

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
  丢弃排队请求、不重放，并释放桥。Front 显示"新会话未能打开"（本次请求尚未发送，原文和材料保留），
  只有律师点"重新开始"才再建一个会话；重新连接只恢复连接，不重建、不重发。

**状态：** 已合并。准入前的这一段有了上界和原因；准入后的定论见 §4「发送结果」。

### 7.2 覆盖层与移动端 home 的整改与条文

§6.2 的自绘层需要整改为并列形态，或改由官方 surface 承担。§6.2 的决定也需要写入共享规范
G「Front、Xiaojian 与文书」。§6.3 待核对。以上均为**规划中**，尚无工单。

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
| 事件解析与 origin 校验（§4） | `frontend/tests/unit/xiaojian-matter-discussion-mounted.test.mjs`、`xiaojian-standalone-entries-mounted.test.mjs` |
| 导航 fail-closed（§5.1） | `ai-engine-v2/packages/xiaojian/tests/desktop-page.spec.ts`、`client-lifecycle.spec.ts` |
| 新建会话准入前的上界（§7.1） | `client-lifecycle.spec.ts`（#422 合并后） |
| 握手与令牌（§5.6） | `frontend/tests/unit/xiaojian-dsh-web-token.test.ts` |

## 9. 相关文档

- 共享规范 G「Front、Xiaojian 与文书」：`lawseekdog-agent-skills/workspace-guidance/guide/business-flows.md`
- [系统架构](overview.md)
- `ai-engine-v2/docs/external-integrations.md`（第三方组件准入，与本契约不同层）
