# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库工作时提供指引。

> 这是 monorepo 根的**生态级护栏**，每个子项目还有自己的 `CLAUDE.md`（叠加加载）。
> 进入某个子项目开发时，那份子 `CLAUDE.md` 是更具体的真理来源——先读它。

## 🚫 红线 / 动手前必读

1. **插件边界不可破**：插件只依赖 `airgate-sdk`，**禁止** import core 内部包；插件要用 core 能力只能经 `Host.Invoke` / `Host.InvokeStream`，不直接访问 core 的 DB/服务。SDK `protocol/` 是 ABI。
2. **生成代码禁止手改**，且改源后必须重新生成并提交：
   - `ent/`（非 `schema/` 部分）→ 改 `ent/schema/` 后 `make ent`
   - `*.pb.go` → 改 `protocol/proto/` 后 `make proto`
   - theme CSS → `make theme`
   - `plugin.yaml` → 由 `genmanifest`/`make manifest` 生成
   漏了重新生成会让 CI 的漂移检查（`verify-ent`/`proto-check`/`theme-check`）失败。
3. **动手前先读**：`airgate-core/docs/architecture/ecosystem-v2.md`（生态边界） ＋ 你所在子项目的 `CLAUDE.md`。
4. **复用优先**：先在对应层找现成 service/handler/组件照着改，别自创结构、别另起炉灶。
5. **各子目录是独立 Git 仓**（经 `go.work` 协作）：跨仓改动要在各自仓分别提交。
6. 代码注释/文档用**中文**，命令/路径/标识符/类型名保留**英文**。
7. **提交信息用约定式提交（Conventional Commits）**：`<type>(<scope>)?: <subject>`（type：`feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`）。各仓 `make setup-hooks` 会装 `commit-msg` 钩子强制校验（脚本在 `scripts/commit-msg`）。

## 任务路由：改什么 → 看哪里

| 你要做的事 | 子项目 / 文档 | Skill |
|---|---|---|
| 加/改 core 后端接口或领域逻辑 | `airgate-core/CLAUDE.md` | `core-backend-feature` |
| 改 Ent schema / 加字段表 | `airgate-core/CLAUDE.md` → 后端分层 | `core-backend-feature`（含 Ent 子流程） |
| 加/改 core 后台前端页面 | `airgate-core/CLAUDE.md` → 前端 | `core-frontend-page` |
| 开发/改网关或扩展插件 | 对应 `airgate-<plugin>/CLAUDE.md` + `airgate-sdk/CLAUDE.md` | `develop-plugin` |
| 改 SDK 接口 / proto / theme | `airgate-sdk/CLAUDE.md` | — |
| 声称"做完"之前的自检 | 各仓 `make ci` | `airgate-ci-check` |

## 仓库布局

这是一个 monorepo，包含 AirGate 生态——一个可插拔的 AI 网关运行时。同级目录是各自独立的 Git 仓，经 `go.work` 协同工作：

| 目录 | 职责 |
|---|---|
| `airgate-sdk/` | 共享 SDK：插件接口（`sdkgo/`）、gRPC 协议（`protocol/proto/`）、运行时桥（`runtimego/grpc/`）、devserver（`devkit/`）、前端主题（`theme/`） |
| `airgate-core/` | 核心运行时：用户/账号/计费/调度/后台 UI、插件生命周期管理 |
| `airgate-openai/` | 网关插件：OpenAI/Anthropic 协议转换 |
| `airgate-claude/` | 网关插件：Claude Messages API |
| `airgate-kiro/` | 网关插件：Kiro（AWS CodeWhisperer） |
| `airgate-playground/` | 扩展插件：Web 聊天 UI |
| `airgate-studio/` | 扩展插件：多模态内容创作 |
| `airgate-epay/` | 扩展插件：支付渠道 |
| `airgate-health/` | 扩展插件：提供商健康监控 |

## 架构

core 通过 hashicorp/go-plugin 把插件作为独立的 gRPC 子进程加载。请求流向：

```
用户 → Core（鉴权/限流/账号调度） → Plugin.Forward() → 上游 AI API
                                          ↓
                                  ForwardOutcome（用量/成本）
                                          ↓
                                  Core 计费 + 账号状态更新
```

关键边界：
- 插件只依赖 `airgate-sdk`，绝不依赖 Core 内部
- 插件经 `Host.Invoke` / `Host.InvokeStream` 调用 Core 能力（不直接访问 DB）
- 插件前端是单个 `index.js` bundle，由 Core 的插件资产处理器提供
- 每个插件有自己的 `go.work`，包含 `../airgate-sdk`

插件接口类型（声明于 `airgate-sdk/sdkgo/`）：

| 类型 | 接口 | 用途 |
|---|---|---|
| `gateway` | `sdk.GatewayPlugin` | 把请求转发到某 AI 平台；声明模型、路由、账号字段 |
| `extension` | `sdk.ExtensionPlugin` | 后台任务、自定义 API、支付、健康监控（非网关） |
| `middleware` | `sdk.MiddlewarePlugin` | 转发周边的副作用拦截器（审计、脱敏、采样、合规） |

权威架构参考：`airgate-core/docs/architecture/ecosystem-v2.md`。

## 生态边界（最重要——动手前必对照）

这套架构的价值就在于**可插拔**。任何越界都会让架构腐化，因此下面是所有开发（**包括改 core**）都要守的铁律。权威依据见 `ecosystem-v2.md` 的「职责速查表 / 当前边界问题 / 宿主能力模型」。

**① 谁负责什么（职责速查表，提炼自 ecosystem-v2）**

