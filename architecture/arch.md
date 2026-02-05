---
title: Legal Agent 图谱参考（草案）
parent: 架构
nav_order: 99
---

# Legal Agent 图谱参考（草案）

> 本页为设计参考，不代表当前仓库实现已经完整落地；请以各仓库代码与 `architecture/progress.md` 为准。

**法律业务（合同生成/合同审查/案例分析/法律文书生成）**的推荐 AI 架构，并把 **4 条业务链全部展开**成可落地的 LangGraph 图（每条都是“完整链路 + 循环 + 门禁 + 可审计证据”）。

我推荐的是：**Workflow 主导 + 1 个 Legal Agent（负责取材/生成）+ 确定性校验子图 + Policy Gate +（可选）Reviewer sub-agent**。  
（V1 就把 Reviewer 节点去掉；V2 再加 Reviewer。）

---

## 0) 推荐的整体架构（组件分层）
- **LangGraph 编排层**：Router → 任务子图（draft/review/memo/doc）→ 统一 Verification 子图 → Policy Gate → 打包交付
- **Legal Agent**：只在“取材/写作/改写”节点使用；其余尽量确定性节点
- **Tools/Skills 白名单**（举例）  
  - 取材：法规检索、判例检索、内部 KB/RAG、模板库、条款库  
  - 解析：OCR/版面解析、条款切分/编号、实体抽取（主体/金额/日期/法域）  
  - 校验：引用可达性、事实一致性、红线/必备条款、编号/格式、术语一致性  
  - 交付：docx/pdf 导出、红线 diff、引用附件打包、审计日志落库
- **Policy Gate（必备）**：决定“可交付/需律师复核/需审批/拒绝”；高风险输出必须拦住

---

## 1) 顶层 Router 图（总入口）
```mermaid
flowchart TD
  S([START]) --> INTAKE["Intake<br/>任务类型/法域/语言/输入材料/交付目标"]
  INTAKE --> NORM["Normalize<br/>OCR/解析/结构化事实/去噪"]
  NORM --> ROUTE{"Router<br/>选择任务子图"}

  ROUTE --> CD_SG["Subgraph: Contract Draft<br/>合同生成"]
  ROUTE --> CR_SG["Subgraph: Contract Review<br/>合同审查/修订"]
  ROUTE --> CM_SG["Subgraph: Case Memo<br/>案例分析/备忘录"]
  ROUTE --> LD_SG["Subgraph: Legal Doc Draft<br/>诉状/律师函/法律意见书"]

  CD_SG --> VERIFY_SG
  CR_SG --> VERIFY_SG
  CM_SG --> VERIFY_SG
  LD_SG --> VERIFY_SG

  VERIFY_SG["Subgraph: Verification<br/>引用/一致性/红线/风险评分"] --> GATE{"Policy Gate<br/>可交付? 需律师复核? 需审批?"}
  GATE -->|需复核/审批| HUMAN["Human Review<br/>律师/法务确认"]
  HUMAN --> PACK["Package & Deliver<br/>docx/pdf + 引用附件 + 风险报告 + 审计日志"]
  GATE -->|可交付| PACK
  PACK --> E([END])
```

---

## 2) 子图一：合同生成（Contract Draft）— 全展开
```mermaid
flowchart TD
  CD_START([START]) --> CD_PREP["准备输入<br/>合同类型/法域/当事人/交易结构/目标条款"]
  CD_PREP --> CD_MISS{"关键信息缺失?"}

  CD_MISS -->|是| CD_ASK["Ask User<br/>输出字段清单/问题清单/表单"] --> CD_PREP
  CD_MISS -->|否| CD_BASE["加载基线<br/>模板选择规则 + 红线/必备条款 + 审批规则"]

  CD_BASE --> CD_PLAN["生成起草大纲<br/>章节/条款清单/变量表"]
  CD_PLAN --> CD_AGENT(("Legal Agent<br/>取材 + 起草<br/>在工具白名单内选用"))

  CD_AGENT -->|tool| CD_TPL["Tool: 模板/范本库<br/>选择/拉取/对齐结构"]
  CD_AGENT -->|tool| CD_CLAUSE["Tool: 条款库<br/>检索/生成/替换"]
  CD_AGENT -->|tool| CD_LAW["Tool: 法规/司法解释检索<br/>用于引用/合规依据"]
  CD_AGENT -->|tool| CD_CASE["Tool: 判例/案例检索<br/>用于风险论证(可选)"]
  CD_AGENT -->|tool| CD_KB["Tool: 内部KB/RAG<br/>公司政策/历史合同/谈判策略"]

  CD_TPL --> CD_AGENT
  CD_CLAUSE --> CD_AGENT
  CD_LAW --> CD_AGENT
  CD_CASE --> CD_AGENT
  CD_KB --> CD_AGENT

  CD_AGENT --> CD_DRAFT["草案 v1<br/>按章节输出 + 引用清单 + 风险点(初版)"]
  CD_DRAFT --> CD_QC1["确定性检查<br/>编号/格式/必填占位符是否都填了"]

  CD_QC1 --> CD_FIX1{"占位符/缺项/格式问题?"}
  CD_FIX1 -->|是| CD_AGENT
  CD_FIX1 -->|否| CD_OUT["输出：合同草案候选<br/>+ 引用清单 + 风险摘要"] --> CD_END([END])
```

