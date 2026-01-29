---
title: files-service（文件与对象存储）
parent: 模块
nav_order: 7
---

# files-service（文件与对象存储）

## 定位

files-service 负责文件相关能力：

- 文件上传/下载（对外）
- 预签名 URL（对外/对内）
- 文件元数据存储（fileId/owner/purpose/contentType/size/parseStatus 等）
- 文件解析（internal parse，供 skills 调用）
- 对象存储适配（MinIO/S3；实现以仓库为准）

## 技术栈

- Java 21 + Spring Boot
- Postgres + Flyway（元数据）
- MinIO/S3（对象存储，按环境配置）

## 对外 API（/api/v1，摘录）

- `POST /api/v1/files/upload`（multipart）
- `GET  /api/v1/files/{fileId}`（元信息）
- `GET  /api/v1/files/user/{userId}`（用户文件列表）
- `GET  /api/v1/files/{fileId}/download-url`（预签名下载）
- `GET  /api/v1/files/{fileId}/download`（直下）
- `POST /api/v1/files/upload-url`（预签名上传）
- `DELETE /api/v1/files/{fileId}`（删除）

鉴权/权限：

- 需要 `X-User-Id`
- 支持 superuser（`X-Is-Superuser`）做越权访问控制（具体逻辑以实现为准）

## 对内 API（/internal，摘录）

供 ai-engine/服务间调用：

- `POST /api/v1/internal/files/upload`（internal 上传）
- `POST /api/v1/internal/files/presigned-url`
- `GET  /api/v1/internal/files/{fileId}`（内部元信息）
- `GET  /api/v1/internal/files/{fileId}/content`（内部内容）
- `POST /api/v1/internal/files/{fileId}/parse`（解析）
- `PUT  /api/v1/internal/files/{fileId}/parse-status`
- `GET  /api/v1/internal/files/{fileId}/exists`
- `DELETE /api/v1/internal/files/{fileId}`

## 与其它服务的关系

- ai-engine：通过 `file__get_info`/`file__parse` 等工具读取材料
- matter-service：文件与事项关联（权限校验/访问控制）
- knowledge-service/templates-service：通过 fileId 关联知识/模板素材
