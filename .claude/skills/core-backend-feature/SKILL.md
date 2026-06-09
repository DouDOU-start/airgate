---
name: core-backend-feature
description: 在 airgate-core 后端新增或修改 API 接口、领域业务逻辑、Ent schema 字段/表时使用。钉死 core 的分层落点与改动顺序（dto → handler → service → store → ent + 装配），并含 Ent 变更子流程，避免自创结构、漏装配、漏 make ent 导致 CI漂移。
---

# core 后端开发（接口 / 领域逻辑 / Ent 变更）

airgate-core 后端是严格的端口-适配器分层。**别自创结构**——照下面的落点和顺序走。
背景见 `airgate-core/CLAUDE.md` 与 `airgate-core/docs/architecture/ecosystem-v2.md`。

## 第 0 步：先抄现成的

动手前先完整读一个同形态的现成域作模板，**首选 `account`**：
- `backend/internal/server/dto/account.go`
- `backend/internal/server/handler/account_handler.go`（+ `_mapper.go` / `_routes.go`）
- `backend/internal/app/account/service.go`（+ `types.go` / `errors.go`）
- `backend/internal/infra/store/account_store.go`

新代码照它的形状写。

## 分层落点（自上而下）

| 层 | 文件 | 放什么 |
|---|---|---|
| DTO | `internal/server/dto/<domain>.go` | 请求/响应结构，`json`/`binding` tag |
| Handler | `internal/server/handler/<domain>_handler{,_routes,_mapper}.go` | 绑定校验 → 调 service → `toXResp` 映射 → `response.*` |
| Service | `internal/app/<domain>/{service,types,errors}.go` | 业务逻辑；`Input/Filter/Result` 与 `Repository` 接口在 `types.go`；哨兵错误在 `errors.go` |
| Store | `internal/infra/store/<domain>_store.go` | `Repository` 的 ent 实现（**仅此层 import ent**） |
| Schema | `ent/schema/<entity>.go` | DB 表/字段/索引 |

## 标准改动顺序

1. **（如需库变更）改 `ent/schema/`** → 见下方 Ent 子流程。
2. **Service**：在 `app/<domain>/types.go` 加 `Input/Filter/Result` 和 `Repository` 方法；`errors.go` 加哨兵错误；`service.go` 写业务逻辑（**不碰 gin/http，不 import ent**）。
3. **Store**：在 `infra/store/<domain>_store.go` 实现新增的 `Repository` 方法（用 ent 查询）。
4. **DTO**：在 `server/dto/<domain>.go` 定义请求/响应结构。
5. **Handler**：在 `server/handler/` 写端点方法——`c.ShouldBind*` 绑定 → 调 `h.service.X(ctx, Input{...})` → `toXResp` 映射（放 `_mapper.go`）→ `response.Success/Error`。错误用 `h.handleError(...)`。
6. **装配（两处都要）**：
   - `internal/bootstrap/http_handlers.go`：在 `NewHTTPHandlers` 里按 `store → service → handler` 构造，挂到 `HTTPHandlers`。
   - `internal/server/router.go`：在 `registerRoutes()` 选对分组（`v1` / `userGroup` / `adminGroup` / `extGroup` …）注册路由，引用 `handlers.<X>.<Method>`。
7. **测试**：同包 `_test.go`，表驱动。service 层逻辑必测。

## Ent 变更子流程

1. 编辑 `ent/schema/<entity>.go`（字段/边/索引/mixin）。
2. 从 `airgate-core/` 跑 **`make ent`** 重新生成。
3. **提交生成代码**（`ent/` 下生成部分）——漏提交会让 `make ci` 的 `verify-ent` 漂移检查失败。
4. 生成代码**不可手改**，只改 `schema/`。

## 🚫 别这么干

- handler 里写业务规则、直接 `import ent` 查库。
- service 里出现 `*gin.Context` / HTTP 状态码 / `response.*`。
- 新接口手拼 `map[string]any` 当响应而不定义 DTO（统计/SSE 等沿用同域现有写法的除外）。
- 改了 schema 不跑 `make ent` 就提交。
- handler 里硬编码 HTTP 状态码 / 把内部 err 直接返回前端：应走 service 哨兵错误（`errors.go`）+ handler 的 `handleError` 映射。⚠️ 上游账号失效用 **422 不用 401**（401 会踹掉管理员登录）。其余编码约定（slog 日志、凭证加密、ctx 透传、分页/时区）见 `airgate-core/CLAUDE.md`「后端编码约定」。

## 完成前自检

```bash
make lint && make test          # 快检
# 改过 schema：
make ent && git status          # 确认生成代码已更新并纳入提交
```
彻底一点跑 `make ci`（含 verify-ent）。详见 skill `airgate-ci-check`。
