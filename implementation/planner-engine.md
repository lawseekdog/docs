---
title: Planner 决策引擎
parent: 核心实现
nav_order: 2
---

# Planner 决策引擎（ai-engine）

本页描述 `ai-engine` 如何在每一轮对话/事项推进中选择“下一步做什么”（call skill / chat respond / phase replan / finish）。

核心目标：让流程决策可解释、可配置、可回归，避免“纯 LLM 选技”导致不可控。

## 1) 输入：state + playbook + phase

Planner 的最小输入：

- `state`：统一状态（thread state），包含 `profile`、data 叶子字段、附件/文件信息、当前 phase 等
- `playbook`：流程配置（allowed_skills、priority_rules、phases、gate/checkpoints 等）
- `phase`：当前阶段配置（从 playbook.phases 中按 `current_task_id` 选取）

这些数据通常由上游（consultations-service/matter-service）组装后传入 ai-engine。

## 2) 可用技能集合（强收敛）

ai-engine 在选技前会先计算“当前阶段可执行技能列表”（available_skills），过滤规则：

1. 必须显式配置 `allowed_skills`
   - phase.allowed_skills 与 playbook.allowed_skills 至少要有一个
   - 不允许兜底为“所有技能都可用”（避免 planner 行为不可控）
2. allowed_skills 中引用的技能必须存在于 registry
   - 缺失技能会直接报错（禁止静默忽略）
3. 技能必须满足 `skill.meta.requires`（all/any 条件）
4. 排除 internal / api_call_only 技能

> 代码位置：`ai-engine/src/application/skill_executor/planner.py`

## 3) 决策策略链（Chain of Responsibility）

Planner 采用“策略链”从高优先级到低优先级依次尝试，首个命中即返回。

默认顺序（以代码为准）：

1. `force_skill`
2. `priority_rules`
3. `query_mode`
4. `phase_complete`
5. `deterministic`
6. `llm_planner`
7. `no_available_skills`

> 代码位置：`ai-engine/src/application/agent/planner/chain.py`

### 3.1 force_skill：强制指定技能

用于“外部 API 直接调用某个技能”或调试场景：

- 上游在 context/state 中写入 `force_skill=<skill>`
- Planner 直接选择该技能（仍受 allowed_skills/requires 等约束）

典型场景：

- `/internal/ai/skills/{skill}/execute`（技能直跑）

### 3.2 priority_rules：配置优先规则（确定性）

Playbook 支持两级 priority_rules：

- playbook.priority_rules（全局）
- phase.priority_rules（阶段内）

命中时强制选择对应技能。

重要约束：

- 若规则命中但技能不可执行，会直接抛错（提示检查 allowed_skills 与 requires），避免“规则形同虚设”。

> 代码位置：`ai-engine/src/application/agent/planner/strategies/priority_rules.py`

### 3.3 query_mode：旁路问答模式

当 state 处于 `_query_mode` 时：

- 仅允许从 playbook.allowed_skills（全局技能白名单）中选择“检索/问答类技能”
- 避免问答场景误触发阶段推进与门控（workflow side effects）

> 代码位置：`ai-engine/src/application/agent/planner/strategies/query_mode.py`

### 3.4 phase_complete：阶段完成 → replan 到下一阶段

当 PhaseManager 判定当前 phase 已完成时：

- 若仍有下一阶段：`action=replan`，并将 `current_task_id` 切到 next phase
- 若无下一阶段：`action=finish`

> 代码位置：`ai-engine/src/application/agent/planner/strategies/phase_complete.py`

### 3.5 deterministic：尽量不用 LLM 的“缺口覆盖”选技

当可用技能 > 1 时：

- 根据当前阶段缺口（missing_goals）与技能 outputs.provides（profile + data fields）尝试选择唯一能覆盖缺口的技能
- 若存在唯一最优技能：直接执行
- 否则交给 LLM 兜底

> 代码位置：`ai-engine/src/application/agent/planner/strategies/deterministic.py`

### 3.6 llm_planner：LLM 兜底选技

当确定性策略都无法给出唯一答案时：

- LLM 在 allowed_skills 约束内选择下一技能
- 必须返回可解释原因（reason），用于 tracing 与回放

> 该策略保证“可用性”，但不应成为唯一手段；priority_rules 与 deterministic 才是主路径。

### 3.7 no_available_skills：兜底

当无可执行技能时：

- 若仍可 chat respond：输出自然语言回复
- 或直接 finish

## 4) PhaseManager：门控与缺口（gate/checkpoints）

PhaseManager 负责：

- 校验 playbook 配置（禁止写法漂移导致卡死）
- 判断 phase 是否完成
- 计算 missing_goals（供 deterministic 策略使用）

关键约束（来自校验逻辑）：

- gate_field/checkpoints 是点路径（不允许写 `state.` 前缀）
- 禁止 `data.<group>.<field>`：data 侧必须直接写叶子字段名
- profile/decisions 字段必须在允许列表内（避免随意造字段）
- 写了 gate_field 必须配 gate_value 或 gate_check（禁止隐式兜底）

> 代码位置：`ai-engine/src/application/skill_executor/phase_manager.py`

## 5) 可回归与可观测（建议）

为了让 Planner 行为“可验证”：

- playbook 变更必须通过静态校验（PhaseManager 已做强收敛）
- skill 输出字段必须通过 schema + validate（见 `implementation/skill-system.md`）
- 关键链路需记录 trace（ai-engine 内置 execution trace；consultations-service 会转发 progress/task_start/task_end 等事件）
