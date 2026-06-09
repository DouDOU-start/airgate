# AGENTS.md

Codex 在本仓库的工作指引。
（内容与根 `CLAUDE.md` 同步，规则一致，Claude/Codex 通用。）

> monorepo 根级护栏。各子项目另有 `CLAUDE.md`（叠加加载）；进入子项目开发时以其为准，先读。

## 🚫 红线 / 前置必读

1. **插件边界**：插件仅依赖 `airgate-sdk`，禁止 import core 内部包；调用 core 能力仅经 `Host.Invoke` / `Host.InvokeStream`，禁止直连 core 的 DB/服务。`protocol/` 为 ABI。
2. **生成代码禁止手改**，改源后须重新生成并提交，否则 CI 漂移检查失败：
   - `ent/`（非 `schema/`）→ 改 `ent/schema/` 后 `make ent`
   - `*.pb.go` → 改 `protocol/proto/` 后 `make proto`
   - theme CSS → `make theme`
   - `plugin.yaml` → `genmanifest` / `make manifest` 生成
3. **前置必读**：`airgate-core/docs/architecture/current/`（**现状架构，开发依据，对得上代码**）及所在子项目 `CLAUDE.md`。
4. **复用优先**：复用对应层现有 service/handler/组件，勿新建结构。
5. **独立 Git 仓**：各子目录经 `go.work` 协作，跨仓改动分仓提交。
6. 注释/文档用**中文**，命令/路径/标识符/类型名保留**英文**。
7. **提交信息**：约定式提交 `<type>(<scope>)?: <subject>`（type：`feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`）。`make setup-hooks` 装 `commit-msg` 钩子（`scripts/commit-msg`）强制校验。
8. **文档对得上代码（防漂移）**：改动涉及架构（分层/契约/转发/计费/调度/插件职责）须同步更新 `airgate-core/docs/architecture/current/` 对应文档。现状以 `current/` 为准；冲突即漂移，须修正。

## 任务路由

| 任务 | 文档 | Skill |
|---|---|---|
| core 后端接口 / 领域逻辑 | `airgate-core/CLAUDE.md` | `core-backend-feature` |
| Ent schema / 字段表 | `airgate-core/CLAUDE.md` → 后端分层 | `core-backend-feature` |
| core 后台前端页面 | `airgate-core/CLAUDE.md` → 前端 | `core-frontend-page` |
| 网关 / 扩展插件 | `airgate-<plugin>/CLAUDE.md` + `airgate-sdk/CLAUDE.md` | `develop-plugin` |
| SDK 接口 / proto / theme | `airgate-sdk/CLAUDE.md` | — |
| 提交前自检 | 各仓 `make ci` | `airgate-ci-check` |

## 仓库布局

AirGate 生态：可插拔 AI 网关运行时。各子目录为独立 Git 仓，经 `go.work` 协作。

| 目录 | 职责 |
|---|---|
| `airgate-sdk/` | 共享 SDK：插件接口（`sdkgo/`）、gRPC 协议（`protocol/proto/`）、运行时桥（`runtimego/grpc/`）、devserver（`devkit/`）、前端主题（`theme/`） |
| `airgate-core/` | 核心运行时：用户/账号/计费/调度/后台 UI、插件生命周期 |
| `airgate-openai/` | 网关插件：OpenAI/Anthropic 协议转换 |
| `airgate-claude/` | 网关插件：Claude Messages API |
| `airgate-kiro/` | 网关插件：Kiro（AWS CodeWhisperer） |
| `airgate-playground/` | 扩展插件：Web 聊天 UI |
| `airgate-studio/` | 扩展插件：多模态内容创作 |
| `airgate-epay/` | 扩展插件：支付渠道 |
| `airgate-health/` | 扩展插件：提供商健康监控 |

## 架构

core 经 hashicorp/go-plugin 将插件作为独立 gRPC 子进程加载。请求流向：

```
用户 → Core（鉴权/限流/账号调度） → Plugin.Forward() → 上游 AI API
                                          ↓
                                  ForwardOutcome（用量/成本）
                                          ↓
                                  Core 计费 + 账号状态更新
```

关键边界：
- 插件仅依赖 `airgate-sdk`，不依赖 Core 内部
- 插件经 `Host.Invoke` / `Host.InvokeStream` 调用 Core 能力，不直连 DB
- 插件前端为单 `index.js` bundle，由 Core 插件资产处理器提供
- 各插件自带 `go.work`，含 `../airgate-sdk`

插件接口类型（声明于 `airgate-sdk/sdkgo/`）：

| 类型 | 接口 | 用途 |
|---|---|---|
| `gateway` | `sdk.GatewayPlugin` | 转发请求至 AI 平台；声明模型、路由、账号字段 |
| `extension` | `sdk.ExtensionPlugin` | 后台任务、自定义 API、支付、健康监控（非网关） |
| `middleware` | `sdk.MiddlewarePlugin` | 转发周边副作用拦截（审计、脱敏、采样、合规） |

权威架构参考：`airgate-core/docs/architecture/current/`。

## 生态边界（最重要，动手前必对照）

架构核心价值为**可插拔**，越界即腐化。以下为所有开发（含改 core）的铁律。下表为**目标边界**；当前代码的违反点（OpenAI 错误格式硬编码、provider 字符串特判等）见 `airgate-core/docs/architecture/current/tech-debt.md`，新代码勿加深。

