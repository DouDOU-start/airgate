# 后端接口 / 领域逻辑 / Ent 变更

airgate-core 后端为严格的端口-适配器分层。按以下落点与顺序开发，勿自创结构。

## 第 0 步：参照现有实现

开发前完整阅读一个同形态领域作模板，首选 `account`：
- `backend/internal/server/dto/account.go`
- `backend/internal/server/handler/account_handler.go`（+ `_mapper.go` / `_routes.go`）
- `backend/internal/app/account/service.go`（+ `types.go` / `errors.go`）
- `backend/internal/infra/store/account_store.go`

## 分层落点

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
3. **Store**：`infra/store/<domain>_store.go` 实现新增 `Repository` 方法。
4. **DTO**：`server/dto/<domain>.go` 定义请求/响应结构。
5. **Handler**：`server/handler/` 编写端点——`c.ShouldBind*` 绑定 → `h.service.X(ctx, Input{...})` → `toXResp` 映射（置于 `_mapper.go`）→ `response.Success/Error`，错误经 `h.handleError(...)`。
6. **装配（两处）**：
   - `internal/bootstrap/http_handlers.go`：`NewHTTPHandlers` 内按 `store → service → handler` 构造，挂载至 `HTTPHandlers`。
   - `internal/server/router.go`：`registerRoutes()` 选对分组（`v1` / `userGroup` / `adminGroup` / `extGroup`）注册路由。
7. **测试**：同包 `_test.go`，表驱动；service 层逻辑必测。

## Ent 变更子流程

1. 编辑 `ent/schema/<entity>.go`（字段/边/索引/mixin）。
2. `airgate-core/` 下运行 **`make ent`** 重新生成。
3. 提交生成代码（`ent/` 生成部分）；漏提交将致 `make ci` 的 `verify-ent` 失败。
4. 生成代码不可手改，仅改 `schema/`。

## 路由分组与中间件

全局中间件栈：CORS → Recovery → RequestLogger → I18n → 业务。

| 分组 | 路径前缀 | 鉴权 | 用途 |
|---|---|---|---|
| base | `/api/v1` | 无 | 公开端点（`/healthz`） |
| authGroup | `/api/v1` | IP 限流（10 req/min） | 登录/注册 |
| userGroup | `/api/v1` | JWT | 用户路由 |
| adminGroup | `/api/v1` | JWT + AdminOnly | 管理后台 |
| extGroup | `/api/v1/ext` | JWT | 扩展插件代理（admin 入口） |

新路由须选对分组，否则鉴权层级不匹配。

## 错误处理规则

### handler `handleError` 映射

service 返回哨兵错误 → handler `handleError` 映射为 HTTP code + 对外消息：

| 哨兵错误 | HTTP code | 说明 |
|---|---|---|
| `ErrXxxNotFound` | 404 | 资源不存在 |
| `ErrReauthRequired` | **422** | 上游 OAuth 失效等业务不可处理 |
| `ErrAPIKeyExpired` / `ErrInvalidAPIKey` | 401 | 凭证失效 |
| `ErrAPIKeyGroupUnbound` | 403 | 权限不足 |
| `ErrAPIKeyQuota` | 402 | 额度不足 |
| 未匹配 | 500 | 日志记录内部错误，对外返回通用消息 |

**422 vs 401**：上游账号 OAuth 失效用 422，**禁止返回 401**——前端有 401 全局拦截会登出当前管理员。

### response 包

统一出口，handler 必须使用：
- `response.Success(c, data)` — 200
- `response.Error(c, httpCode, code, msg)` — 自定义错误
- `response.BadRequest / NotFound / Forbidden / Unauthorized / InternalError` — 便捷函数
- `response.PagedData(list, total, page, pageSize)` — 分页

### API Key 路由错误格式

API Key 路由（`/api/v1/<path>` 携带 API Key）使用 `abortWithOpenAIError()` 返回 OpenAI 兼容错误格式，**不用** `response.*`。

## 鉴权规则

### JWT 令牌

- `GenerateToken()` — 管理员/用户登录令牌
- `GenerateAPIKeyToken()` — API Key 登录令牌，`role="api_key"`，**不继承 admin/user 权限**
- `ParseTokenForRefresh()` 允许 token 过期后 grace period 内刷新
- `RefreshToken()` 按 `claims.APIKeyID > 0` 分支到正确的刷新逻辑

### Admin API Key

- 特殊令牌，`admin-` 前缀，验证走 Settings 表（非 APIKey 实体）
- 仅用于管理后台 API，不能访问用户路由
- `auth.IsAdminAPIKey(tokenStr)` 检测前缀

### 凭证加密

- AES-256-GCM 加密，`APIKeySecret` 须 32+ 字节 hex
- `EncryptAPIKey` / `DecryptAPIKey`，nonce 随密文一起存储
- 凭证不明文落库、不写日志

## 禁止

- handler 编写业务规则、或直接 `import ent` 查库
- service 出现 `*gin.Context` / HTTP 状态码 / `response.*`
- 新接口手拼 `map[string]any` 作响应而不定义 DTO（统计/SSE 等沿用同域现有写法除外）
- 改 schema 后未 `make ent` 即提交
- 上游账号失效返回 401（应用 422）
- API Key 路由使用 `response.*` 而非 `abortWithOpenAIError()`
