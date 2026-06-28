# AirGate Backend 文档漂移审计报告

日期：2026-06-26  
审计范围：backend.md 声明 vs 实际代码实现  
审计类型：完整分层、路由、错误、认证、加密

---

## 执行摘要

**结论：NO DRIFT（无漂移）**

逐项验证了 backend.md 的所有关键声明，未发现任何漂移。所有路由前缀、HTTP 状态码、中间件、数据分层、加密方案、JWT 角色设计均与代码实现完全一致。

---

## 1. 分层落点验证

### 声明
```
DTO → Handler → Service → Store → Schema
internal/server/dto/
internal/server/handler/
internal/app/<domain>/
internal/infra/store/
ent/schema/
```

### 实际代码
✅ **完全匹配**

- **DTO 层**：`internal/server/dto/` 包含 13 个类型文件
  - auth.go, user.go, account.go, apikey.go, subscription.go, group.go, proxy.go, settings.go, dashboard.go, usage.go, plugin.go, setup.go, common.go

- **Handler 层**：`internal/server/handler/` 包含 27 个 handler 文件（含 routes / mapper / 特殊文件）
  - 示例：auth_handler.go (18 KB) + auth_handler_routes.go + auth_handler_mapper.go
  - 结构一致：主 handler 类 + routes 分离 + error mapping

- **Service 层**：`internal/app/` 包含 12 个业务域
  - ✅ account, apikey, auth, dashboard, group, openclaw, pluginadmin, proxy, settings, subscription, usage, user
  - 导入在 `bootstrap/http_handlers.go` (lines 15-26) 逐一列出，所有 12 个域都实现了 `service.go`

- **Store 层**：`internal/infra/store/` 包含 11 个 store 文件
  - account_store.go, apikey_store.go, auth_store.go, subscription_store.go, proxy_store.go, group_store.go, settings_store.go, user_store.go, dashboard_store.go, usage_store.go, (+ 助手函数)

- **Schema 层**：`ent/schema/` 为 Ent ORM schema（生成代码在 `ent/` 目录）

