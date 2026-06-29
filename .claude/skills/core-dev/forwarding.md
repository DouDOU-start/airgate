# 转发、调度与计费规则

## 转发管线阶段顺序

请求必须按以下顺序通过，不可跳过或调换：

1. **余额预检** — 不扣款，只挡余额不足
2. **只读元信息快车道** — 插件声明 `Metadata["metadata_only"]` 的路由跳过后续全部阶段，由插件本地合成响应
3. **客户端闸门** — API Key 级 RPM/并发限制
4. **failover 循环**（最多 3 次）：选号 → 账号并发槽（排队上限 60s）→ `Plugin.Forward()` → 按 `ForwardOutcome.Kind` 决策
5. **计费 + 记录** — 三管道计费 + 写 `usage_log`

---

## ForwardOutcome 判决

`Kind` 是插件的**唯一裁决依据**，Core 据此决策：

| Kind | 含义 | Failover | 账号责任 | 账号状态变更 |
|---|---|---|---|---|
| `OutcomeSuccess` | 2xx | 否 | 否 | → Active（Disabled 除外，需人工） |
| `OutcomeClientError` | 4xx 客户端错误 | 否 | 否 | 不变 |
| `OutcomeAccountRateLimited` | 429 限流 | **是** | **是** | → RateLimited + state_until |
| `OutcomeAccountDead` | 401/403 凭证失效 | **是** | **是** | → Disabled（池账号 403 除外） |
| `OutcomeUpstreamTransient` | 5xx/网络抖动 | **是** | 否 | 不变（池账号 → Degraded 60s） |
| `OutcomeStreamAborted` | 流中途断 | 否 | 否 | 不变 |
| `OutcomeUnknown` | 零值 | 否 | 否 | 不变 |

- `OutcomeSuccess` 时 `Usage` **必填**，否则无法计费
- `UpdatedCredentials` 非空时 Core 回写凭证（OAuth token 轮换）

---

## 账号状态机

4 个状态：

| 状态 | 含义 | 可调度性 |
|---|---|---|
| `Active` | 健康可用 | Normal |
| `RateLimited` | 上游限流中，带 `state_until` 超时 | state_until 未过期 → NotSchedulable；已过期 → Normal（惰性恢复） |
| `Degraded` | 池账号软降级，60s 超时 | 超时内 → StickyOnly（优先用其他账号）；已过期 → Normal |
| `Disabled` | 凭证失效，需人工 | NotSchedulable（**受保护：成功也不自动恢复**） |

状态迁移触发：

| 触发 | Active → | RateLimited → | Degraded → | Disabled → |
|---|---|---|---|---|
| Success | 保持 | → Active | → Active | **保持**（需人工） |
| RateLimited | → RateLimited | 更新 state_until | → RateLimited | → RateLimited |
| Dead（非池/池 401） | → Disabled | → Disabled | → Disabled | 保持 |
| Dead（池 403） | 不变 | 不变 | 不变 | 不变 |
| UpstreamTransient（非池） | 不变 | 不变 | 不变 | 不变 |
| UpstreamTransient（池） | → Degraded 60s | → Degraded 60s | 更新 state_until | 不变 |

### 冷却时长

| 类型 | 最小 | 默认 | 最大 | 来源 |
|---|---|---|---|---|
| RateLimited | 200ms | 60s | 7 天 | `ForwardOutcome.RetryAfter` |
| Degraded（仅池账号） | — | 60s | 10 分钟 | 硬编码 |
| FamilyCooldown | 1ms | 按 RetryAfter | Redis TTL | RetryAfter |

---

## 调度选号

选号管线顺序：

1. **模型路由** — 按 `group.ModelRouting` 规则过滤（精确/前缀匹配）
2. **状态过滤** — 综合以下维度取**最严**可调度性：
   - 账号状态机（Active/RateLimited/Degraded/Disabled）
   - FamilyCooldown（按 `(account, model_family)` 维度）
   - 并发负载（80% → StickyOnly，100% → NotSchedulable）
   - 滑动窗口成本（80% → StickyOnly，≥上限+reserve → NotSchedulable）
   - RPM（80% → StickyOnly，100% → NotSchedulable）
   - 会话数（达上限 → StickyOnly）
