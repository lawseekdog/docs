# 2026-09-08 工作检查点（WIP，未部署）

用户要求先全仓提交并 push，停止等待修复与测试。本次不是发布版，也未通过整体业务验收。

## 已确认交互

案件新业务只有现有新建案件入口。显式开始先创建 DSH Session，保存消息、材料引用和讨论成果；这就是工作记录，不提前创建 Matter，不新增工作记录表。讨论分析后展示建案、实际所需事项及成果保存的完整影响，用户确认后写各 Owner，关联并继续原 Session。合同审查、文书起草沿用同一开始/讨论/确认/保存边界；文书是成果，Matter 是实际律师工作范围。legal-matter 是新获批的内部事项插件，不是新页面入口；任务与接案插件保留。

## 提交范围

ai-engine-v2、matter-service、document-workspace-service、firm-service、platform-service、frontend、lawseekdog-agent-skills、docs。其余仓库无代码改动。私有配置、凭据、日志、构建产物不提交。

## 必须继续处理

- AI：默认建案接分析准备稿的代码未接完注册参数/schema和完整审批链；Case 当前 3 项 TypeScript 编译错误（preparation-source assertion 类型，tools 中 programCode/准备稿字段不匹配）。Document 328 项测试中 6 项尚待更新与验证。legal-matter 关联/生命周期、Opinion 可空客户与剩余旧提示尚未完成。
- Matter/Firm/Platform：最近全量 Maven 通过，但最后新增委托到案件关联 PG 测试未执行。M7 lineage 仍须显式 source_matter_id，不能按 Case/Proceeding 唯一或最新猜选。
- DWS：历史 publication v1 只读 schema 已添加部分定义，snapshot 引用和公共制品生成尚未收尾；新写应只接受 v2，不补写历史 purpose/hash。旧 v8 草稿当前发布要求重新授权，不声称可直接发布。
- Front：DWS 历史 v1 消费和生成类型同步未完成，Matter vendor 契约未同步；此前单测 1142/1151 通过，8 个失败点已调整未重跑，1 个旧样式断言未处理。build:lawyer 未跑。
- AI root/profile/catalog 的 legal-matter 首两个 Tool 已接入，peer check、dump-config、36 项能力入口回归通过；新加工具/事件须继续同步，锁文件需与最后 Case 依赖修改再次核验。
- 共享指引核心检查和 3 项 checker 测试通过；全仓 AGENTS 引用检查仍报告 assistant-service、docs、gateway-service、organization-service 的既有缺口。
- 尚未 push 后验证、部署或真实 Edge 默认业务链验收。不要用旧环境或单项测试替代当前整体验收。

## 下一步

拉取上述各仓 main 后先核对 WIP 编译与依赖锁，完成默认建案一次完整影响确认及原准备成果保存，补全消费者契约，再验证和选服务发布。不得回退用户已合并的 Responses 返回格式及 opaque native tool-call identity 修改。不得重开会话或重做分析来掩盖关联缺失。
