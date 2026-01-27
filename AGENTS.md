# AI 代理协作说明（Codex）

> 本文件用于给 Codex 类工具提供仓库上下文与约束。
>
> 若本文件与 `CLAUDE.md` 不一致，以本文件为准。

## 1. 仓库角色

- `docs` 是 LawSeekDog 的文档站（GitHub Pages）。
- 目标：以“当前仓库实现”为准沉淀架构/流程/交付规范，避免文档漂移。

## 2. 技术基线

- Jekyll + `just-the-docs`（见 `_config.yml`）
- Mermaid 图由 GitHub 渲染（注意 Mermaid 语法兼容性）
- Pages 发布：`.github/workflows/pages.yml`（默认仅在打 `v*` tag 时发布，避免占用 Actions 配额）

## 3. 修改原则（强制）

- 文档必须与代码对齐：写“现状/规划”要明确区分；不把设想写成已实现。
- Mermaid 图优先使用 GitHub 兼容语法（避免连接到 subgraph 等易踩坑写法）。
- 不提交大体积二进制制品（图片/视频请控制体积；优先用矢量或外链）。

## 4. 本地预览（可选）

- `bundle install`
- `bundle exec jekyll serve --livereload`
