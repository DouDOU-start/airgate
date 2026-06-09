---
name: develop-plugin
description: 开发或修改 AirGate 网关/扩展插件（airgate-openai、airgate-claude、airgate-kiro、airgate-playground、airgate-studio、airgate-epay、airgate-health）时使用。钉死插件边界铁律（只依赖 SDK、经 Host.Invoke 调 core、plugin.yaml 自动生成）、前端单 bundle 约定与构建/发布流程，避免越界 import core 或手改生成物。
---

# 插件开发（网关 / 扩展）

所有插件同构：只依赖 `airgate-sdk`，作为独立 gRPC 子进程被 core 加载。
本仓特定信息（id/type/上游）见该插件仓自己的 `CLAUDE.md`；接口契约见 `airgate-sdk/CLAUDE.md`。

## 🚫 边界铁律（最容易踩）

- **只依赖 `airgate-sdk`**，**禁止** import `airgate-core` 任何内部包。
- 要用 core 能力（用量、配置、对话、账务…）**只能经 `Host.Invoke` / `Host.InvokeStream`**，绝不直连 core 的 DB/服务。
- **`plugin.yaml` 由 `make manifest` 生成，不可手改**：模型、路由、账号字段都在 Go 代码里声明（看 `backend/internal/<name>/metadata.go` 一类文件）。
- 前端是**单个 `index.js` bundle**，输出到 `web/dist/index.js`，用 `@doudou-start/airgate-theme`。

## 先判断：这个能力到底该放哪（最关键）

写代码前先按**职责速查表**（见根 `CLAUDE.md`「生态边界」/ `ecosystem-v2.md`）定位，别凭手感往当前文件里塞：

- 对外 API 兼容（route、请求/响应格式、SSE/错误形态）→ **Gateway**。
- 对上游厂商的认证/token/session/传输/模型发现 → **Provider 职责**（当前多与网关混在一仓，新增时至少独立成模块，别和对外路由耦合）。
- 跨 provider 共享的权限/路由/任务/资产/计费/模型目录 → **不是插件该做的，归 Core**，经宿主能力调用。
- 用户产品页面/状态 → **UI 插件**，经 Core public API。

🚩 自检：如果你正要在网关插件里写"上游 OAuth/session、任务持久化、计费策略、跨账号路由"——停，那是 Provider/Core 的活，正在加深越界。

## 选对插件类型

| 类型 | 接口 | 用途 |
|---|---|---|
| `gateway` | `sdk.GatewayPlugin` | 转发请求到某 AI 平台；声明 models/routes/account fields，`Forward()` 返回 `ForwardOutcome`（usage/cost 交 core 计费） |
| `extension` | `sdk.ExtensionPlugin` | 后台任务、自定义 API、支付、健康监控等**非网关**能力 |
| `middleware` | `sdk.MiddlewarePlugin` | forward 周边的副作用拦截（审计、脱敏、采样、合规） |

## 宿主能力：插件怎么用 core 的能力

插件**不直连** core，而是声明并调用宿主能力（core 启动时按声明注入）：

- 在 manifest 用 `requires.host` 声明需要的能力（如 `host.routing@1`、`host.tasks@1`、`host.assets@1`、`host.billing@1`、`host.models@1`）。
- 运行时经 `Host.Invoke` / `Host.InvokeStream` 调用，**不要**找 core 内部包或 DB 的捷径。
- 区分：`requires.host` = core 注入给插件的宿主能力；`provides.operations` = 本插件对外可被编排的业务能力。两者别混。
- 需要新的 host 能力但 core 还没暴露时，**不要在插件侧硬抄 core 逻辑**——这是 core 该补的能力，先确认而不是绕过。

## 开发流程

1. **先读现成插件作模板**：网关参考 `airgate-openai` / `airgate-claude`；扩展参考 `airgate-playground` / `airgate-health`。
2. 改 Go 实现（`backend/internal/<name>/`）；声明类元信息改完要重生成 manifest。
3. 要调 core 能力 → 用 `Host.Invoke`，不要找捷径。
4. 改前端 → 仍是单 `index.js`，复用 theme。
5. **`make manifest`** 重新生成 `plugin.yaml`。
6. **`make dev`** 用 devserver 脱离 core 独立调试。
7. **`make ci`**（lint+test+vet+build）自检。

## 构建 / 发布

```bash
make install   # 前端 + 后端依赖
make dev       # devserver 独立调试
make manifest  # 重新生成 plugin.yaml（改了模型/路由/账号字段后）
make build     # 前端 → embed → Go 二进制
make ci        # lint + test + vet + build
make release   # 交叉编译 linux-amd64，供上传
```

生产部署时，core 的 `make build-plugins` 会把各插件 `web/dist` 同步到
`airgate-core/backend/data/plugins/<plugin-id>/assets`，由 core 的 `servePluginAsset` 提供。
