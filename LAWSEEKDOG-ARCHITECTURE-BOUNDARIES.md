# LawSeekDog 架构边界与技能使用基线

本文档记录当前代码对应的业务 Owner、状态轴、入口和会话边界。共享工程规则以 `lawseekdog-agent-skills/workspace-guidance/LAWSEEKDOG-WORKSPACE-GUIDANCE.md` 为准；本文只记录项目事实和审查结论。

## 活跃控制面

```text
Front / Xiaojian
  -> 官方 DSH Conversation Surface
  -> DSH Agent / Session / Tool / Question / Approval / Workflow / Subagent
  -> legal-* plugin Tool
  -> 唯一 Business Owner API
  -> Owner 数据库事务与真实 typed receipt
  -> DSH SessionEvent 与业务导航投影
```

`ai-engine-v2/packages/host` 只处理身份、组织、来源和 DSH 接入；`packages/xiaojian` 只处理会话界面、上传入口和业务导航；法律插件不直接连接业务数据库。

## Owner 与状态轴

| 业务对象 | 唯一 Owner | 核心状态轴 | 写入入口 |
|---|---|---|---|
| FirmIntake、Engagement、正式服务范围、利冲 | `firm-service` | intake / engagement / scope | `legal-service-intake`、`legal-engagement` Tool 经原生 Approval |
| Case、Proceeding、程序事件、期限 | `case-service` | case / proceeding / stage | `legal-case`、`legal-procedure` Tool 经 Owner API |
| Matter、Task、Product、交付绑定 | `matter-service` | matter、task、product | `legal-matter` 管通用事项，`legal-task` 管任务；专业插件只写所属成果 |
| Document Session、draft、review、publish、deliver | `document-workspace-service` | document lifecycle | `legal-document` Tool 经 Owner API |
| 文件、revision、checksum、访问边界 | `files-service` | file/material | 文件 API |
| 模板、schema、确定性渲染 | `templates-service` | template/render | 模板 API |
| DSH Session、SessionEvent、Approval、Question | 官方 DSH | session event lifecycle | DSH 官方控制面 |

Matter 是平级工作事项，按类型和明确范围区分，不存在父子事项。案件、程序与委托按实际关系关联多个事项；Task、Product 与交付绑定直接归属各自 Matter。每个事项独立核验任务、成果、授权范围履行及完成条件，不因另一个事项未终结而形成父子阻塞。同类型允许具有不同工作范围的多个事项，不按案件与类型自动合并。

Matter 的生命周期开放状态为 `draft -> provisional -> active`，终态包括 `completed`、`cancelled`、`declined`、`superseded`。Task、Document、Contract Review、Case Stage 和 DSH Session 是独立状态轴，不能互相推断或合并。

## 业务入口

业务模型统一为“入口 → Matter → 成果 → 律师决定”。首页及移动端提供新建案件、合同审查、文书起草、合规审查、尽职调查、谈判支持、法律咨询七个入口。明确开始后，先用一次原生 Approval 初始化该入口需要的业务记录并认领 Session 材料；普通咨询默认只讨论，明确要求保存时才建立咨询事项。律师拒绝初始化则继续 Session 讨论，不冒充已保存成果。

新建案件初始化包含 Intake、主体、Case、Proceeding 和 case_analysis Matter；利冲只在正式承接阶段进行。单项分析准备稿完成即自动保存到当前 Matter，综合分析引用已保存版本，不等全部分析结束才登记。合同审查先建立 contract_review Matter，审查标准绑定须确认，完整审查成果自动保存。已存在事项中的文书直接创建 DWS 草稿并自动保存版本，不再单独确认创建文书工作区；审阅、发布和交付仍需原生 Approval。已有案件中的继续分析是续办，不另设独立案件分析入口。

讨论阶段的工作记录就是 Session，不新增工作记录表。Matter 表达律师真实工作范围，文书只是交付物；律师函、法律意见书和合同是 legal-document 支持的文书种类，不是独立事项类型或插件。已有案件或合同事项中的文书不因插件或文书数量另建事项，DWS 是文书草稿与版本的唯一记录，Matter 只保留交付绑定。`legal-matter` 是内部工具能力，不是新的页面入口。客户、Case、Engagement 依据实际关系关联；首页独立工作不强制委托，客户页“新建工作”则必须选择该客户的有效委托并核验范围。文种不等于代理处分，DWS 定稿不等于签章或对外提交。

- 律所工作台：`frontend/src/apps/firm/firmRouter.tsx`，入口为 intake、engagement、documents、work、team 等经营路径。
- 管理端：`frontend/src/apps/admin/adminRouter.tsx`，只负责平台管理和 DSH 观测，不承载律师业务写入。
- DSH 业务入口：`ai-engine-v2/packages/legal-*` 的 typed Tool。自然语言请求先读取 Owner Context；只读请求直接返回 typed result；已有明确对象与用户工作范围内的普通内容保存无需逐次审批；创建对象、变更归属或正式范围、律师采纳／复核／完成决定和程序／期限动作，先形成完整影响并走官方 Approval。两类写入都由 Owner 鉴权并校验精确对象、版本和幂等。
- Matter 内部 API：`matter-service/src/main/java/com/lawseekdog/matter/api/controller`，只接受内部服务或律师端授权请求，不接受前端自行构造的业务 mutation。

## Session 与业务边界

Session 可以保存 `business reference`、当前 focus、来源材料和导航信息，但这些只是当前会话的引用选择。Matter、Case、Engagement、Document 和 Task 的长期关系与状态必须保存在各自 Owner 数据库中。Owner 不应以 `origin_session_id` 或 `origin_tool_call_id` 作为业务状态来源。

需审批操作的合法写入链是：

```text
用户意图 -> Tool 读取 Owner Context -> 精确 mutation proposal
  -> DSH 原生 Approval -> Owner 鉴权/OCC/幂等/事务
  -> typed receipt -> SessionEvent / UI projection
```

禁止从聊天正文、Session 状态、前端卡片或“最新一条”投影推断业务状态。

## 过期代码结论

`ai-engine-v2/src/runtime` 当前只包含 Python `__pycache__/*.pyc` 产物；活跃源码入口在 `packages/host`、`packages/xiaojian` 和 `packages/legal-*`。旧 Run、WorkUnit、Attempt、pending-card、retry-phase、northbound Run API 等路径不得恢复。

## 技能使用基线

1. 首先读取 `workspace-guidance`，再读取目标服务的 `AGENTS.md`。
2. 单次 DSH/Owner 故障使用 `lawseekdog-ai-run-quality-audit`。
3. 生产模型矩阵使用 `lawseekdog-llm-live-quality-gate`。
4. 发布级全链路验收使用 `lawseekdog-xiaojian-stability-gate`。
5. 文档、模板和 DOCX 质量使用 `templates-service-template-quality`。
6. 通用 DDD、死代码和 Java 风格技能只作为专项辅助，不作为 LawSeekDog 的架构真相源。
7. UI 三个技能只在前端设计、实现后打磨或明确采用 shadcn 方向时加载。
