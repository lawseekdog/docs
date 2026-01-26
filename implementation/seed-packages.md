---
title: Seed Packages
parent: 核心实现
nav_order: 6
---

# Seed Packages（collector-service）

Seed Packages 是 LawSeekDog 的“系统资源分发机制”，用于把：

- Playbooks（流程配置）
- 平台配置（service types、business categories、feature flags 等）
- 知识结构化 seeds（要素/清单/系统 KB 文档）
- 模板系统资源

以可审计、可回放的方式推送到各微服务的 `/internal/**` 接口。

## 1) 资源位置与目录结构

seed packages 存放在：

- `collector-service/resources/seed_packages/`

每个包一个目录：

```
collector-service/resources/seed_packages/<package_id>/
  manifest.json
  data/...
  scripts/...        # 可选
```

示例包：

- `matters_system_resources`：业务分类、Playbook、`matters.service_types` 配置
- `knowledge_structured_seeds`：诉讼要素/非诉清单/系统 KB 文档清单
- `templates_system_resources`：模板系统资源（以包内容为准）

## 2) manifest.json 结构（schema_version=1）

以 `matters_system_resources/manifest.json` 为例（摘录）：

```json
{
  "schema_version": 1,
  "package_id": "matters_system_resources",
  "name": "事项中心系统配置（分类/Playbook/服务类型）",
  "version": "2026-01-09",
  "items": [
    {
      "item_id": "platform_playbook_configs_import",
      "kind": "platform.seed.playbook_configs.import",
      "depends_on": ["platform_business_categories_import"],
      "config": {
        "playbooks_dir": "${PACKAGE_DIR}/data/playbooks"
      }
    }
  ]
}
```

关键字段：

- `package_id`：包唯一标识
- `items[]`：执行项列表（可声明依赖顺序 `depends_on`）
- `kind`：执行器类型（由 collector-service 的 executor 映射到具体动作）
- `config`：执行器所需配置；支持 `${PACKAGE_DIR}` 占位符

## 3) 执行器（kind → internal API 调用）

collector-service 的执行器会把 `kind` 映射为对目标服务的 internal API 调用。

常见映射（示例）：

- `platform.seed.playbook_configs.import`
  - 调用 platform-service：`POST /internal/platform/playbook-configs/batch`
- `platform.seed.system_configs.upsert`
  - 调用 platform-service：写入 config key/value（用于 matters.service_types 等）
- `knowledge.seed.structured.import`
  - 调用 knowledge-service：`POST /internal/seed/structured/import`
- `knowledge.seed.system_kb_documents.import`
  - 调用 knowledge-service：`POST /internal/seed/system-kb-documents/import`

说明：

- manifest 的 `description` 可能仍包含历史实现措辞（例如“写入 Weaviate”），但执行器以 **当前 internal API** 为准。

## 4) collector-service API（管理端/运维）

collector-service 同时挂载在 `/api/v1` 与 `/internal`（方便对齐 Java 服务的协议/网关策略）。

### 4.1 列出与查看 seed packages（internal api key）

- `GET /internal/seed-packages`
- `GET /internal/seed-packages/{package_id}/manifest`
- `GET /internal/seed-packages/{package_id}/download`（下载 zip）

以上接口使用 `X-Internal-Api-Key` 鉴权。

### 4.2 执行 seed packages

两种方式：

1) 需要登录权限（管理端入口）：
   - `POST /api/v1/seed-packages/{package_id}/apply`

2) 运维/CI 用（仅 internal api key）：
   - `POST /internal/seed-packages/apply-internal`

请求体（示例）：

```json
{
  "package_ids": ["matters_system_resources", "knowledge_structured_seeds"],
  "dry_run": false,
  "force": false
}
```

### 4.3 启动时自动执行（env）

collector-service 支持启动时自动执行种子包：

- `SEED_PACKAGE_IDS_ON_STARTUP=...`（逗号分隔）
- 或 `SEED_RESOURCES_ON_STARTUP=true` 使用默认包集合（dev 可容忍失败继续启动）

## 5) seed-init CLI（lawseekdog-seed-init）

`lawseekdog-seed-init` 是一个轻量 CLI，通过调用 collector-service 的 internal seed 接口完成：

- list packages
- view manifest
- apply packages

适合本地开发/运维脚本化。
