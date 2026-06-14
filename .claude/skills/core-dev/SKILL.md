---
name: core-dev
description: airgate-core 全栈开发规范。后端接口/领域逻辑/Ent schema、后台前端页面、子系统（转发管线/调度/计费/任务状态机/插件契约）改动时使用。规定分层落点、改动顺序、子系统决策规则与约束，避免自创结构、破坏调度/计费/状态迁移或插件契约。
---

# airgate-core 开发规范

开发约定见 `airgate-core/CLAUDE.md`，生态边界见根 `CLAUDE.md`。

按任务类型读取对应规则文件（同目录下）：

| 任务 | 规则文件 |
|---|---|
| 后端接口 / 领域逻辑 / Ent schema | `backend.md` |
| 后台前端页面 | `frontend.md` |
| 转发管线 / 响应码处理 / 账号状态机 / 调度选号 / 计费 | `forwarding.md` |
| 任务状态机 | `task.md` |
| 插件契约 / Metadata / Host.Invoke / 技术债 | `plugin-contract.md` |

读取方法：用 Read 工具读 `.claude/skills/core-dev/<文件名>`。涉及多个领域时读取多个文件。

## 自检

```bash
make lint && make test          # 快检
make ent && git status          # 改过 schema：确认生成代码已更新
make dev-frontend               # 前端预览
make ci                         # 完整 CI（含 verify-ent）
```

详见 skill `airgate-ci-check`。
