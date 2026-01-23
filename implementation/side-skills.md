# 旁路技能与确认型技能（收敛清单）

目标：把“流程推进类主技能”与“旁路/确认/确定性护栏技能”区分清楚，便于收敛与维护。

## 旁路技能（Side Skills）

旁路技能一般不负责推进主流程产出结论，而是作为护栏/预处理/导购在特定条件下插入执行（常见来源：`playbook.priority_rules` 或咨询态循环阶段）。

### file-classify（文件分类）
- 用途：对新附件做类型识别/归类，写入 `data.files.file_classifications`
- 触发：`priority_rules` 命中（例如本轮有新附件且未分类）
- 是否调用 LLM：是（preprocess 只准备输入，分类由 LLM 完成）
- 收敛建议：保留为旁路技能；不要把分类逻辑掺进各 intake 技能，避免重复与漂移

### system_action:kickoff（事项启动统一入口，系统节点）
- 用途：统一开场白 + 通过卡片收集 `profile.facts` + 上传材料 `attachment_file_ids`
- 触发：各 playbook 首阶段（`phase.system_action = "kickoff"`，由 `system_phase` 节点发卡并 interrupt）
- 是否调用 LLM：否（确定性 UI 步骤，不走 skill registry）
- 收敛建议：保留为系统节点；不要放在 skills 目录，避免 registry 膨胀与脚本/校验成本

### due-diligence-materials-collect（尽调材料收集）
- 用途：无材料时强制上传；有材料则直接继续
- 触发：尽调链路的材料阶段
- 是否调用 LLM：否（全在 preprocess 内确定性产出）
- 收敛建议：保留；避免“分析阶段才发现没材料”导致流程卡死/反复追问

### readiness-assessment（咨询态就绪度评估）
- 用途：咨询态对“是否建议建档/进入正式事项”做导购式推荐，输出 `data.recommendation_card`
- 触发：咨询态 playbook 循环阶段
- 是否调用 LLM：当前实现为“无信号/已建档时跳过，否则可能调用 LLM”
- 收敛建议（可选）：
  1) 保留现状：LLM 评估更灵活
  2) 进一步收敛为确定性（推荐走关键词/规则评分）：降低成本与延迟，但需要接受“推荐精度”下降

## 确认型技能（Confirm/Selection Skills）

这类技能的核心价值是：把关键决策落到可审计的字段里，并通过卡片让用户确认，避免模型“自作主张”推进。

### documents（文书推荐与选择）
- 用途：从 `playbook.document_pool` 选择 1-5 个文书，写入 `profile.decisions.selected_documents`
- 是否调用 LLM：是（但若已选过会在 preprocess 直接跳过）
- 收敛建议：保留；它把“生成哪些文书”显式化，便于回溯与复用

### work-plan（阶段计划生成与确认）
- 用途：把策略落为可执行计划 `profile.work_plan`（draft/confirmed），可由系统后处理发卡确认
- 是否调用 LLM：是（但 confirmed 时会跳过）
- 收敛建议：保留；计划属于高风险输出，建议保留“确认”这一层

### knowledge-deposit（知识沉淀与入库）
- 用途：抽取 1-8 条候选知识并让用户选择是否入库（或跳过），写入 data/profile 相应字段
- 是否调用 LLM：候选生成阶段会调用 LLM；用户选择后入库在 preprocess 中确定性执行
- 收敛建议：保留；它把“可复用经验”沉淀为显式资产，并有明确的隐私化约束

## 未被 playbook 直接引用的技能（仍可能保留）

以下技能在 seed playbooks 中未直接出现，但可能用于：API 强制调用（force_skill）、内部流程、后台任务或后续扩展：
- memory-extraction：对话记忆提取（常作为后台/旁路能力）
- knowledge-ingest：知识库入库增强（internal）
- document-editing：文书修改（用户驱动触发，不一定走 playbook）
- strategy-planning：进攻策略规划（当前 playbook 更偏向 dispute-strategy-planning/defense-planning，可按业务再决定是否合并/弃用）
