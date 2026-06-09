---
name: develop-plugin
description: 开发或修改 AirGate 网关/扩展插件（airgate-openai、airgate-claude、airgate-kiro、airgate-playground、airgate-studio、airgate-epay、airgate-health）时使用。规定插件边界铁律（仅依赖 SDK、经 Host.Invoke 调 core、plugin.yaml 自动生成）、能力归属判断、前端单 bundle 约定与构建流程。
---

# 插件开发（网关 / 扩展）

所有插件同构：仅依赖 `airgate-sdk`，作为独立 gRPC 子进程被 core 加载。
本仓信息（id/type/上游）见该插件仓 `CLAUDE.md`；接口契约见 `airgate-sdk/CLAUDE.md`。

## 🚫 边界铁律

- 仅依赖 `airgate-sdk`，禁止 import `airgate-core` 任何内部包。
- 调用 core 能力（用量、配置、对话、账务）仅经 `Host.Invoke` / `Host.InvokeStream`，禁止直连 core 的 DB/服务。
- `plugin.yaml` 由 `make manifest` 生成、不可手改；模型/路由/账号字段在 Go 代码声明（见 `backend/internal/<name>/metadata.go`）。
- 前端为单 `index.js` bundle，输出至 `web/dist/index.js`，基于 `@doudou-start/airgate-theme`。

## 能力归属判断（最关键）

编码前按职责速查表（根 `CLAUDE.md`「生态边界」/ `ecosystem-v2.md`）定位，勿凭直觉就地堆放：

- 对外 API 兼容（route、请求/响应格式、SSE/错误形态）→ **Gateway**。
- 上游厂商认证/token/session/传输/模型发现 → **Provider**（当前多与网关同仓，新增时独立成模块，勿与对外路由耦合）。
- 跨 provider 共享的权限/路由/任务/资产/计费/模型目录 → **Core**，经宿主能力调用。
- 用户产品页面/状态 → **UI 插件**，经 Core public API。

🚩 若正要在网关插件写「上游 OAuth/session、任务持久化、计费策略、跨账号路由」，即为越界，归 Provider/Core。

## 插件类型

| 类型 | 接口 | 用途 |
|---|---|---|
| `gateway` | `sdk.GatewayPlugin` | 转发请求至 AI 平台；声明 models/routes/account fields，`Forward()` 返回 `ForwardOutcome`（usage/cost 交 core 计费） |
| `extension` | `sdk.ExtensionPlugin` | 后台任务、自定义 API、支付、健康监控等**非网关**能力 |
| `middleware` | `sdk.MiddlewarePlugin` | forward 周边副作用拦截（审计、脱敏、采样、合规） |

## 宿主能力：插件如何调用 core

插件不直连 core，而是声明并调用宿主能力（core 启动时按声明注入）：

- manifest 用 `requires.host` 声明所需能力（如 `host.routing@1`、`host.tasks@1`、`host.assets@1`、`host.billing@1`、`host.models@1`）。
- 运行时经 `Host.Invoke` / `Host.InvokeStream` 调用，禁止绕道 core 内部包或 DB。
- `requires.host` = core 注入给插件的宿主能力；`provides.operations` = 本插件对外可被编排的业务能力，二者勿混。
- 所需 host 能力 core 尚未暴露时，勿在插件侧复制 core 逻辑——应由 core 补充该能力，先确认而非绕过。

## 开发流程

1. 参照现有插件：网关参考 `airgate-openai` / `airgate-claude`，扩展参考 `airgate-playground` / `airgate-health`。
2. 改 Go 实现（`backend/internal/<name>/`）；改动声明类元信息后重新生成 manifest。
3. 调 core 能力经 `Host.Invoke`，勿绕道。
4. 前端仍为单 `index.js`，复用 theme。
5. `make manifest` 重新生成 `plugin.yaml`。
6. `make dev` 经 devserver 脱离 core 独立调试。
7. `make ci`（lint + test + vet + build）自检。

## 构建 / 发布

```bash
make install   # 前后端依赖
make dev       # devserver 独立调试
make manifest  # 重新生成 plugin.yaml（改模型/路由/账号字段后）
make build     # 前端 → 嵌入 → Go 二进制
make ci        # lint + test + vet + build
make release   # 交叉编译 linux-amd64
```

生产部署时，core 的 `make build-plugins` 将各插件 `web/dist` 同步至 `airgate-core/backend/data/plugins/<plugin-id>/assets`，由 `servePluginAsset` 提供。