> 合同生成里，“AI 判断用什么工具”主要发生在 `Legal Agent` 节点：  
> 但它的“可选空间”被 **模板/红线/必备条款/缺项检测**这些确定性节点强约束住了。

---

## 3) 子图二：合同审查/修订（Contract Review）— 全展开
目标交付一般是：**风险矩阵 + 条款级修改建议 + 红线版合同 + 引用依据/内部规则依据**。

```mermaid
flowchart TD
  CR_START([START]) --> CR_INGEST["Ingest<br/>上传合同(docx/pdf) -> 解析文本"]
  CR_INGEST --> CR_STRUCT["结构化<br/>章节/条款切分 + 编号对齐 + 元数据(法域/主体/金额/期限)"]
  CR_STRUCT --> CR_BASE["加载基线<br/>标准模板/条款库/红线规则集/审批规则"]

  CR_BASE --> CR_GAP["Gap Analysis<br/>缺失条款/异常条款/偏离标准/冲突定义"]
  CR_GAP --> CR_PRI["优先级排序<br/>高风险优先(责任/违约/付款/保密/管辖/数据)"]

  CR_PRI --> CR_NEED_EVID{"是否需要补外部依据?<br/>(存在高争议/需引用支撑)"}
  CR_NEED_EVID -->|是| CR_AGENT(("Legal Agent<br/>证据补强 + 生成修改理由"))
  CR_NEED_EVID -->|否| CR_EDIT

  CR_AGENT -->|tool| CR_LAW["Tool: 法规/司法解释检索"]
  CR_AGENT -->|tool| CR_CASE["Tool: 判例/案例检索"]
  CR_AGENT -->|tool| CR_KB["Tool: 内部KB/RAG<br/>公司政策/历史谈判结论"]
  CR_LAW --> CR_AGENT
  CR_CASE --> CR_AGENT
  CR_KB --> CR_AGENT
  CR_AGENT --> CR_EDIT["生成修改建议<br/>条款级: 风险 -> 建议文本 -> 理由 -> 引用/内部规则"]

  CR_EDIT --> CR_REWRITE["生成红线版本<br/>替换/新增/删除条款(条款级)"]
  CR_REWRITE --> CR_DIFF["Tool: Redline/Diff<br/>输出对比稿 + 变更摘要"]

  CR_DIFF --> CR_OUT["输出：审查包候选<br/>风险矩阵 + 红线稿 + 变更摘要 + 引用清单"]
  CR_OUT --> CR_END([END])
```

---

## 4) 子图三：案例分析/备忘录（Case Memo）— 全展开
目标交付一般是：**争点清单 + 时间线 + 规则/判例依据 + 适用分析(IRAC) + 不确定点/需补事实清单 + 建议路径**。

