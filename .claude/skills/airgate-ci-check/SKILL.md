---
name: airgate-ci-check
description: 在 AirGate 任一子项目（core / sdk / 插件）声称"做完""改好了""可以提交"之前的强制自检护栏。按所在仓跑对应 make ci，重点把关生成代码漂移（ent / proto / theme / plugin.yaml）——AI 常因漏 regen 导致 CI 必挂。
---

# 提交前自检（漂移护栏）

**在说"做完了"之前，按所在子项目跑这里的检查。** 最常见的翻车是：改了 schema/proto/theme 却没 regen 提交，本地能跑、CI 必挂。

## 按仓选择

### airgate-core（从 `airgate-core/`）

```bash
make ci      # lint + test + vet + verify-ent + build
```
- 改过 `ent/schema/` → 必须先 `make ent` 并把生成代码纳入提交（`verify-ent` 会卡漂移）。
- 只改前端可先 `make lint`（含 tsc + eslint）+ `make dev-frontend` 验证。

### airgate-sdk（从 `airgate-sdk/`）

```bash
make ci      # lint + test + vet + build + proto-check + theme-check
```
- 改过 `protocol/proto/` → 先 `make proto` 提交 `*.pb.go`（`proto-check` 卡漂移）。
- 改过 `theme/` → 先 `make theme` 提交生成 CSS（`theme-check` 卡漂移）。

### 插件仓（从 `airgate-<plugin>/`）

```bash
make ci      # lint + test + vet + build
```
- 改过模型/路由/账号字段声明 → 先 `make manifest` 重新生成 `plugin.yaml` 并提交。

## 漂移检查清单（改了源就必须 regen + commit）

| 改了什么 | 必跑 | 提交什么 |
|---|---|---|
| `ent/schema/` | `make ent` | `ent/` 生成代码 |
| `protocol/proto/` | `make proto` | `*.pb.go` |
| `theme/` | `make theme` | 生成的 CSS/dist |
| 插件元信息 | `make manifest` | `plugin.yaml` |

## 报告原则

- 测试/CI 失败就如实报，附输出；别声称"已通过"却没真正跑。
- 跨多个子仓的改动，记得各仓分别跑 ci、分别提交（它们是独立 Git 仓）。

## Commit 规范

- 所有仓用 **约定式提交**：`<type>(<scope>)?: <subject>`，type ∈ `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`。
- `make setup-hooks` 会装 `commit-msg` 钩子（`scripts/commit-msg`）强制校验，并装 `pre-commit`（跑 `make pre-commit`）。新克隆/换机器后记得重跑。
