---
title: 模块
nav_order: 5
has_children: true
---

本节按仓库/服务描述每个模块的职责、边界与关键接口（以当前代码为准）。

建议阅读顺序（从主链路到支撑能力）：

- [consultations-service（咨询会话）](./consultations-service.md)
- [matter-service（事项中心）](./matter-service.md)
- [platform-service（平台配置与控制面）](./platform-service.md)
- [ai-engine（AI 执行引擎）](./ai-engine.md)
- [knowledge-service（知识库）](./knowledge-service.md)
- [memory-service（记忆/事实）](./memory-service.md)
- [files-service（文件与对象存储）](./files-service.md)
- [templates-service（模板/文书）](./templates-service.md)
- [auth-service / user-service / organization-service](./auth-service.md)

工程化：

- [ai-boot-framework（工程脚手架）](./ai-boot-framework.md)
- [infra-templates（复用 CI/CD 工作流）](./infra-templates.md)
- [infra-live（环境 Terraform + 整体发布）](./infra-live.md)

