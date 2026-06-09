# AGENTS.md

This file provides guidance to Codex when working with code in this repository.
（内容与根 `CLAUDE.md` 保持同步——同一套规则，Claude/Codex 通用。）

> 这是 monorepo 根的**生态级护栏**，每个子项目还有自己的 `CLAUDE.md`（叠加加载）。
> 进入某个子项目开发时，那份子 `CLAUDE.md` 是更具体的真理来源——先读它。

## 🚫 红线 / 动手前必读

1. **插件边界不可破**：插件只依赖 `airgate-sdk`，**禁止** import core 内部包；插件要用 core 能力只能经 `Host.Invoke` / `Host.InvokeStream`，不直接访问 core 的 DB/服务。SDK `protocol/` 是 ABI。
2. **生成代码禁止手改**，且改源后必须 regen 并提交：
   - `ent/`（非 `schema/` 部分）→ 改 `ent/schema/` 后 `make ent`
   - `*.pb.go` → 改 `protocol/proto/` 后 `make proto`
   - theme CSS → `make theme`
   - `plugin.yaml` → 由 `genmanifest`/`make manifest` 生成
   漏 regen 会让 CI 的 drift 检查（`verify-ent`/`proto-check`/`theme-check`）失败。
3. **动手前先读**：`airgate-core/docs/architecture/ecosystem-v2.md`（生态边界） ＋ 你所在子项目的 `CLAUDE.md`。
4. **复用优先**：先在对应层找现成 service/handler/组件照着改，别自创结构、别另起炉灶。
5. **各子目录是独立 Git 仓**（经 `go.work` 协作）：跨仓改动要在各自仓分别提交。
6. 代码注释/文档用**中文**，术语/命令/路径/标识符保留**英文**。
7. **Commit 用 Conventional Commits**：`<type>(<scope>)?: <subject>`（type: `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`）。各仓 `make setup-hooks` 会装 `commit-msg` 钩子强制校验（脚本在 `scripts/commit-msg`）。

## 任务路由：改什么 → 看哪里

| 你要做的事 | 子项目 / 文档 | Skill |
|---|---|---|
| 加/改 core 后端接口或领域逻辑 | `airgate-core/CLAUDE.md` | `core-backend-feature` |
| 改 Ent schema / 加字段表 | `airgate-core/CLAUDE.md` → 后端分层 | `core-backend-feature`（含 Ent 子流程） |
| 加/改 core 后台前端页面 | `airgate-core/CLAUDE.md` → 前端 | `core-frontend-page` |
| 开发/改网关或扩展插件 | 对应 `airgate-<plugin>/CLAUDE.md` + `airgate-sdk/CLAUDE.md` | `develop-plugin` |
| 改 SDK 接口 / proto / theme | `airgate-sdk/CLAUDE.md` | — |
| 声称"做完"之前的自检 | 各仓 `make ci` | `airgate-ci-check` |

## Repository Layout

This is a monorepo containing the AirGate ecosystem — a pluggable AI gateway runtime. Sibling directories are separate Git repos that work together via `go.work`:

| Directory | Role |
|---|---|
| `airgate-sdk/` | Shared SDK: plugin interfaces (`sdkgo/`), gRPC protocol (`protocol/proto/`), runtime bridge (`runtimego/grpc/`), devserver (`devkit/`), frontend theme (`theme/`) |
| `airgate-core/` | Core runtime: user/account/billing/scheduling/admin UI, plugin lifecycle manager |
| `airgate-openai/` | Gateway plugin: OpenAI/Anthropic protocol translation |
| `airgate-claude/` | Gateway plugin: Claude Messages API |
| `airgate-kiro/` | Gateway plugin: Kiro (AWS CodeWhisperer) |
| `airgate-playground/` | Extension plugin: web chat UI |
| `airgate-studio/` | Extension plugin: multi-modal content creation |
| `airgate-epay/` | Extension plugin: payment channels |
| `airgate-health/` | Extension plugin: provider health monitoring |

## Architecture

Core loads plugins as independent gRPC subprocesses via hashicorp/go-plugin. The request flow:

```
User → Core (auth/ratelimit/account scheduling) → Plugin.Forward() → Upstream AI API
                                                        ↓
                                              ForwardOutcome (usage/cost)
                                                        ↓
                                              Core billing + account status update
```

Key boundaries:
- Plugins depend only on `airgate-sdk`, never on Core internals
- Plugins call Core capabilities via `Host.Invoke` / `Host.InvokeStream` (no direct DB access)
- Plugin frontends are single `index.js` bundles served by Core's plugin asset handler
- Each plugin has its own `go.work` that includes `../airgate-sdk`

Plugin interface types (declared in `airgate-sdk/sdkgo/`):

| Type | Interface | Purpose |
|---|---|---|
| `gateway` | `sdk.GatewayPlugin` | Forward requests to an AI platform; declares models, routes, account fields |
| `extension` | `sdk.ExtensionPlugin` | Background jobs, custom APIs, payments, health monitoring (non-gateway) |
| `middleware` | `sdk.MiddlewarePlugin` | Side-effect interceptors around forward (audit, masking, sampling, compliance) |

Canonical architecture reference: `airgate-core/docs/architecture/ecosystem-v2.md`.

## 生态边界（最重要——动手前必对照）

这套架构的价值就在于**可插拔**。任何越界都会让架构腐化，因此下面是所有开发（**包括改 core**）都要守的铁律。权威依据见 `ecosystem-v2.md` 的「职责速查表 / 当前边界问题 / Host capability 模型」。

**① 谁负责什么（职责速查表，提炼自 ecosystem-v2）**

