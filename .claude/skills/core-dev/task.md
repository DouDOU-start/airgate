# 任务状态机规则

> 字段权威来源：`airgate-core/backend/ent/schema/task.go`。改 schema 后须同步本表（防漂移）。

## 状态迁移

7 个状态，`completed` / `failed` / `cancelled` 为终态，**不可回退**：

```
pending → processing → completed
                     → failed
                     → retrying → pending
                                → failed
                     → cancelling → cancelled
                                  → failed
pending → failed / cancelled
processing → cancelled
```

`status` 为 Enum，取值仅限上述 7 个（`pending/processing/retrying/completed/failed/cancelling/cancelled`），默认 `pending`。

## 字段约束

Core 只保留**通用字段**，**禁止为业务类型新增固定列**；业务扩展参数（分辨率、时长、质量等）一律放 `input` / `attributes` / `execution`。

### 身份与分类

| 字段 | 规则 |
|---|---|
| `plugin_id` | 所属插件 ID（如 `airgate-playground`），非空 |
| `task_type` | 业务类型字符串（如 `image_generation`），Core 不解释 |
| `status` | 见上「状态迁移」，Enum，Core 维护迁移合法性 |
| `stage` | 插件声明的当前阶段，**仅用于展示/调试**，Core 不据此决策 |
| `user_id` | 所属用户，正整数 |

### 业务数据（JSON，Core 不理解含义）

| 字段 | 规则 |
|---|---|
| `input` | 原始任务输入（JSONB），创建时写入 |
| `output` | 最终业务输出（JSONB），格式由插件约定 |
| `attributes` | 少量展示/筛选维度（JSONB），不放大文本 |
| `execution` | 插件内部执行状态容器（JSONB），Core 不解释；持久化上游 task id、轮询游标等 |

### 错误（结构化）

| 字段 | 规则 |
|---|---|
| `error_type` | 错误大类（默认空串） |
| `error_code` | 错误码（默认空串） |
| `error_message` | 错误详情（默认空串） |

> 进入 `failed` 时由插件/Core 填写，供前端结构化展示，勿把错误塞进 `output`。

### 进度 / 重试 / 优先级

| 字段 | 规则 |
|---|---|
| `progress` | 进度 0–100（默认 0，Core 钳制范围） |
| `priority` | 越高越优先处理（默认 0） |
| `attempts` | 已尝试次数（默认 0） |
| `max_attempts` | 最大尝试次数（默认 3）；`retrying` 路径据此判断是否转 `failed` |

### 关联 / 幂等 / 对外 ID

| 字段 | 规则 |
|---|---|
| `usage_id` | 关联 `usage_log.id`，**不复制计费字段**；完成后的模型/计量/费用事实以 usage 为准 |
| `public_task_id` | 对外暴露的任务 ID，**全局唯一**（唯一索引）；**不参与幂等语义** |
| `idempotency_key` | 幂等键，唯一性作用域为 `(plugin_id, user_id, task_type, idempotency_key)`（唯一索引） |

### 时间戳

| 字段 | 规则 |
|---|---|
| `created_at` | 创建时间，不可变 |
| `updated_at` | 更新时间，自动刷新 |
| `started_at` | 进入 `processing` 时间（可空） |
| `completed_at` | 进入终态时间（可空） |
| `cancel_requested_at` | 收到取消请求时间（可空）；据此进入 `cancelling` |
| `expires_at` | 过期时间（可空）；超期任务由清理逻辑处理 |

### 索引（`task.go` Indexes）

- `(plugin_id, status, created_at)`、`(user_id, created_at)`、`(status, created_at)` — 列表/调度查询
- `public_task_id` **唯一**
- `(plugin_id, user_id, task_type, idempotency_key)` **唯一** — 幂等去重

## 同步与异步

Core 不区分，插件负责适配：
- **同步**：插件调上游 → 立即 `completed`
- **异步**：上游 task id 写入 `execution.upstream.task_id` → 后台轮询
- **不可查询上游**：阻塞等待或拒绝后台任务，**不允许只靠内存**

Usage 是完成后的唯一事实来源，任务只经 `usage_id` 关联，只记一个实际模型。
