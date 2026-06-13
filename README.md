# AirGate 生态

可插拔 AI 网关运行时。Core 经 hashicorp/go-plugin 将插件作为独立 gRPC 子进程加载，统一完成鉴权、账号调度、转发、计费；协议兼容与上游对接由插件承担。

本目录为 monorepo 工作区：各子目录是**独立 Git 仓**，经 `go.work` 协作开发，跨仓改动分仓提交。

## 仓库清单（9 个子仓）

| 仓 | 类型 | 职责 |
|---|---|---|
| [`airgate-core/`](airgate-core/) | 运行时 | 用户/账号/计费/调度/任务/资产/后台 UI、插件生命周期 |
| [`airgate-sdk/`](airgate-sdk/) | 契约层 | 插件接口、gRPC 协议（ABI）、运行时桥、devserver、前端主题 |
| [`airgate-openai/`](airgate-openai/) | 网关插件 | OpenAI 协议 + Anthropic 转换 |
| [`airgate-claude/`](airgate-claude/) | 网关插件 | Claude Messages API |
| [`airgate-kiro/`](airgate-kiro/) | 网关插件 | Kiro（AWS CodeWhisperer） |
| [`airgate-playground/`](airgate-playground/) | 扩展插件 | Web 聊天 UI |
| [`airgate-studio/`](airgate-studio/) | 扩展插件 | 多模态内容创作 |
| [`airgate-epay/`](airgate-epay/) | 扩展插件 | 支付渠道 |
| [`airgate-health/`](airgate-health/) | 扩展插件 | 提供商健康监控 |

## 快速开始

```bash
cd airgate-core
make install   # 安装全部依赖（Go + pnpm + 插件前端 + air）
make dev       # 完整开发环境（后端热重载 + 前端 vite + 插件 watch）
```

部署与配置详见 [`airgate-core/README.md`](airgate-core/README.md)。

## 文档地图（先看这里，避免找错文档）

| 想了解 | 看哪份 |
|---|---|
| **架构现状（权威，对得上代码）** | [`airgate-core/docs/architecture/current/`](airgate-core/docs/architecture/current/)：运行时 / 插件契约 / 插件现状 / 技术债 |
| 生态边界与开发护栏 | 根 [`CLAUDE.md`](CLAUDE.md)（职责速查表、红线）；各子仓 `CLAUDE.md` 叠加本仓规则 |
| core ↔ 插件契约细节 | [`plugin-contract.md`](airgate-core/docs/architecture/current/plugin-contract.md)（proto、Host.Invoke、Metadata 约定键、manifest） |
| 已知架构差距 / 勿加深清单 | [`tech-debt.md`](airgate-core/docs/architecture/current/tech-debt.md) |
| 边界治理 / 重构路线（规划） | [`boundary-refactor-plan.md`](airgate-core/docs/architecture/boundary-refactor-plan.md) |
| 任务子系统设计 | [`task-state-machine.md`](airgate-core/docs/architecture/task-state-machine.md) |
| SDK 包内边界规范 | [`airgate-sdk/docs/sdk-package-boundaries.md`](airgate-sdk/docs/sdk-package-boundaries.md) |
| 插件前端样式规范 | [`airgate-sdk/docs/plugin-style-guide.md`](airgate-sdk/docs/plugin-style-guide.md) |
| 历史设计稿（仅存档，勿作依据） | `airgate-core/docs/architecture/archived/` |

文档维护规则：**现状以 `current/` 为准，改架构须同步改文档**（防漂移红线，见根 `CLAUDE.md`）。设计/规划类文档须在开头声明定位，与现状文档区分。

## 开发

AI 辅助开发护栏见 [`CLAUDE.md`](CLAUDE.md)（[`AGENTS.md`](AGENTS.md) 为其指针）。共享 skill 位于 `.claude/skills/`：`core-backend-feature` / `core-frontend-page` / `develop-plugin` / `airgate-ci-check`。
