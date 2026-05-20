---
title: Knowledge Seed Packages
parent: 核心实现
nav_order: 6
---

# Knowledge Seed Packages（knowledge-service）

Knowledge Seed Packages 是 LawSeekDog 知识服务的系统知识初始化机制，用于把：

- 知识结构化 seeds（要素/清单/系统 KB 文档）
- 系统知识库资源（法条/案例/网页资料/规范等）

以可审计、可回放的方式写入 `knowledge-service` 当前 canonical corpus。

## 1) 资源位置与目录结构

seed packages 存放在：

- `knowledge-service/resources/seed_packages/`

每个包一个目录：

```
knowledge-service/resources/seed_packages/<package_id>/
  manifest.json
  data/...
```

示例包：

- `knowledge_system_resources`：系统知识库基础资源（法条/案例/网页资料/规范）

## 2) manifest.json 结构

以 `knowledge_system_resources/manifest.json` 为准，manifest 描述包标识、版本、数据文件与入库约束。执行逻辑不由 `collector-service` 维护。

## 3) 执行入口

当前唯一运维入口在 knowledge-service 内：

- `knowledge-service/tools/knowledge_seed/apply_seed_package.py`
- `knowledge-service/tools/knowledge_seed/audit_seed_package.py`
- `knowledge-service/tools/knowledge_seed/smoke_search_seed.py`

这些工具直接围绕 knowledge-service 的种子包、入库与检索质量工作；不通过独立初始化仓库，也不通过 collector-service 的 seed API。