```mermaid
flowchart TD
  CM_START([START]) --> CM_INGEST["Ingest<br/>事实材料/证据/沟通记录/对方主张"]
  CM_INGEST --> CM_EXTRACT["结构化事实<br/>主体/关系/金额/日期/行为/证据索引"]
  CM_EXTRACT --> CM_TL["构建时间线<br/>事件序列 + 证据编号映射"]

  CM_TL --> CM_ISSUE["Issue Spotting<br/>争点清单 + 需补事实/需补证据"]
  CM_ISSUE --> CM_MISS{"关键信息缺失?"}

  CM_MISS -->|是| CM_ASK["Ask User<br/>补充问题清单/证据清单"] --> CM_INGEST
  CM_MISS -->|否| CM_RETRIEVE["检索阶段<br/>法规/判例/类案/内部指引"]

  CM_RETRIEVE --> CM_AGENT(("Legal Agent<br/>组织分析框架 + 引用支撑"))
  CM_AGENT -->|tool| CM_LAW["Tool: 法规/司法解释检索"]
  CM_AGENT -->|tool| CM_CASE["Tool: 判例/案例检索"]
  CM_AGENT -->|tool| CM_SIM["Tool: 类案检索/相似度检索(可选)"]
  CM_AGENT -->|tool| CM_KB["Tool: 内部KB/RAG<br/>公司政策/过往处理"]
  CM_LAW --> CM_AGENT
  CM_CASE --> CM_AGENT
  CM_SIM --> CM_AGENT
  CM_KB --> CM_AGENT

  CM_AGENT --> CM_IRAC["生成 IRAC 结构<br/>每争点: 规则 -> 适用 -> 结论(暂定) + 不确定性"]
  CM_IRAC --> CM_MEMO["输出 Memo v1<br/>摘要/争点/分析/风险/建议/引用清单"]
  CM_MEMO --> CM_END([END])
```

---

## 5) 子图四：法律文书生成（诉状/答辩状/律师函/法律意见书）— 全展开
目标交付一般是：**结构化文书 + 事实与证据目录 + 管辖/时效/主体资格提示 + 引用/模板来源 + 可交付格式**。

```mermaid
flowchart TD
  LD_START([START]) --> LD_TYPE["选择文书类型<br/>起诉状/答辩状/律师函/法律意见书"]
  LD_TYPE --> LD_SCHEMA["加载模板Schema<br/>必填字段/附件目录/格式要求"]
  LD_SCHEMA --> LD_MISS{"必填字段缺失?"}

  LD_MISS -->|是| LD_ASK["Ask User<br/>生成表单字段/示例填写"] --> LD_SCHEMA
  LD_MISS -->|否| LD_RULES["规则提示<br/>管辖/时效/主体资格/送达要点(提示性)"]

  LD_RULES --> LD_AGENT(("Legal Agent<br/>按模板起草 + 引用绑定"))
  LD_AGENT -->|tool| LD_TPL["Tool: 文书模板库"]
  LD_AGENT -->|tool| LD_LAW["Tool: 程序/实体规则检索<br/>(按法域/类型)"]
  LD_AGENT -->|tool| LD_KB["Tool: 内部格式/口径/历史文书"]
  LD_TPL --> LD_AGENT
  LD_LAW --> LD_AGENT
  LD_KB --> LD_AGENT

  LD_AGENT --> LD_EVID["生成证据目录<br/>附件编号/对应事实点"]
  LD_EVID --> LD_DOC["生成文书 v1<br/>结构化段落 + 引用清单 + 风险提示"]
  LD_DOC --> LD_OUT["输出：文书候选<br/>docx/markdown + 附件目录 + 引用清单"]
  LD_OUT --> LD_END([END])
```

---

## 6) 统一的 Verification（确定性校验）子图（建议所有任务共用）
你可以把下面这个作为一个真正的 LangGraph subgraph，在 4 个业务子图后统一调用：

```mermaid
flowchart TD
  V_START([START]) --> V_CITE["引用可达性校验<br/>引用必须能映射到 evidence.excerpt/source_id"]
  V_CITE --> V_CONS["一致性校验<br/>主体/金额/日期/术语/定义/编号"]
  V_CONS --> V_PLAY["红线/必备条款/合规规则检查<br/>触发: 必须复核/必须审批/禁止交付"]
  V_PLAY --> V_SCORE["风险评分/置信度封顶<br/>证据覆盖度 + 规则结果"]
  V_SCORE --> V_END([END])
```

---

## 7) 你最终“推荐落地”的版本（一句话）
先用：**V1 + 统一 Verification + Policy Gate** 把链路跑通；  
法律业务我建议你直接上 **V2（加一个 Reviewer 节点/子 agent）**做质量保险，但不要一开始就拆 Researcher/Drafter 多 agent。

如果你愿意，我下一步可以把这套“全部展开”的图，进一步落成：  
- 每个节点的 `State` 输入输出 schema（Pydantic）  
- tools/skills 的接口定义（含引用证据结构、hash、可回放）  
- Policy Gate 的规则样例（哪些情况必须律师复核/必须审批/禁止交付）
