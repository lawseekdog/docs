---
title: 仓库与维护职责
parent: 架构
nav_order: 5
---

# 仓库与维护职责

| 分组 | 仓库 | 维护内容 |
| --- | --- | --- |
| 用户界面 | `frontend` | React/TypeScript 律师、律所、管理端及 Xiaojian 嵌入 |
| Agent 与专业能力 | `ai-engine-v2` | 官方 DSH profile、Host、Xiaojian、法律与社区插件、契约/eval |
| 业务 Owner | `firm-service`、`case-service`、`matter-service`、`document-workspace-service` | 委托、案件程序、事项成果任务、文书生命周期 |
| 来源与渲染 | `files-service`、`templates-service`、`platform-service`、`knowledge-service`、`collector-service` | 文件版本、模板渲染、目录与规则、知识素材与采集 |
| 身份与支持域 | `user-service`、`lawyer-profile-service`、`auth-service`、`billing-service`、`notification-service` | 各自领域，以服务契约为准 |
| 工程 | `ai-boot-framework`、`infra-templates`、`infra-live` | Java 构建基线、CI 资源、拓扑与发布 |
| 文档与开发技能 | `docs`、`lawseekdog-agent-skills`、`lawseekdog-codex-plugins` | 项目事实、共享规范与技能、开发插件 |

这是维护导航，不是启用服务清单。实际部署对象由 infra topology 和该次 release scope 决定；目录存在不能证明活跃。`organization-service`、`assistant-service`、`memory-service`、`shared-libs` 不应恢复为当前依赖。

各仓库独立 Git、独立提交与验证。Java Owner 使用仓库锁定的 `ai-boot-framework` 源码提交本地安装；API 消费者使用各自消费锁与生成物。模型配置不复制到服务指引。发布通过 `infra-live/scripts/release.py` 当前 CLI，具体凭据与参数查运行规范，不在本文固定。