**① 职责速查表**

| 组件 | 负责 | **不负责（越界红灯）** |
|---|---|---|
| **Core** | 身份、账号、路由、任务、资产、模型目录、计费、插件生命周期 | 外部协议格式、上游 token/session、产品页面 |
| **Gateway 网关** | 对外 API 兼容：route、请求/响应格式、SSE/WS、错误形态 | 上游认证、provider session、任务持久化、计费策略 |
| **Provider 适配** | 上游对接：auth、token/session、传输、模型发现 | 对外 `/v1/...` route、产品 UI、Core task 状态、计费 |
| **UI 插件** | 页面、widget、产品状态、调 Core public API | provider 通信协议、gateway 兼容、task 执行 |
| **SDK/Protocol** | 稳定 ABI、插件作者 API、运行时/工具分层 | core 产品逻辑 / 业务 |

判断规则：对外客户端兼容 → Gateway；上游厂商认证传输 → Provider；跨 provider 的权限/路由/任务/资产/计费/模型 → Core；用户产品页面状态 → UI 插件。

**② 依赖方向（单向）**

```
插件 ──依赖──▶ airgate-sdk ◀──依赖── core
插件 ──调用 core 能力──▶ 仅经 Host.Invoke / Host.InvokeStream
```
- 插件禁止 import `airgate-core`，禁止直连 core 的 DB/Redis/服务。
- core 禁止 import 具体插件包；仅经 SDK 接口 + 插件 manifest 识别插件。
- `protocol/` 为通信层 ABI：只增不破坏；破坏性变更将打挂 core 与全部插件。

**③ 三类常见越界（审计已记录，勿加深）**

1. **core 混入协议/产品逻辑**：转发层写 OpenAI 形态错误、调度/路由特判 provider/模型字符串、core UI 硬编码插件路由 → 协议归网关、产品页面归 UI 插件。
2. **SDK 混入业务**：账号/计费/任务等产品逻辑写进 SDK → SDK 仅放契约与运行时。
3. **网关插件兼任 provider/UI**：上游 OAuth/session、图像任务编排、账号 UI 堆入网关插件 → 按职责拆分。

> 部分插件仍为"网关+provider+UI"混合的过渡态。无需拆历史代码，但新增/改动须按职责归位、勿加深混合。归属存疑时查职责速查表与 `tech-debt.md`（差距与方向）。

## 构建命令

### airgate-core（`airgate-core/`）

```bash
make install        # 安装全部依赖（Go + pnpm + 插件前端 + air）
make dev            # 完整开发环境（后端热重载 + 前端 vite + 插件 watch）
make dev-backend    # 仅后端
make dev-frontend   # 仅前端 vite
make build          # 生产构建（前端 → 嵌入 → 后端二进制）
make ent            # 改 schema 后重新生成 Ent ORM 代码
make ci             # 完整 CI：lint + test + vet + verify-ent + build
make pre-commit     # 轻量检查（跳过测试）
make lint           # golangci-lint + tsc + eslint
make test           # Go 测试（backend/）
make fmt            # goimports 格式化
make setup-hooks    # 安装 git 钩子
```

单测（`airgate-core/backend/`）：

```bash
go test ./internal/plugin/... -run TestForwarder       # 单包单测
go test ./internal/billing -run TestQuota -v -count=1  # 详细、不缓存
```

### airgate-sdk（`airgate-sdk/`）

```bash
make ci             # lint + test + vet + build + proto-check + theme-package-check + theme-check
make proto          # 重新生成 protobuf 代码
make proto-check    # 校验 proto 生成代码无漂移
make theme          # 构建主题包 + 生成 devserver theme.css
make test           # Go 测试（race + 覆盖率）
make lint           # golangci-lint
```

### 插件仓（如 `airgate-openai/`）

```bash
make install        # 安装前后端依赖
make dev            # devserver 独立调试（脱离 Core）
make build          # 前端 → 嵌入 → Go 二进制
make ci             # lint + test + vet + build
make release        # 交叉编译 linux-amd64
```

## 技术栈

- **后端**：Go 1.25、Gin、Ent ORM、PostgreSQL 17、Redis 8
- **前端**：React 19、Vite、TanStack Query、Tailwind CSS
- **插件协议**：hashicorp/go-plugin over gRPC
- **插件前端**：单 `index.js` bundle，基于 `@doudou-start/airgate-theme`

## 关键约定

- Go module 前缀：`github.com/DouDOU-start/`
- 插件经 `go.work` 引用 `../airgate-sdk`
- 前端资产经 `//go:embed` 嵌入二进制
- Ent schema 改动须 `make ent` 后提交生成代码
- proto 改动须 `make proto` 后提交 `.pb.go`
- 插件前端输出至 `web/dist/index.js`；dev 模式 Core 从 `<plugin>/web/dist` 读取
- 生产环境 `make build-plugins` 同步各插件 `web/dist` 至 `airgate-core/backend/data/plugins/<plugin-id>/assets`，由 `servePluginAsset` 读取
- `plugin.yaml` 由 `genmanifest`（Core）/ `make manifest`（插件）生成，禁止手改
- import 顺序：stdlib → 外部 → `github.com/DouDOU-start`（goimports 强制）
- Lint：Go 用 `golangci-lint`，前端用 `tsc --noEmit` + `eslint`
- CI 漂移检查：proto、Ent、theme CSS 生成物须提交为最新
