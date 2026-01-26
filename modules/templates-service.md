---
title: templates-service（模板/文书）
parent: 模块
nav_order: 8
---

# templates-service（模板/文书）

## 定位

templates-service 提供模板与文书相关能力：

- 模板 CRUD / 版本管理
- 文书生成任务（生成/交付物）
- 文档编辑能力的对外代理（当前转发到 ai-engine 的 python-docx 编辑器）
- seed 导入（由 collector-service 分发）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（模板/版本/任务等元数据）

## 对外 API（/api/v1，摘要）

按 controller 命名大致包含：

- `TemplateController` / `TemplateVersionController`：模板与版本
- `TemplateTaskController`：生成任务/交付物
- `SampleDocumentController`：示例文档
- `DocumentEditorController`：文档编辑器代理（见下文）

## 文档编辑器代理（重要）

当前 `DocumentEditorController` 的实现是“代理模式”：

- 对前端暴露：`/api/v1/document-editor/**`
- 对内转发：`ai-engine` 的 `/internal/document-editor/**`
- 通过 `X-Internal-Api-Key` 鉴权（由配置 `ai.boot.client.internal-api-key` 注入）

这样做的目的：

- 复用 Python 侧 `python-docx` 的段落编辑能力
- 对前端保持统一的 ApiResponse 协议与鉴权入口

## 对内 API（/internal）

seed 导入：

- `POST /internal/seed/curated-templates/import`

由 collector-service 的 seed 包触发。

## 与其它服务的关系

- ai-engine：文档编辑代理与文书生成技能交互
- files-service：模板素材与生成文档的 fileId 归档
- matter-service：在事项流程中选择/生成文书
