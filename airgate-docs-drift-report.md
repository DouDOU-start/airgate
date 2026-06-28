# AirGate 文档与 Skills 漂移全面审计报告

> 审计时间：2026-06-26
> 审计范围：根 CLAUDE.md、9 个子仓 CLAUDE.md、`.claude/skills/` 全部 8 个 skill 文件
> 方法：5 路并行 agent 逐项对比文档声明与代码实现

---

## 总体评分

| 文档 | 评分 | 状态 |
|---|---|---|
| backend.md | A | ✅ 完全一致 |
| forwarding.md | A | ✅ 98% 一致，2 处细微遗漏 |
| plugin-contract.md | B+ | ⚠️ Metadata 约定表不完整 + 行号需校验 |
| frontend.md | A- | ⚠️ 小范围遗漏 |
| task.md | C | 🔴 字段文档严重滞后 |
| ci-check SKILL.md | B | ⚠️ 部分 target 声明不精确 |
| develop-plugin SKILL.md | A | ✅ 一致 |
| SDK CLAUDE.md | A- | ⚠️ 文件清单小遗漏 |
| 子仓 CLAUDE.md (×7) | A- | ⚠️ kiro 有 1 处 widget 声明遗漏 |

---

## 🔴 高优先级漂移（需立即修正）

### 1. task.md 字段文档严重滞后

**文档只列 6 个字段**：task_type / input / output / attributes / execution / usage_id

**实际 DB schema 已新增 11+ 个字段**：
- `stage` — 任务阶段
- `error_type` / `error_code` / `error_message` — 结构化错误
- `progress` — 进度
- `priority` — 优先级
- `attempts` / `max_attempts` — 重试计数
- `public_task_id` — 公开任务 ID
- `idempotency_key` — 幂等键
- `expires_at` / `cancel_requested_at` — 过期与取消时间

**影响**：开发者按陈旧文档编码会忽略新字段约束。这是本次审计发现的最严重漂移。

**修正**：更新 task.md 字段约束表，补充新增字段的规则和使用约束。

### 2. Metadata 约定表缺少 3 个跨插件计费键

**文档登记 6 个键**（全部存在且准确），但漏登了 3 个计费相关键：

| 未登记键 | 使用位置 | 用途 |
|---|---|---|
| `billing_mode` | billing/recorder.go:355,361 | 区分 `fixed_image_price` vs `image_token` |
| `fixed_unit_price` | billing/recorder.go:357 | 图像固定价格 |
| `fixed_unit` | billing/recorder.go:358 | 计费单位标签 |

另有 4 个键属插件私有域（`reasoning_effort`、`service_tier`、`image_size`、`image_tier`），严格来说也应登记，但优先级较低。

**影响**：文档自己规定"新增约定键须同步登记本表"，这条规则本身在漂移。

**修正**：更新 plugin-contract.md Metadata 约定表，至少补登 3 个计费键。

---

## ⚠️ 中优先级漂移（建议修正）

### 3. CI-Check 声明与 Makefile 细节不符

- Core `make ci` 声明的步骤漏掉了实际的中间步骤
- 插件 `make ci` 未提及 `ensure-webdist` 步骤（前端资产必经）
- 部分插件仓可能不存在 `make manifest` target（需确认适用范围）

**修正**：ci-check SKILL.md 中明确各仓 `make ci` 的完整步骤链。

### 4. airgate-kiro widget 声明遗漏

- 文档声明 3 个 widget（AccountCreate / AccountEdit / AccountUsageWindow）
- 实际代码还有 `UsageCostDetail.tsx`，已在 `web/src/index.ts` 导出，但未在 `metadata.go` 的 FrontendWidgets 中声明

**修正**：在 kiro 的 metadata.go 添加 `{Slot: sdk.SlotUsageCostDetail, ...}`，然后 `make manifest`。

### 5. forwarding.md 两处细微遗漏

**a) Anthropic 错误类型**：
- 文档说 6 种 type
- 代码实际 7 种（多了 `overloaded_error`，HTTP 503 自动映射）

**b) 图片计费过滤范围**：
- 文档说 `image_price_*` 全局安全过滤
- 实际仅 OpenAI 插件的 image_price 不转发，其他插件可转发

