---
title: GitHub Delivery（LSD Delivery）
parent: 核心实现
nav_order: 90
---

# GitHub Delivery（LSD Delivery）

本页描述 LawSeekDog 的 **多 Repo、多 AI/多人** 协作交付方式：让不同人/不同 AI agent 通过同一套 **GitHub Issues → PR → Projects** 协作，而不是“AI 之间对话”。

核心目标：把每次 AI 开发变成**可领取、可限域、可追踪、可验收**的 Work Item，并且让 AI 在流程中自动维护看板状态。

## 1) 三个协作对象（必须）

1. **Issue（Work Item）**：工作的真源（范围/验收/测试/风险都在这里）。
2. **PR**：实现与审查的载体（必须关闭 Issue：`Closes #<issue>`）。
3. **Projects 看板（LawSeekDog Delivery）**：跨仓统一 tasklist（状态、负责人、优先级、Agent 等字段）。

> 约束：**1 个 Work Item = 1 个 repo = 1 个 PR**（跨仓改动拆成 Epic + 多个子 Issue，每个 repo 一个子 Issue）。

## 2) Issue 必填字段（用于“管住 AI”）

每个 Work Item Issue 必须包含（组织模板已内置这些段落）：

- Target Repo（目标仓库）
- Allowed Scope（允许修改的路径/glob）
- Out of Scope（禁止触碰的路径/文件）
- Acceptance Criteria（可勾选的验收标准）
- Test Plan（命令或验证步骤）
- Risk（Low/Med/High）

这些字段会被 CI 校验（Scope Guard），用于阻止越界修改。

## 3) 团队统一入口：lsd-delivery skill（单一事实来源）

不要在各仓库复制脚本/流程文档；统一以组织仓库 `lawseekdog/.github` 提供的 team skill 为准：

- 安装：`npx -y skills add lawseekdog/.github@lsd-delivery -g -y`
- 验证：`~/.agents/skills/lsd-delivery/scripts/lsd-work doctor`
- 完整可用命令：`~/.agents/skills/lsd-delivery/scripts/lsd-work --help`

## 4) 标准日常流程（AI 或人都一样）

### 4.1 没有 Issue 时（推荐）

让 AI 直接创建并领取 Issue（并同步 Projects 字段）：

```bash
~/.agents/skills/lsd-delivery/scripts/lsd-work kickoff \
  --repo lawseekdog/<repo> \
  --type Feature \
  --title "[FEATURE] ..." \
  --allowed "src/**" \
  --ac "..." \
  --test "..." \
  --priority Medium \
  --agent Codex \
  --sync-labels
```

### 4.2 已有 Issue 时

领取（会设置 Projects 字段 + assign 自己 + 留痕评论）：

```bash
~/.agents/skills/lsd-delivery/scripts/lsd-work claim <issue-url> --agent Codex --sync-labels
```

开发分支（建议用 GitHub CLI 绑定 issue）：

```bash
gh issue develop <num> --checkout
```

开 PR 后把状态推进到 `In Review`（或用 `lsd-work pr-open`）：

```bash
~/.agents/skills/lsd-delivery/scripts/lsd-work move <issue-url> --status "In Review" --sync-labels
```

合并后把状态推进到 `Done`：

```bash
~/.agents/skills/lsd-delivery/scripts/lsd-work move <issue-url> --status Done --sync-labels
```

## 5) 看板

- 项目：LawSeekDog Delivery（Projects v2 #1）
- 规则与字段以看板 Readme 为准（Status / Agent / Work Type / Priority / Target Repo / Epic 等）