3. **Sticky 会话** — 有 sessionID 且 Redis 中存在绑定 → 优先使用绑定账号（含 StickyOnly 候选）
4. **负载均衡** — 在最高优先级账号中：`score = (1 - load_ratio) × 100 + LRU_minutes`，从 top-32 中随机选取

### 可调度性三级

| 级别 | 含义 |
|---|---|
| Normal | 完全可调度 |
| StickyOnly | 仅允许 sticky 会话复用，新请求优先其他账号 |
| NotSchedulable | 完全跳过 |

**合并规则**：多个维度取最严（NotSchedulable > StickyOnly > Normal）。无 Normal 候选但有 StickyOnly 时，降级使用 StickyOnly。

### FamilyCooldown

- 限流冷却按 **(account, model_family)** 维度，非整账号
- Redis key `family-cooldown:v1:{accountID}:{family}`，TTL = 冷却剩余时间
- 家族判定优先级：`ModelInfo.Metadata["family"]` → 硬编码（`gpt-image-*` → `gpt-image`）→ 模型名本身
- 某模型限流不牵连同账号其他家族的流量

### RPM

- Redis STRING，分钟粒度 key `rpm:{accountID}:{minute_unix}`，TTL 120s
- **非对称回退**：调度阶段预占 RPM，执行失败立即回退（Decrement）；仅成功后 `Apply` 才保留
- 阈值：≥100% → NotSchedulable，≥80% → StickyOnly

### 并发槽

- Redis ZSET `concurrency:v2:{accountID}`，member=requestID，score=时间戳
- 默认上限 `DefaultAccountMaxConcurrency = 10`，slot TTL 5 分钟（防泄漏）
- 三级独立：账号级、API Key 级、用户级
- 阈值：≥100% → NotSchedulable，≥80% → StickyOnly

### 滑动窗口成本

- 5 小时窗口（可配 `account.extra["window_hours"]`），查 `usage_log` 的 `actual_cost` 总和
- Redis 缓存 30s，增量更新（Lua 脚本原子操作）
- 阈值：≥80% → StickyOnly，≥上限+sticky_reserve（默认 10.0）→ NotSchedulable

### Sticky 会话

- Redis key `sticky:{userID}:{platform}:{sessionID}` → accountID，TTL 30 分钟
- 选号时优先匹配已绑定账号（含 StickyOnly 候选）
- 会话数上限：`account.extra["max_sessions"]`，达上限 → StickyOnly

### 消息锁（串行化）

- 同一账号的用户消息请求串行化（防并发状态冲突）
- 可重入：同一 requestID 重试不阻塞
- 最大等待者：`account.extra["msg_lock_max_waiters"]`（默认 8），超出 → failover
- 等待超时：`account.extra["msg_lock_wait_seconds"]`（默认 3s），超时 → failover
- RPM 自适应延迟：RPM <50% → 200ms，50-80% → 线性插值至 2000ms，≥80% → 2000ms，±15% 抖动

---

## Failover

### 决策树

```
流已写入客户端 → 不 failover（不可逆）
插件 panic → failover
Kind.ShouldFailover():
  RateLimited / Dead / UpstreamTransient → failover
  ClientError / Success / Unknown / StreamAborted → 不 failover
```

### 迭代逻辑

- 每条路由最多 3 次尝试
- 账号责任（RateLimited/Dead）→ hard_exclude（本次请求不再尝试）
- 非账号责任（UpstreamTransient）→ soft_exclude（可重试）
- 无可用账号时：指数退避轮询（初始 3-200ms，上限 2s）

### 最终失败响应

按优先级选择：
1. 有 RateLimited → **429** + 最早的 RetryAfter（默认 1s）
2. 有 Timeout → **504**
3. 有上游错误 → **502**
4. 账号/可用性问题 → **503**

---

## Middleware

- `OnForwardBegin` 按 Priority 升序进，`OnForwardEnd` 降序出（LIFO 栈语义）
- Begin 返回 `DecisionDeny` 时直接拒绝请求
- `OnForwardBegin` **只在首次 attempt** 调用（避免 failover 污染审计计数）

## ProbeForward 隔离

探测转发跳过 `usage_log`、扣款、RPM/并发限流，**但仍** `ReportResult` 更新状态机——使降级账号有机会被探测恢复。