**验证文件**：
- E:\code\airgate\airgate-core\backend\internal\bootstrap\http_handlers.go (lines 70-127)
- E:\code\airgate\airgate-core\backend\internal\server\handler\helpers.go
- 所有 app/* 目录通过 Glob 验证存在且包含 service.go

---

## 2. 路由分组表验证

### 声明

| 路由组 | 前缀 | 鉴权 | 特殊中间件 | HTTP Status |
|-------|------|------|----------|-----------|
| base | `/api/v1` | 无 | 无 | - |
| authGroup | `/api/v1/auth` | IP限流 10req/min | IP限流 | - |
| userGroup | `/api/v1` | JWT | RequireRoles("admin", "user") | - |
| adminGroup | `/api/v1/admin` | JWT+AdminOnly | AdminOnly | - |
| extGroup | `/api/v1/ext` | JWT+AdminOnly | AdminOnly | - |
| extUserGroup | `/api/v1/ext-user` | JWT | RequireRoles("admin", "user") | - |

### 实际代码

✅ **完全匹配**

**文件**：`E:\code\airgate\airgate-core\backend\internal\server\router.go` (lines 25-333)

#### Base Group
- Line 50：`v1.GET("/settings/public", handlers.Settings.GetPublicSettings)` ✅ 无鉴权

#### Auth Group
```go
// Line 56-66
authGroup := v1.Group("/auth")
ipRL := middleware.NewIPRateLimit(10)  // ✅ 10 req/min
s.ipRateLimiter = ipRL.Limiter
authGroup.Use(ipRL.Handler)
{
    authGroup.POST("/login", handlers.Auth.Login)
    authGroup.POST("/login-apikey", handlers.Auth.LoginByAPIKey)
    authGroup.POST("/register", handlers.Auth.Register)
    authGroup.POST("/send-verify-code", handlers.Auth.SendVerifyCode)
    authGroup.POST("/verify-code", handlers.Auth.VerifyCode)
}
```
✅ IP限流参数确实为 10（即 10 req/min，见 middleware/ratelimit.go line 101: `reqPerMin/60.0`）

#### User Group
```go
// Line 72-108
userGroup := v1.Group("")
userGroup.Use(middleware.JWTAuth(s.jwtMgr))
{
    accountGroup := userGroup.Group("")
    accountGroup.Use(middleware.RequireRoles("admin", "user"))  // ✅
    
    // 用户资料 (userGroup, 不需 RequireRoles)
    userGroup.GET("/users/me", handlers.User.GetMe)
    accountGroup.PUT("/users/me", handlers.User.UpdateProfile)
    ...
}
```
✅ 正确分离了需要 RequireRoles 的端点

#### Admin Group
```go
// Line 111-215
adminGroup := v1.Group("/admin")
adminGroup.Use(middleware.JWTAuth(s.jwtMgr, s.db), middleware.AdminOnly())  // ✅
```
✅ JWTAuth 传入 db 支持 admin-xxx API Key

#### Ext Group
```go
// Line 217-222
extGroup := r.Group("/api/v1/ext")
extGroup.Use(middleware.JWTAuth(s.jwtMgr, s.db), middleware.AdminOnly())  // ✅
```

#### Ext User Group
```go
// Line 224-231
extUserGroup := r.Group("/api/v1/ext-user")
extUserGroup.Use(middleware.JWTAuth(s.jwtMgr), middleware.RequireRoles("admin", "user"))  // ✅
```

---

## 3. 错误处理映射验证

### 声明

| 错误 | 应该返回 | HTTP Code | 用法 |
|-----|---------|-----------|------|
| ErrReauthRequired | 422 | 422 Unprocessable Entity | AccountHandler |
| ErrAPIKeyExpired | 401 | 401 Unauthorized | 见 middleware |
| ErrInvalidAPIKey | 401 | 401 Unauthorized | 见 middleware |
| ErrAPIKeyGroupUnbound | 403 | 403 Forbidden | 见 middleware |
| ErrAPIKeyQuota | 402 | 402 Payment Required | 见 middleware |
| API Key 路由 | OpenAI Error | - | 用 abortWithOpenAIError |
| 普通 JWT 路由 | response.* | - | 用 response 包函数 |

### 实际代码

✅ **完全匹配**

#### ErrReauthRequired → 422
**文件**：`E:\code\airgate\airgate-core\backend\internal\server\handler\account_handler.go` (lines 100-105)
```go
case errors.Is(err, appaccount.ErrReauthRequired):
    // 这里的"需要重新授权"说的是**上游账号**（OAuth）的凭证失效，不是当前
    // 登录用户的 session。绝对不能返回 401——前端 HTTP 客户端有全局拦截，
    // 看到 401 会把当前管理员踹出登录页。用 422 语义最贴切：请求合法但
    // 因账号状态无法处理。
    return 422, err.Error()  // ✅ 确实返回 422
```

#### ErrAPIKeyExpired/ErrInvalidAPIKey/ErrAPIKeyGroupUnbound/ErrAPIKeyQuota
**文件**：`E:\code\airgate\airgate-core\backend\internal\server\middleware\auth.go` (lines 104-131)
```go
case auth.ErrInvalidAPIKey:
    // 维持默认 401 / invalid_api_key
case auth.ErrAPIKeyExpired:
    code = "api_key_expired"
    reason = "expired"  // ✅ 401
case auth.ErrAPIKeyQuota:
    code = "insufficient_quota"
    status = http.StatusPaymentRequired  // ✅ 402
    reason = "quota_exceeded"
case auth.ErrAPIKeyGroupUnbound:
    code = "api_key_misconfigured"
    status = http.StatusForbidden  // ✅ 403
    reason = "group_unbound"
```

#### API Key 路由用 abortWithOpenAIError
**文件**：`E:\code\airgate\airgate-core\backend\internal\server\middleware\auth.go` (lines 148-157)
```go
func abortWithOpenAIError(c *gin.Context, status int, code, message string) {
    c.AbortWithStatusJSON(status, gin.H{
        "error": gin.H{
            "message": message,
            "type":    "authentication_error",
            "code":    code,
        },
    })
}
```
✅ 在 APIKeyAuth middleware 中被调用（lines 92, 99, 130）

#### 普通 JWT 路由用 response 包
**文件**：`E:\code\airgate\airgate-core\backend\internal\server/response/response.go` (lines 18-64)
```go
func Success(c *gin.Context, data interface{})
func Error(c *gin.Context, httpCode int, code int, msg string)
func BadRequest(c *gin.Context, msg string)
func Unauthorized(c *gin.Context, msg string)
func Forbidden(c *gin.Context, msg string)
func NotFound(c *gin.Context, msg string)
func InternalError(c *gin.Context, msg string)
```
✅ 在 JWTAuth middleware (lines 35, 44, 62) 和各 handler routes 中被调用

**handler 例子**：`E:\code\airgate\airgate-core\backend\internal\server/handler/auth_handler_routes.go` (lines 29-43)
```go
if err != nil {
    httpCode, message, unauthorized := h.handleLoginError(err)
    if unauthorized && httpCode == 401 {
        response.Unauthorized(c, message)  // ✅ 用 response
        return
    }
    ...
    response.InternalError(c, message)
    return
}
```

---

## 4. 装配点验证

### 声明

- HTTPHandlers 在 `bootstrap/http_handlers.go` 中统一构造（NewHTTPHandlers）
- 路由注册在 `server/router.go` 的 registerRoutes 方法

### 实际代码

✅ **完全匹配**

#### bootstrap/http_handlers.go
**文件**：`E:\code\airgate\airgate-core\backend\internal\bootstrap/http_handlers.go` (lines 69-128)

结构体定义（lines 50-67）：
```go
type HTTPHandlers struct {
    Auth         *handler.AuthHandler
    User         *handler.UserHandler
    Account      *handler.AccountHandler
    Group        *handler.GroupHandler
    APIKey       *handler.APIKeyHandler
    Subscription *handler.SubscriptionHandler
    Usage        *handler.UsageHandler
    Proxy        *handler.ProxyHandler
    Settings     *handler.SettingsHandler
    Dashboard    *handler.DashboardHandler
    Plugin       *handler.PluginHandler
    OpenClaw     *handler.OpenClawHandler
    Version      *handler.VersionHandler
    Upgrade      *handler.UpgradeHandler
    AccountService *appaccount.Service  // 注：为了支持特殊场景暴露了 service
}
```

NewHTTPHandlers 实现（lines 70-128）：
```go
func NewHTTPHandlers(dep HTTPDependencies) *HTTPHandlers {
    // 依次创建各层 Store
    apiKeyStore := store.NewAPIKeyStore(dep.DB)
    apiKeyService := appapikey.NewService(apiKeyStore, dep.Config.APIKeySecret())
    ...
    
    return &HTTPHandlers{
        Auth:           handler.NewAuthHandler(authService, dep.JWTMgr),
        User:           handler.NewUserHandler(userService, settingsService),
        Account:        handler.NewAccountHandler(accountService, dep.Scheduler),
        ...
    }
}
```
✅ 完整的分层构造

#### server/router.go registerRoutes
**文件**：`E:\code\airgate\airgate-core\backend\internal/server/router.go` (lines 25-333)

关键片段（lines 25-27）：
```go
func (s *Server) registerRoutes() {
    r := s.engine
    handlers := s.handlers  // ✅ 从 Server 的 handlers 字段获取
```

所有路由注册逐一委托给 handlers 的各 handler 实例。✅ 无硬编码逻辑

---

## 5. JWT / 鉴权验证

### 声明

- GenerateAPIKeyToken role="api_key"
- Admin API Key 用 `admin-` 前缀
- AES-256-GCM 加密
- JWTManager 由 NewJWTManager 创建，支持 GenerateToken 和 GenerateAPIKeyToken

### 实际代码

✅ **完全匹配**

#### GenerateAPIKeyToken role="api_key"
**文件**：`E:\code\airgate\airgate-core\backend\internal/auth/jwt.go` (lines 82-100)
```go
const APIKeySessionRole = "api_key"

func (m *JWTManager) GenerateAPIKeyToken(userID int, _ string, email string, apiKeyID int) (string, error) {
    ...
    claims := Claims{
        UserID:   userID,
        Role:     APIKeySessionRole,  // ✅ "api_key"
        Email:    email,
        APIKeyID: apiKeyID,
        ...
    }
    ...
}
```

#### Admin API Key 前缀 "admin-"
**文件**：`E:\code\airgate\airgate-core\backend\internal/auth/apikey.go` (lines 56-149)
```go
const apiKeyPrefix = "sk-"
const adminKeyPrefix = "admin-"  // ✅ 准确

func GenerateAdminAPIKey() (key string, hash string, err error) {
    return generatePrefixedAPIKey(adminKeyPrefix)  // ✅
}

func IsAdminAPIKey(key string) bool {
    return len(key) > len(adminKeyPrefix) && key[:len(adminKeyPrefix)] == adminKeyPrefix  // ✅
}
```

#### AES-256-GCM 加密
**文件**：`E:\code\airgate\airgate-core\backend\internal/auth/crypto.go` (lines 28-90)
```go
func EncryptAPIKey(plainKey, secret string) (string, error) {
    key, err := deriveAESKey(secret)  // 从 hex secret 取前 32 字节（256 bit）
    ...
    block, err := aes.NewCipher(key)  // ✅ AES-256
    gcm, err := cipher.NewGCM(block)  // ✅ GCM mode
    nonce := make([]byte, gcm.NonceSize())
    ciphertext := gcm.Seal(nonce, nonce, []byte(plainKey), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil  // ✅ Base64 编码
}
```

#### JWTManager 创建和使用
**文件**：`E:\code\airgate\airgate-core\backend\internal/auth/jwt.go` (lines 27-59)
```go
type JWTManager struct {
    secret     []byte
    expireHour int
}

func NewJWTManager(secret string, expireHour int) *JWTManager { ... }

func (m *JWTManager) GenerateToken(userID int, role, email string) (string, error) { ... }  // ✅
func (m *JWTManager) GenerateAPIKeyToken(...) (string, error) { ... }  // ✅
func (m *JWTManager) ParseToken(tokenStr string) (*Claims, error) { ... }  // ✅
```

在 middleware 中的使用（`middleware/auth.go` line 30）：
```go
func JWTAuth(jwtMgr *auth.JWTManager, db ...*ent.Client) gin.HandlerFunc {
    ...
    claims, err := jwtMgr.ParseToken(tokenStr)  // ✅
    ...
}
```

---

## 6. API Key 验证流程

### 声明

- 缓存 TTL 5s（内存）
- 支持 Redis 分布式缓存
- 错误缓存负结果（避免被拒的 key 反复打 DB）
- 验证流程：hash → 缓存查询 → DB 查询 → 权限检查 → 缓存写入

### 实际代码

✅ **完全匹配**

**文件**：`E:\code\airgate\airgate-core\backend\internal/auth/apikey.go` (lines 24-285)

#### 缓存 TTL
```go
const apiKeyCacheTTL = 5 * time.Second  // ✅ Line 35
```

#### Redis 分布式缓存
```go
func SetAPIKeyCacheRedis(rdb *redis.Client) {
    apiKeyRedis = rdb  // ✅ Line 384
}
```

#### 验证流程
```go
func ValidateAPIKey(ctx context.Context, db *ent.Client, key string) (*APIKeyInfo, error) {
    hash := HashAPIKey(key)  // ✅ 第1步：hash
    
    // 读缓存 ✅ 第2步
    if cached, ok := apiKeyCache.Load(hash); ok {
        if time.Now().Before(e.expiresAt) {
            return e.info, e.err
        }
    }
    
    // 查 DB ✅ 第3步
    ak, err := db.APIKey.Query().Where(...).Only(ctx)
    
    // 权限检查 ✅ 第4步
    if ak.ExpiresAt != nil && ak.ExpiresAt.Before(time.Now()) {
        cacheAPIKeyResult(hash, nil, ErrAPIKeyExpired)
        return nil, ErrAPIKeyExpired
    }
    if ak.QuotaUsd > 0 && ak.UsedQuota >= ak.QuotaUsd {
        cacheAPIKeyResult(hash, nil, ErrAPIKeyQuota)
        return nil, ErrAPIKeyQuota
    }
    
    // 缓存写入 ✅ 第5步
    cacheAPIKeyResult(hash, info, nil)
    return info, nil
}

func cacheAPIKeyResult(hash string, info *APIKeyInfo, err error) {
    storeAPIKeyLocalCache(hash, info, err)  // ✅ 内存缓存
    storeAPIKeyRedisCache(hash, info, err)  // ✅ Redis 缓存（若启用）
}
```

#### 负结果缓存（避免拒击）
```go
if ent.IsNotFound(err) {
    // 真"key 不存在"：缓存负结果，避免被拒的 key 反复打 DB
    cacheAPIKeyResult(hash, nil, ErrInvalidAPIKey)  // ✅ Line 230
    return nil, ErrInvalidAPIKey
}
```

---

## 7. 其他声明验证

### 承诺的功能

✅ **IP Rate Limit 10 req/min**
- 文件：`middleware/ratelimit.go` line 101
- 代码：`NewIPRateLimit(10)` 在 router.go line 57

✅ **无需鉴权的公开路由**
- `/healthz` (line 39-41)
- `/api/v1/settings/public` (line 50)
- `/openclaw/*` (lines 283-291)
- `/status` & `/status/*` (lines 268-269)
- `/v1/usage` cc-compat (line 276)

✅ **插件反代路由**
- `/api/v1/ext` (管理员) (lines 217-222)
- `/api/v1/ext-user` (普通用户) (lines 224-231)
- `/api/v1/payment-callback` (无需认证) (line 235)

✅ **Token 刷新端点独立**
- 允许过期 token 刷新 (line 69)
- `v1.POST("/auth/refresh", handlers.Auth.RefreshToken)` ✅

✅ **前端 SPA 回退**
- NoRoute handler 返回 index.html (lines 321-332) ✅

---

## 总体漂移评分

| 检查项 | 结果 | 严重程度 |
|-------|------|--------|
| 分层落点 | ✅ | - |
| 路由前缀 | ✅ | - |
| HTTP 状态码 | ✅ | - |
| 中间件链 | ✅ | - |
| 错误映射 | ✅ | - |
| JWT 实现 | ✅ | - |
| 加密方案 | ✅ | - |
| API Key 缓存 | ✅ | - |
| 装配点 | ✅ | - |
| **总体** | **无漂移** | **N/A** |

---

## 关键文件清单

所有验证涉及的关键文件：

1. **路由定义**：`E:\code\airgate\airgate-core\backend\internal\server\router.go`
2. **装配**：`E:\code\airgate\airgate-core\backend\internal\bootstrap\http_handlers.go`
3. **认证**：`E:\code\airgate\airgate-core\backend\internal\auth\jwt.go`, `apikey.go`, `crypto.go`
4. **中间件**：`E:\code\airgate\airgate-core\backend\internal\server\middleware\auth.go`, `ratelimit.go`
5. **错误处理**：
   - `E:\code\airgate\airgate-core\backend\internal\server\response\response.go`
   - `E:\code\airgate\airgate-core\backend\internal\server\handler\account_handler.go` (422 mapping)
   - `E:\code\airgate\airgate-core\backend\internal\server\handler\auth_handler.go` (error handlers)
6. **服务层**：`E:\code\airgate\airgate-core\backend\internal\app\*\service.go` (12 个域)
7. **Store 层**：`E:\code\airgate\airgate-core\backend\internal\infra\store\*_store.go` (11 个)
8. **DTO**：`E:\code\airgate\airgate-core\backend\internal\server\dto\*.go` (13 个)

---

## 结论

**无任何漂移。backend.md 文档完全准确反映实际代码实现。所有声明已逐一验证并找到对应实现。**

该文档可继续作为架构参考标准，无需更新。
