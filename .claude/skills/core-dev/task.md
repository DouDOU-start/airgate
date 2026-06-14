# 任务状态机规则

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

## 字段约束

Core 只保留通用字段，**禁止为业务类型新增固定列**：

| 字段 | 规则 |
|---|---|
| `task_type` | 业务类型字符串，Core 不解释 |
| `input` | 原始任务输入（JSON） |
| `output` | 最终业务输出（JSON），格式由插件约定 |
| `attributes` | 少量展示/筛选维度（JSON），不放大文本 |
| `execution` | 执行状态容器（JSON），Core 不解释；插件持久化上游 task id、轮询游标 |
| `usage_id` | 关联 `usage_log.id`，**不复制计费字段** |

扩展参数（分辨率、时长、质量）一律放 `input` / `attributes` / `execution`。

## 同步与异步

Core 不区分，插件负责适配：
- **同步**：插件调上游 → 立即 `completed`
- **异步**：上游 task id 写入 `execution.upstream.task_id` → 后台轮询
- **不可查询上游**：阻塞等待或拒绝后台任务，**不允许只靠内存**

Usage 是完成后的唯一事实来源，任务只经 `usage_id` 关联，只记一个实际模型。
