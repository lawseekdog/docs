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

Matter 的生命周期开放状态为 `draft -> provisional -> active`，终态包括 `completed`、`cancelled`、`declined`、`superseded`。Task、Document、Contract Review、Case Stage 和 DSH Session 是独立状态轴，不能互相推断或合并。

## 业务入口

案件新业务从“新建案件”入口进入：显式开始后先建立 DSH Session、上传材料并讨论分析；精确确认前不创建 Case 或 Matter。建案及对应事项、成果保存经完整影响确认后执行，关联并继续原会话。已有案件中的继续分析是续办，不另设独立案件分析入口。合同审查、文书起草遵循相同开始/讨论/确认/保存边界。

讨论阶段的工作记录就是 Session，不新增工作记录表。Matter 表达律师真实工作范围，文书只是交付物；已有案件或合同事项中的文书不因插件或文书数量另建事项。`legal-matter` 是内部工具能力，不是新的页面入口。客户、Case、Engagement 依据实际关系关联；独立工作并非一律先建立委托。文种不等于代理处分，DWS 定稿不等于签章或对外提交。

- 律所工作台：`frontend/src/apps/firm/firmRouter.tsx`，入口为 intake、engagement、documents、work、team 等经营路径。
- 管理端：`frontend/src/apps/admin/adminRouter.tsx`，只负责平台管理和 DSH 观测，不承载律师业务写入。
- DSH 业务入口：`ai-engine-v2/packages/legal-*` 的 typed Tool。自然语言请求先读取 Owner Context；只读请求直接返回 typed result；写请求先形成完整影响并走官方 Approval，再调用 Owner。
- Matter 内部 API：`matter-service/src/main/java/com/lawseekdog/matter/api/controller`，只接受内部服务或律师端授权请求，不接受前端自行构造的业务 mutation。

## Session 与业务边界

Session 可以保存 `business reference`、当前 focus、来源材料和导航信息，但这些只是当前会话的引用选择。Matter、Case、Engagement、Document 和 Task 的长期关系与状态必须保存在各自 Owner 数据库中。Owner 不应以 `origin_session_id` 或 `origin_tool_call_id` 作为业务状态来源。

合法写入链是：

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