---

## 计费三管道

三条**独立**管道，互不影响：

| 管道 | 公式 | 落点 |
|---|---|---|
| `actual_cost` | `total_cost × billing_rate` | 扣 `User.balance`（平台真实成本） |
| `billed_cost` | `total_cost × sell_rate` | 累加 `APIKey.used_quota`（终端客户可见；`sell_rate=0` 回退 `actual_cost`） |
| `account_cost` | `total_cost × account_rate` | 写 `usage_log`（仅统计，不影响余额） |

图片固定价覆盖：`ImageBillingCostOverride` / `ImageBilledCostOverride` / `OutputBillingCostOverride` 绕过倍率按固定单价计。

### 图片定价规则

- 图片分辨率/质量 → tier 映射（`billing.ImageTierForSize`），SDK `Usage` 中 `ImageTier` 优先，fallback 到 `ImageSize`
- `gpt-image-*` / `dall-e-*` 模型：固定价**替换**整个 total_cost（非累加）
- 其他模型：固定价**累加**到 total_cost
- 图片数量 fallback：`Usage` 直接字段 → metadata `image_count` / `count` / `quantity`

### Usage 适配规则

SDK `Usage` → 计费 `CalculateInput` 的转换（`usage_adapter.go`）：
- Token 类型：input / output / cached_input / cache_creation（5m/1h 变体）/ reasoning_output
- 同义归一化：`prompt_tokens` → `input_tokens`，`completion_tokens` → `output_tokens`
- 成本来源：`CostDetails[].AccountCost` + `Metadata["unit_price"]`
- 多源 fallback 链：直接字段 → attributes 数组 → metadata map

---

## 错误格式协议

转发层错误**必须经 `protocolError()` 输出**，不用 `response.*`。

- 格式选择：`RouteDefinition.Metadata["error_format"]`（路由级优先）→ `PluginInfo.Metadata["error_format"]` → 默认 `openai`
- `setRequestErrorFormat()` 在请求入口写入 Gin context，后续所有 `protocolError()` 按此格式
- Anthropic 格式映射 7 种 type：`invalid_request_error` / `authentication_error` / `permission_error` / `not_found_error` / `rate_limit_error` / `api_error` / `overloaded_error`（其中 `overloaded_error` 对应 HTTP 503）
- 429 响应同时写 `Retry-After`（秒）和 `Retry-After-Ms`（毫秒）两个 header

---

## 客户端配额（两层）

用户级和 API Key 级并发独立管控：

- 请求进入时同时获取两个 slot，任一层满 → 429
- slot ID 跨 failover 复用（不随 attempt 变化）
- 释放顺序：API Key 先、用户后（LIFO）
- 释放 context 用 `context.Background()`（请求取消后仍能清理）

---

## 请求头转发规则

Core 向插件注入的 header：

| Header | 内容 |
|---|---|
| `X-Airgate-User-ID` | 当前用户 ID |
| `X-Airgate-API-Key-ID` | 当前 API Key ID |
| `X-Airgate-Group-ID` | 当前分组 ID |
| `X-Airgate-Service-Tier` | 服务等级 |
| `X-Airgate-Force-Instructions` | 强制指令 |
| `X-Airgate-Account-Image-Protocols` | 账号支持的图片协议列表 |
| `X-Airgate-Plugin-{plugin}-{key}` | 分组级插件设置（canonical header case） |
| `X-Forwarded-Path` / `Method` / `Query` | 原始请求路径/方法/查询参数 |

**安全过滤**：`image_price_1k` / `image_price_2k` / `image_price_4k` 等计费设置**不转发**给插件。

---

## 请求解析规则

- 请求体解析提取：`model` / `stream` / `sessionID`（from `metadata.user_id`）/ `reasoning_effort`
- **Reasoning effort 归一化**：候选来源 `fields.ReasoningEffort` → `fields.Reasoning.Effort` → `fields.OutputConfig.Effort`；无显式 effort 但有 `OutputConfig` 或 `Thinking` → 默认 `high`；统一为 `low/medium/high/xhigh/max`
- **图片请求判定**：API 路径匹配 OR image-only 模型 OR 强制选择 `image_generation` 工具 → `NeedsImage` → 影响账号能力过滤和超时窗口