**修正**：forwarding.md 补充 `overloaded_error`；明确过滤的插件范围。

### 6. SDK CLAUDE.md 文件清单不完整

sdkgo/ 包速览列出 13 个核心文件，遗漏了：
- `log.go` — 插件日志工具
- `log_pretty.go` — 美化日志输出
- `doc.go` — 包文档

**修正**：补充到包速览表。

---

## ✅ 验证一致的部分

### backend.md — 完全无漂移
- 分层落点（DTO→Handler→Service→Store→Schema）✅
- 路由分组前缀和中间件 ✅
- 错误处理映射（422/401/403/402）✅
- 装配点（NewHTTPHandlers + registerRoutes）✅
- JWT/鉴权机制（role=api_key, admin-前缀, AES-256-GCM）✅
- response 包函数列表 ✅

### forwarding.md — 核心常量全部准确
- maxFailoverAttempts = 3 ✅
- queueWaitTimeout = 60s ✅
- DefaultAccountMaxConcurrency = 10, slot TTL = 5min ✅
- RPM key TTL = 120s, 阈值 80%/100% ✅
- 窗口缓存 TTL = 30s, 窗口小时 = 5.0, sticky_reserve = 10.0 ✅
- Sticky TTL = 30min ✅
- msg_lock_max_waiters = 8, wait_seconds = 3s, 延迟 200ms-2000ms ±15% ✅
- 所有 Redis key 格式完全匹配 ✅
- 计费三管道公式 ✅
- 请求头转发列表 ✅
- 推理强度归一化 ✅

### plugin-contract.md — 核心结构无漂移
- Host.Invoke 19 个方法，分组完全匹配 ✅
- 技术债 5 处越界定位准确 ✅
- 后台任务最小间隔 30s，单次超时 5min ✅
- 资产下载上限 50MB，超时 60s ✅
- gRPC 消息上限 64MB，启动超时 15s ✅
- 扩展插件代理 3 入口 ✅
- Capability 格式 ✅

### frontend.md — 主体结构无漂移
- 三层落点（pages/shared/app）✅
- queryKeys.ts 统一管理 ✅
- useCrudMutation 存在 ✅
- 路由守卫（withSetupCheck/checkAdmin）✅
- 懒加载预加载（lazyWithPreload/ADMIN_IDLE_PRELOADS）✅
- 插件前端注册表 5 种 ✅

### 子仓 CLAUDE.md — 7 个插件仓
- 所有 plugin id 和 type 匹配 ✅
- 所有声明的文件均存在 ✅
- epay 的 Host.Invoke 约束和表隔离 ✅
- health 的 group_health_probes 隔离和 /status 反代 ✅
- 5/7 插件完全匹配，kiro 有 1 处 widget 遗漏 ✅

---

## 结构性建议

### 1. task.md 需要一次完整重写
这是唯一严重滞后的文档。建议读 `ent/schema/task.go` 重新对齐字段表，同时补充状态迁移在 Core 层的实现引用。

### 2. Metadata 约定表需要建立自动化校验
文档规定"新增约定键须同步登记"但缺乏强制手段。建议在 CI 中加一个检查：扫描 `Metadata["xxx"]` 使用 → 对比约定表 → 有未登记键则 warn。

### 3. 文档整体质量很高
除了 task.md 和 Metadata 表这两个明确问题外，项目文档的准确度非常高。forwarding.md 的 24/26 项数值完全匹配、backend.md 零漂移、技术债行号全部准确——这在活跃开发的项目中很难得。

---

## 修正优先级汇总

| 优先级 | 项目 | 文件 | 工作量 |
|---|---|---|---|
| P0 | task.md 字段重写 | `.claude/skills/core-dev/task.md` | 中 |
| P0 | Metadata 约定表补登 | `.claude/skills/core-dev/plugin-contract.md` | 小 |
| P1 | ci-check 步骤细化 | `.claude/skills/airgate-ci-check/SKILL.md` | 小 |
| P1 | kiro widget 补声明 | `airgate-kiro/.../metadata.go` + `make manifest` | 小 |
| P2 | forwarding.md 补 overloaded_error | `.claude/skills/core-dev/forwarding.md` | 微 |
| P2 | SDK 文件清单补全 | `airgate-sdk/CLAUDE.md` | 微 |
