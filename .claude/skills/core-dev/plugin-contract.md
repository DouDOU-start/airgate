# 插件契约规则

## Metadata 约定表

**新增约定键须同步登记本表**；未声明回退 Core 历史默认。按来源分三类登记。

### A. 插件静态声明键（ModelInfo / RouteDefinition / PluginInfo）

调度/路由/错误格式据此分支，未声明回退 Core 默认。

| 键 | 位置 | 含义 |
|---|---|---|
| `metadata_only` | RouteDefinition | 只读元信息路由，跳过调度/计费 |
| `error_format` | RouteDefinition（优先）/ PluginInfo | 错误体格式：`openai`（默认）/ `anthropic` |
| `family` | ModelInfo | 限流冷却的模型家族键 |
| `scheduling_model` | ModelInfo | 调度等价模型映射（精确 ID） |
| `scheduling_model_map` | RouteDefinition | 请求模型 → 调度模型前缀映射表（JSON） |
| `account.oauth_plans` | PluginInfo | OAuth 套餐识别规则（JSON） |

### B. 用量快照键（`ForwardOutcome.Usage.Metadata`，插件每次转发上报）

Core 经 `internal/plugin/usage_adapter.go` / `outcome.go` 读取，落入 `usage_log` 展示列。属插件私有维度，Core 不据此计费，仅记录展示。

| 键 | 消费位置 | 含义 |
|---|---|---|
| `image_size` | usage_adapter.go:96,112,145（亦认 `size`/`resolution`） | 图像分辨率 |
| `image_tier` | usage_adapter.go:100,115,142（亦认 `tier`/`resolution_tier`） | 图像档位 |
| `service_tier` | usage_adapter.go:92,109（亦认 `tier`） | 服务档位（如 fast/flex） |
| `reasoning_effort` | outcome.go:328（`normalizeReasoningEffort` 归一化） | 推理强度 |

### C. 计费注解键（`UsageLog.Metadata`，**Core 计费时写入**，非插件声明）

由 `internal/billing/recorder.go` 在记账时写入，供前端展示/审计，插件勿自行设置。

| 键 | 写入位置 | 含义 |
|---|---|---|
| `billing_mode` | recorder.go:355,361 | 计费模式：`fixed_image_price` / `image_token` |
| `fixed_unit_price` | recorder.go:357 | 图像固定单价（仅 `fixed_image_price` 模式） |
| `fixed_unit` | recorder.go:358 | 固定计费单位标签（如 `USD/image`） |

## Host.Invoke 方法

| 分组 | method |
|---|---|
| 调度 | `scheduler.select_account`、`scheduler.report_account_result` |
| 探测 | `probe.forward` |
| 转发 | `gateway.forward` |
| 元数据 | `groups.list`、`platforms.list`、`models.list` |
| 用户 | `users.get`、`users.update_balance`（`idempotency_key` 必填） |
| 资产 | `assets.store`、`assets.store_url`、`assets.get_url`、`assets.get_bytes`、`assets.delete` |
| 任务 | `tasks.create`、`tasks.update`、`tasks.get`、`tasks.list`、`tasks.delete` |

- 新增 method 须属**跨插件平台能力**，单插件业务勿入
- `Init()` 阶段 capability 未绑定，不能调 Host RPC

## Capability

- 插件在 `PluginInfo.Capabilities` 声明，`CapabilityForHostMethod("tasks.create")` → `host.invoke.tasks.create`
- 授权由 Core 方法注册表启动时执行，SDK 仅自检
- `plugin.yaml` 由 `make manifest` 生成，**禁止手改**

## 技术债（勿加深）

### 越界判定标准（core 侧 provider/模型/插件名硬编码）

铁律：**core 禁止按 provider 名 / 模型字符串 / 插件 id 做行为分支**。协议与平台差异一律经插件 Metadata 约定键声明（见上「Metadata 约定表」），core 只保留与具体厂商无关的「默认兜底」。一处硬编码是否越界，按此机械判定：

- **OK-兜底**（允许）：仅当插件未声明对应 Metadata 键时才生效，且不写死某具体厂商的产品语义。
  - 例：`scheduler/family.go` 的 `gpt-image` 家族折叠（`ModelInfo.Metadata["family"]` 优先）；`plugin/quota.go` 的 `metadata_only` 路径回退（`Metadata["metadata_only"]` 优先）；`plugin/error_format.go` 的格式常量（由 `Metadata["error_format"]` 选择，default OpenAI）。
