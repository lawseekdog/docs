---
title: 进度与验收如何核验
parent: 架构
nav_order: 6
---

# 进度与验收如何核验

本文不保存易过期的“当前全部完成”清单。恢复任务时分别核验：

| 状态 | 必须具备的证据 |
| --- | --- |
| 已实现、已提交 | 仓库实际 diff、commit 与受影响测试 |
| 已推送或合入 main | fresh fetch 后的远端 ref 与祖先关系 |
| 已部署 | 明确云目标、实际镜像/提交、rollout 与健康读回 |
| 已验收 | 同一部署窗口内的真实 Session、Owner request/result/readback、UI 和适用法律内容审阅 |

静态测试不能证明模型调用了保存工具，DSH Tool 成功不能单独证明读回正文一致，历史复核也不证明当前版本通过。跨服务部分失败须列出实际已完成与未完成对象，不能由总计通过数掩盖。

每轮记录留在工作区 `output/` 的独立 artifact 目录，包含测试 scope、部署版本、真实身份、失败样本和未覆盖分支，不把临时 ID、密钥或日志复制到永久文档。部署中变更版本会改变验收窗口，保留旧样本并核验新窗口。

边界见[架构说明](../LAWSEEKDOG-ARCHITECTURE-BOUNDARIES.md)，执行入口见 `TASK-ROUTING.md` 和 `OPERATIONS.md`。不存在需要另建一套本地全栈或永久质量 gate 的要求。