| 组件 | 负责 | **不负责（越界红灯）** |
|---|---|---|
| **Core** | 身份、账号、路由、任务、资产、模型目录、计费、插件生命周期 | 外部协议格式、上游 token/session、产品页面 |
| **Gateway 网关** | 对外 API 兼容：route、请求/响应格式、SSE/WS、错误形态 | 上游认证、provider session、任务持久化、计费策略 |
| **Provider 适配** | 上游对接：auth、token/session、传输、模型发现 | 对外 `/v1/...` route、产品 UI、Core task 状态、计费 |
| **UI 插件** | 页面、widget、产品状态、调 Core public API | provider 通信协议、gateway 兼容、task 执行 |
| **SDK/Protocol** | 稳定 ABI、插件作者 API、运行时/工具分层 | 把 core 产品逻辑/业务塞进 SDK |

判断规则：对外客户端兼容→Gateway；对上游厂商认证传输→Provider；跨 provider 的权限/路由/任务/资产/计费/模型→Core；用户产品页面状态→UI 插件。

**② 依赖方向（单向，不可逆）**

```
插件 ──依赖──▶ airgate-sdk ◀──依赖── core
插件 ──调用 core 能力──▶ 只能经 Host.Invoke / Host.InvokeStream
```
- 插件**禁止** import `airgate-core` 任何包，**禁止**直连 core 的 DB/Redis/服务。
- core **禁止** import 任何具体插件包；core 只通过 SDK 接口 + 插件 manifest 认识插件。
- SDK `protocol/` 是通信层 ABI：只增不破坏，破坏性变更同时打挂 core 和所有插件。

**③ 三类最常见越界（审计已记录，别再加深）**

1. **core 混入协议/产品逻辑**：转发层写 OpenAI 形态错误、调度/路由里特判 provider/模型字符串、core UI 硬编码插件产品路由 → 协议归网关、产品页面归 UI 插件。
2. **SDK 混入业务**：把账号/计费/任务等产品逻辑写进 SDK → SDK 只放契约与运行时。
3. **网关插件兼任 provider/UI**：把上游 OAuth/session、图像任务编排、账号 UI 都堆进一个网关插件 → 新增能力时按职责拆，别继续堆叠。

> 当前部分插件仍是"网关+provider+UI"的过渡混合态。**不要求你去拆历史代码，但新增/改动时按职责归位，不要加深混合。** 拿不准某能力放哪，先查职责速查表，再看 `ecosystem-v2.md`。

## 构建命令

### airgate-core（在 `airgate-core/` 下）

```bash
make install        # 安装所有依赖（Go + pnpm + 插件前端 + air）
make dev            # 启动完整开发环境（后端 air 热重载 + 前端 vite + 所有插件 watch）
make dev-backend    # 仅后端（装了 air 则用 air）
make dev-frontend   # 仅前端 vite 开发服务器
make build          # 完整生产构建（前端 → 嵌入 → 后端二进制）
make ent            # 改 schema 后重新生成 Ent ORM 代码
make ci             # 完整 CI 检查：lint + test + vet + verify-ent + build
make pre-commit     # 较轻量检查（跳过测试）
make lint           # golangci-lint + tsc + eslint
make test           # Go 测试（backend/）
make fmt            # goimports 格式化
make setup-hooks    # 安装 git pre-commit 钩子
```

跑单个 Go 测试（在 `airgate-core/backend/` 下）：

```bash
go test ./internal/plugin/... -run TestForwarder       # 一个包，一个测试
go test ./internal/billing -run TestQuota -v -count=1  # 详细输出，不用缓存
```

### airgate-sdk（在 `airgate-sdk/` 下）

```bash
make ci             # 完整 CI：lint + test + vet + build + proto-check + theme-package-check + theme-check
make proto          # 重新生成 protobuf 代码（装固定版本 protoc + 插件）
make proto-check    # 校验 proto 生成代码无漂移
make theme          # 构建主题包 + 生成 devserver theme.css
make test           # 带 race 检测 + 覆盖率的 Go 测试
make lint           # golangci-lint
```

### 插件仓（如在 `airgate-openai/` 下）

每个插件遵循同样的模式：

```bash
make install        # 安装前端 + 后端依赖
make dev            # 启动 devserver（脱离 Core 独立调试插件）
make build          # 完整构建：前端 → 嵌入 → Go 二进制
make ci             # lint + test + vet + build
make release        # 交叉编译 linux-amd64 二进制供上传
```

## 技术栈

- **后端**：Go 1.25、Gin、Ent ORM、PostgreSQL 17、Redis 8
- **前端**：React 19、Vite、TanStack Query、Tailwind CSS
- **插件协议**：hashicorp/go-plugin over gRPC
- **插件前端**：单个 `index.js` bundle，使用 `@doudou-start/airgate-theme`

## 关键约定

- Go module 路径前缀：`github.com/DouDOU-start/`
- 插件用 `go.work` 引用 `../airgate-sdk` 做本地开发
- 前端资产经 `//go:embed` 嵌入 Go 二进制
- Core 的 Ent schema 改动需 `make ent` 后提交生成代码
- SDK proto 改动需 `make proto` 后提交生成的 `.pb.go`
- 插件前端输出到 `web/dist/index.js`；dev 模式下 Core 从 `<plugin>/web/dist` 读取
- 生产环境 `make build-plugins` 把每个插件的 `web/dist` 同步到 `airgate-core/backend/data/plugins/<plugin-id>/assets`；Core 的 `servePluginAsset` 处理器从那里读取
- `plugin.yaml` 由 `backend/cmd/genmanifest`（Core）或 `make manifest`（插件）自动生成——绝不手改
- import 顺序：stdlib → 外部 → `github.com/DouDOU-start`（由 goimports 强制）
- Lint：Go 用 `golangci-lint`，前端用 `tsc --noEmit` + `eslint`
- CI 漂移检查：proto 生成代码、Ent 生成代码、theme CSS 都必须提交为最新