- **越界**（禁止新增，已登记项勿加深）：core 用 `platform=="openai"`、`HasPrefix(model,"claude-")`、插件名常量等做**调度 / 计费 / 路由 / 资产**的行为分支，且无 Metadata 间接层。

### 已登记的硬编码越界（勿加深，按排期治理）

经审计确认的现存越界（与「已收口」相反，core 仍假定「OpenAI 是唯一图像 / 协议翻译 provider」）。**新增图像或协议翻译插件前须先把对应项归位，否则功能不正确**：

| 位置 | 越界 | 应归位 |
|---|---|---|
| `internal/plugin/scheduling_model.go:31,34,87-118,137` | `platform=="openai"` + `/v1/messages` 特判 → `openAIAnthropicSchedulingModels` 按 `claude-*` 前缀映射到 gpt/kiro 模型；strip `openai/`/`oai/` 前缀 | 全量迁到插件 `RouteDefinition.Metadata["scheduling_model_map"]`，删除 core 内 claude/openai 翻译分支 |
| `internal/routing/selector.go:113-117` | 仅 `platform=="openai"` 需 `image_enabled` 门控，其余平台默认放行 | 改为按 `ModelInfo.Capabilities` / Metadata 声明的图像能力门控 |
| `internal/billing/image_pricing.go:9,35` | 仅 `pluginName=="openai"`（`OpenAIPluginSettingsKey`）可声明图片档位定价 | 改为 capability/Metadata 门控，任意图像 provider 平等 |
| `internal/plugin/image_pricing.go:44-46` | `gpt-image`/`dall-e` 前缀判定图像模型（capability 检查在前，前缀为 fallback） | 前缀清单移出 core，由插件模型目录声明 image capability |
| `internal/plugin/asset_cleanup.go:20` | `generatedTaskExecutorPluginID = "gateway-openai"` 写死图像任务归属插件 | 改为 Metadata 声明资产清理归属，允许多图像 provider |

> 治理顺序：先归位「图像 provider 假定」一类（selector/billing/image_pricing/asset_cleanup），再拆 `scheduling_model.go` 的协议翻译映射（最大、牵涉 openai 插件 Metadata 补全）。

### 仍开放的结构性债务
- **HostService 过宽**：单一 Invoke 通道暴露 19 个 method，目标为版本化 capability 分组
- **Manifest 无 v2**：无 `requires.host` / `provides` 声明
- **网关插件混合职责**：gateway + provider + UI 同仓，新代码按职责归位、勿加深
- **Playground 兼做协议转发**：目标为 UI-only

新代码按边界归位即可，无需顺手重构。

---

## 扩展插件代理

三种入口类型，按路径前缀自动判定：

| 路径 | 入口类型 | 鉴权 | Header |
|---|---|---|---|
| `/api/v1/ext/:pluginName/*` | `admin` | JWT | `X-Airgate-Entry: admin` |
| `/api/v1/ext-user/:pluginName/*` | `user` | JWT | `X-Airgate-Entry: user` |
| `/api/v1/payment-callback/:pluginName/*` | `callback` | **无** | `X-Airgate-Entry: callback` |

- `HandleNamed()` 用于固定插件路由（如 `/status` → `airgate-health`，entry=`public`）
- 请求体上限：100 MB

## 后台任务约束

- 仅 extension 插件可声明后台任务
- **最小间隔 30 秒**（防 DoS），低于此值被强制提升
- 每个任务独立 goroutine，共享 cancel context
- 首次立即执行（不等 ticker），后续按间隔
- 单次执行超时：5 分钟

## 资产存储规则

5 种资产用途（enum，插件只能使用这些值）：

| 用途 | 含义 |
|---|---|
| `Chat` | 对话附件 |
| `Upload` | 用户上传 |
| `Generated` | 生成产物 |
| `TaskInput` | 任务输入 |
| `Temp` | 临时文件 |

- 路径格式：`{purpose}/{plugin_id}/{date}/{uuid}.ext`
- 下载上限：50 MB，超时 60s
- 本地默认目录：`data/assets`；可配 S3/R2（Settings 表 `storage` 分组）
- 迁移任务每小时运行，**不删除本地文件**（非破坏性）
- 迁移跳过 `.tmp` 文件和缩略图变体

## 插件运行时约束

- gRPC 消息上限：64 MB（收发）
- 插件启动超时：15 秒（Init + handshake）
- `Init()` 阶段 capability 未绑定，**不能调 Host RPC**
- 插件错误不能控制响应格式——只返回 `ForwardOutcome`，Core 负责协议格式化
