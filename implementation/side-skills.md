# 旁路技能与确认型能力（收敛清单）

目标：把“主链路产物技能”与“旁路/确认/确定性护栏能力”区分清楚，便于收敛与维护。

## 旁路能力（Side Skills / System Nodes）

旁路能力一般不直接产出最终结论，而是作为护栏/预处理/门禁在特定条件下插入执行。

### file-classify（文件分类）

- 用途：对新附件做类型识别/归类，写入 `data.files.file_classifications`
- 触发：通常由 `materials` 子图优先触发（本轮有新附件/材料指纹变化）
- 收敛建议：保留为旁路能力；不要把分类逻辑掺进各 intake 技能，避免重复与漂移

### human_review（统一中断点）

- 用途：承接所有 `ask_user` 卡片中断（统一的人机协同入口）
- 触发：任何子图/skill 需要补问/选择/确认时
- 关键约束：进入中断前必须先 `sync_data`（保证断线可恢复）

## 确认型能力（Confirm/Selection）

这类能力的核心价值是：把关键决策落到可审计字段里，并通过卡片让用户确认，避免模型“自作主张”推进。

### 文书选择与复核（docgen 子图）

- 用途：
  - 基于 `data.work_product.recommended_documents` 推荐交付件
  - 记录用户/律师确认到 `profile.decisions.selected_documents`、`profile.decisions.document_reviewed` 等字段
- 触发：当输出阶段产生推荐文书，且需要生成/复核时
- 收敛建议：保留在 `docgen` 子图中作为统一能力（而不是散落在各业务子图/技能里）

### policy_gate（交付门禁）

- 用途：对“可交付/需复核/需修改”做统一门禁与确认（高风险输出必须拦住）
- 触发：在各 goal 子图产出后、交付前

## 未被主链路直接引用的技能（仍可能保留）

以下技能可能用于：API 强制调用（force_skill）、后台任务或后续扩展：

- memory-extraction：对话记忆提取（常作为后台/旁路能力）
- knowledge-ingest：知识库入库增强（internal）
- document-editing：文书修改（用户驱动触发，不一定走主链路）
- sample-structure-parse：范文结构解析（internal；用于把 DOCX 范文萃取为可复用模板结构）
