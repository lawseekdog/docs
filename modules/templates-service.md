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
- 文档编辑能力（/api/v1/document-editor/**，本地 Apache POI 实现；AI 修改建议由 ai-engine skill 产出）
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

## 文档编辑器（重要）

`DocumentEditorController` 对前端暴露：`/api/v1/document-editor/**`

当前实现为“本地编辑 + AI 建议”：

- docx 的解析/替换/插入/撤销/保存：templates-service 本地 Apache POI + files-service（不再依赖 ai-engine 的内部 doc-editor）
- AI 修改建议：`/api/v1/document-editor/{fileId}/ai-modify` 调用 ai-engine 的 `document-editing` 技能，仅返回修改建议（changes），具体写入由 `apply-changes/save` 完成

## 对内 API（/internal）

seed 导入：

- `POST /api/v1/internal/seed/curated-templates/import`

由 collector-service 的 seed 包触发。

## 与其它服务的关系

- ai-engine：文档编辑代理与文书生成技能交互
- files-service：模板素材与生成文档的 fileId 归档
- matter-service：在事项流程中选择/生成文书