| 组件 | 负责 | **不负责（越界红灯）** |
|---|---|---|
| **Core** | 身份、账号、路由、任务、资产、模型目录、计费、插件生命周期 | 外部协议格式、上游 token/session、产品页面 |
| **Gateway 网关** | 对外 API 兼容：route、请求/响应格式、SSE/WS、错误形态 | 上游认证、provider session、任务持久化、计费策略 |
| **Provider 适配** | 上游对接：auth、token/session、传输、模型发现 | 对外 `/v1/...` route、产品 UI、Core task 状态、计费 |
| **UI 插件** | 页面、widget、产品状态、调 Core public API | provider wire protocol、gateway 兼容、task 执行 |
| **SDK/Protocol** | 稳定 ABI、插件作者 API、运行时/工具分层 | 把 core 产品逻辑/业务塞进 SDK |

判断规则：对外客户端兼容→Gateway；对上游厂商认证传输→Provider；跨 provider 的权限/路由/任务/资产/计费/模型→Core；用户产品页面状态→UI 插件。

**② 依赖方向（单向，不可逆）**

```
插件 ──依赖──▶ airgate-sdk ◀──依赖── core
插件 ──调用 core 能力──▶ 只能经 Host.Invoke / Host.InvokeStream
```
- 插件**禁止** import `airgate-core` 任何包，**禁止**直连 core 的 DB/Redis/服务。
- core **禁止** import 任何具体插件包；core 只通过 SDK 接口 + 插件 manifest 认识插件。
- SDK `protocol/` 是 wire ABI：只增不破坏，破坏性变更同时打挂 core 和所有插件。

**③ 三类最常见越界（审计已记录，别再加深）**

1. **core 混入协议/产品逻辑**：转发层写 OpenAI 形态错误、调度/路由里特判 provider/模型字符串、core UI 硬编码插件产品路由 → 协议归网关、产品页面归 UI 插件。
2. **SDK 混入业务**：把账号/计费/任务等产品逻辑写进 SDK → SDK 只放契约与运行时。
3. **网关插件兼任 provider/UI**：把上游 OAuth/session、图像任务编排、账号 UI 都堆进一个网关插件 → 新增能力时按职责拆，别继续堆叠。

> 当前部分插件仍是"网关+provider+UI"的过渡混合态。**不要求你去拆历史代码，但新增/改动时按职责归位，不要加深混合。** 拿不准某能力放哪，先查职责速查表，再看 `ecosystem-v2.md`。

## Build Commands

### airgate-core (from `airgate-core/`)

```bash
make install        # Install all deps (Go + pnpm + plugin frontends + air)
make dev            # Start full dev env (backend with air hot-reload + frontend vite + all plugin watches)
make dev-backend    # Backend only (uses air if installed)
make dev-frontend   # Frontend vite dev server only
make build          # Full production build (frontend → embed → backend binary)
make ent            # Regenerate Ent ORM code after schema changes
make ci             # Full CI check: lint + test + vet + verify-ent + build
make pre-commit     # Lighter check (skips tests)
make lint           # golangci-lint + tsc + eslint
make test           # Go tests (backend/)
make fmt            # goimports formatting
make setup-hooks    # Install git pre-commit hook
```

Run a single Go test (from `airgate-core/backend/`):

```bash
go test ./internal/plugin/... -run TestForwarder       # one package, one test
go test ./internal/billing -run TestQuota -v -count=1  # verbose, no cache
```

### airgate-sdk (from `airgate-sdk/`)

```bash
make ci             # Full CI: lint + test + vet + build + proto-check + theme-package-check + theme-check
make proto          # Regenerate protobuf code (installs pinned protoc + plugins)
make proto-check    # Verify proto generated code has no drift
make theme          # Build theme package + generate devserver theme.css
make test           # Go tests with race detection + coverage
make lint           # golangci-lint
```

### Plugin repos (e.g. from `airgate-openai/`)

Each plugin follows the same pattern:

```bash
make install        # Install frontend + backend deps
make dev            # Start devserver (standalone plugin debugging without Core)
make build          # Full build: frontend → embed → Go binary
make ci             # lint + test + vet + build
make release        # Cross-compile linux-amd64 binary for upload
```

## Tech Stack

- **Backend**: Go 1.25, Gin, Ent ORM, PostgreSQL 17, Redis 8
- **Frontend**: React 19, Vite, TanStack Query, Tailwind CSS
- **Plugin protocol**: hashicorp/go-plugin over gRPC
- **Plugin frontend**: single `index.js` bundle using `@doudou-start/airgate-theme`

## Key Conventions

- Go module path prefix: `github.com/DouDOU-start/`
- Plugins use `go.work` to reference `../airgate-sdk` for local development
- Frontend assets are embedded into Go binaries via `//go:embed`
- Core's Ent schema changes require `make ent` then commit the generated code
- SDK proto changes require `make proto` then commit generated `.pb.go` files
- Plugin frontends output to `web/dist/index.js`; Core reads from `<plugin>/web/dist` in dev mode
- In prod, `make build-plugins` syncs each plugin's `web/dist` to `airgate-core/backend/data/plugins/<plugin-id>/assets`; Core's `servePluginAsset` handler reads from there
- `plugin.yaml` is auto-generated by `backend/cmd/genmanifest` (Core) or `make manifest` (plugins) — never edit by hand
- Import ordering: stdlib → external → `github.com/DouDOU-start` (enforced by goimports)
- Linting: `golangci-lint` for Go, `tsc --noEmit` + `eslint` for frontend
- CI drift checks: proto generated code, Ent generated code, theme CSS must all be committed up-to-date
