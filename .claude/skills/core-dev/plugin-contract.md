# 插件契约规则

## Metadata 约定表

**新增约定键须同步登记本表**；未声明回退 Core 历史默认。

| 键 | 位置 | 含义 |
|---|---|---|
| `metadata_only` | RouteDefinition | 只读元信息路由，跳过调度/计费 |
| `error_format` | RouteDefinition（优先）/ PluginInfo | 错误体格式：`openai`（默认）/ `anthropic` |
| `family` | ModelInfo | 限流冷却的模型家族键 |
| `scheduling_model` | ModelInfo | 调度等价模型映射（精确 ID） |
| `scheduling_model_map` | RouteDefinition | 请求模型 → 调度模型前缀映射表（JSON） |
| `account.oauth_plans` | PluginInfo | OAuth 套餐识别规则（JSON） |

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

一类热点（协议/产品硬编码）已全部收口为 Metadata 声明 + Core 默认兜底，**勿再扩展硬编码**。

仍开放的结构性债务：
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
