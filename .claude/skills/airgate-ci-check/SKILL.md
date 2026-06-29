---
name: airgate-ci-check
description: 提交前强制自检。按所在仓运行 make ci，重点把关生成代码漂移（ent / proto / theme / plugin.yaml），避免漏 regen 导致 CI 失败。
---

# 提交前自检（漂移护栏）

声明完成前，按所在子项目执行检查。最易遗漏：改 schema/proto/theme 后未 regen 提交，本地通过但 CI 失败。

## 按仓选择

### airgate-core（`airgate-core/`）

```bash
make ci      # lint + test + vet + verify-ent + verify-readme + build
```
- 改 `ent/schema/` → 先 `make ent` 并提交生成代码（`verify-ent` 检查漂移）。
- 改 `README.md` 或 `README_EN.md` → 两份须结构同步，否则 `verify-readme` 失败。
- 仅改前端 → `make lint`（tsc + eslint）+ `make dev-frontend` 验证。

### airgate-sdk（`airgate-sdk/`）

```bash
make ci      # lint + test + vet + build + proto-check + theme-check
```
- 改 `protocol/proto/` → 先 `make proto` 提交 `*.pb.go`。
- 改 `theme/` → 先 `make theme` 提交生成 CSS。

### 插件仓（`airgate-<plugin>/`）

```bash
make ci      # ensure-webdist + lint + test + vet + build
```
- `ensure-webdist`：保证 `web/dist` 非空（`go:embed` 要求至少一个文件），CI 自动前置。
- 改模型/路由/账号字段声明 → 先 `make manifest` 重新生成 `plugin.yaml` 并提交。

## 漂移清单（改源即须 regen + 提交）

| 改动 | 运行 | 提交 |
|---|---|---|
| `ent/schema/` | `make ent` | `ent/` 生成代码 |
| `protocol/proto/` | `make proto` | `*.pb.go` |
| `theme/` | `make theme` | 生成的 CSS/dist |
| 插件元信息 | `make manifest` | `plugin.yaml` |

## 报告原则

- 测试/CI 失败如实报告并附输出，未运行时勿声称通过。
- 跨子仓改动须各仓分别运行 ci、分别提交。

## 提交规范

- 约定式提交：`<type>(<scope>)?: <subject>`，type ∈ `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`。
- `make setup-hooks` 安装 `commit-msg`（`scripts/commit-msg`）与 `pre-commit`（`make pre-commit`）钩子；新克隆或换机器后须重跑。
