---
name: core-backend-feature
description: 在 airgate-core 后端新增或修改 API 接口、领域逻辑、Ent schema 字段/表时使用。规定分层落点与改动顺序（dto → handler → service → store → ent + 装配），含 Ent 变更子流程，避免自创结构、漏装配、漏 make ent。
---

# core 后端开发（接口 / 领域逻辑 / Ent 变更）

airgate-core 后端为严格的端口-适配器分层。按以下落点与顺序开发，勿自创结构。
现状实现见 `airgate-core/docs/architecture/current/core-runtime.md`，开发规则见 `airgate-core/CLAUDE.md`。

## 第 0 步：参照现有实现

开发前完整阅读一个同形态领域作模板，首选 `account`：
- `backend/internal/server/dto/account.go`
- `backend/internal/server/handler/account_handler.go`（+ `_mapper.go` / `_routes.go`）
- `backend/internal/app/account/service.go`（+ `types.go` / `errors.go`）
- `backend/internal/infra/store/account_store.go`

新代码沿用其结构。

## 分层落点（自上而下）

| 层 | 文件 | 职责 |
|---|---|---|
| DTO | `internal/server/dto/<domain>.go` | 请求/响应结构，`json`/`binding` tag |
| Handler | `internal/server/handler/<domain>_handler{,_routes,_mapper}.go` | 绑定校验 → 调 service → `toXResp` 映射 → `response.*` |
| Service | `internal/app/<domain>/{service,types,errors}.go` | 业务逻辑；`Input/Filter/Result` 与 `Repository` 接口在 `types.go`；哨兵错误在 `errors.go` |
| Store | `internal/infra/store/<domain>_store.go` | `Repository` 的 ent 实现（**仅此层 import ent**） |
| Schema | `ent/schema/<entity>.go` | DB 表/字段/索引 |

## 标准改动顺序

1. **（如需库变更）改 `ent/schema/`** → 见下方 Ent 子流程。
2. **Service**：`types.go` 加 `Input/Filter/Result` 与 `Repository` 方法；`errors.go` 加哨兵错误；`service.go` 写业务逻辑（**不碰 gin/http，不 import ent**）。
3. **Store**：`infra/store/<domain>_store.go` 实现新增 `Repository` 方法（ent 查询）。
4. **DTO**：`server/dto/<domain>.go` 定义请求/响应结构。
5. **Handler**：`server/handler/` 编写端点——`c.ShouldBind*` 绑定 → `h.service.X(ctx, Input{...})` → `toXResp` 映射（置于 `_mapper.go`）→ `response.Success/Error`，错误经 `h.handleError(...)`。
6. **装配（两处）**：
   - `internal/bootstrap/http_handlers.go`：`NewHTTPHandlers` 内按 `store → service → handler` 构造，挂载至 `HTTPHandlers`。
   - `internal/server/router.go`：`registerRoutes()` 选对分组（`v1` / `userGroup` / `adminGroup` / `extGroup`）注册路由，引用 `handlers.<X>.<Method>`。
7. **测试**：同包 `_test.go`，表驱动；service 层逻辑必测。

## Ent 变更子流程

1. 编辑 `ent/schema/<entity>.go`（字段/边/索引/mixin）。
2. `airgate-core/` 下运行 **`make ent`** 重新生成。
3. 提交生成代码（`ent/` 生成部分）；漏提交将致 `make ci` 的 `verify-ent` 失败。
4. 生成代码不可手改，仅改 `schema/`。

## 🚫 禁止

- handler 编写业务规则、或直接 `import ent` 查库。
- service 出现 `*gin.Context` / HTTP 状态码 / `response.*`。
- 新接口手拼 `map[string]any` 作响应而不定义 DTO（统计/SSE 等沿用同域现有写法除外）。
- 改 schema 后未 `make ent` 即提交。
- handler 硬编码 HTTP 状态码、或将内部 err 直接返回前端：应经 service 哨兵错误（`errors.go`）+ handler `handleError` 映射。⚠️ 上游账号失效用 **422**，非 401（401 触发前端拦截、登出管理员）。其余编码约定（slog、凭证加密、ctx 透传、分页/时区）见 `airgate-core/CLAUDE.md`「后端编码约定」。

## 完成前自检

```bash
make lint && make test          # 快检
make ent && git status          # 改过 schema：确认生成代码已更新并纳入提交
```
完整把关运行 `make ci`（含 verify-ent）；详见 skill `airgate-ci-check`。
