---
title: 项目总览与文档导航
parent: 架构
nav_order: 1
---

# 项目总览与文档导航

LawSeekDog 是面向律师工作的多仓库平台。前端使用 React、TypeScript 和 Vite；官方 DSH 承载 Agent、Session、Tool、Question、Approval、Workflow 和 Subagent；法律插件经唯一 Owner API 保存业务结果。`ai-engine-v2` 是 DSH production profile、Host、Xiaojian 和插件工作区，不是另一套 Agent Runtime。

## 按问题定位

| 问题 | 维护来源 |
| --- | --- |
| 谁拥有业务事实，入口如何初始化，什么保存需要批准 | [架构边界](../LAWSEEKDOG-ARCHITECTURE-BOUNDARIES.md)及其引用的共享规范 |
| 当前主链路与服务关系 | [架构概览](overview.md)、[拓扑](microservices.md) |
| Session 准备稿、Owner 保存及历史版本如何衔接 | [数据流](data-flow.md) |
| 仓库分组与构建/部署入口 | [仓库地图](repositories.md) |
| 到底部署、验收到了哪一步 | [进度核验方法](progress.md)，再读该次发布真实证据 |
| 任务该读哪个规范或技能 | `lawseekdog-agent-skills/workspace-guidance/TASK-ROUTING.md` |

共享硬边界仅在 `LAWSEEKDOG-WORKSPACE-GUIDANCE.md` 维护；本文档站描述职责和源码导航，技能只提供具体操作流程与证据要求，服务 `AGENTS.md` 保留本地责任和最小检查。法律插件目录由共享规范维护，不在多页复制清单。开发代理不固定某个模型；产品 Provider/Model Settings 只由 DSH 管理。

发生冲突时遵循用户明确指令、Owner 契约与代码、共享硬约束、服务本地约束的顺序；先报告并修正唯一维护来源，不能用新兼容分支掩盖。架构说明不证明部署或业务验收成功。
