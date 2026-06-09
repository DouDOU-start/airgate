---
name: core-frontend-page
description: 在 airgate-core 后台前端（web/，React 19 + Vite + TanStack Query + Tailwind）新增或修改页面、列表、表单时使用。钉死 pages/shared/app 三层落点、TanStack Query + queryKeys + shared/api 数据流约定与 theme 用法，避免自创目录或绕开统一封装。
---

# core 后台前端页面开发

技术栈：React 19 + Vite + TanStack Query + Tailwind + `@doudou-start/airgate-theme`。
**别自创目录结构**——照三层落点和现成页面走。

## 第 0 步：先抄现成页面

先完整读一个同形态的现成页面作模板，优先 `web/src/pages/admin/` 下的列表/表单页，照它的形状写。

## 三层落点

| 层 | 位置 | 放什么 |
|---|---|---|
| 页面 | `web/src/pages/{admin,user,setup}/` | 路由页面组件 |
| 复用 | `web/src/shared/{api,hooks,components,ui,columns,types}` | API 封装、查询 hook、通用组件、表格列定义 |
| 装配 | `web/src/app/{router,providers,layout}` | 路由注册、Provider、布局 |

## 数据流约定（务必遵守）

- **服务端状态用 TanStack Query**，不要自己 `useState + fetch` 拉数据。
- **query key 统一在 `shared/queryKeys.ts`** 定义/复用，别散落字面量。
- **HTTP 调用走 `shared/api`** 的封装，不直接 `fetch`/裸 axios。
- 可复用的 UI 进 `shared/components` / `shared/ui`；表格列进 `shared/columns`。
- 样式用 Tailwind + theme，**别引入新 UI 库**或硬编码主题色。

## 编码约定（高频、易违反）

- **HTTP 统一走 `shared/api/client.ts`**：它已封装 `ApiError`、401 自动刷新 token 重试、网络错误的 i18n 消息。**别绕过它裸 `fetch`/axios**，否则丢掉全局鉴权与错误处理。
- **变更操作用 `useCrudMutation`**（`shared/hooks`）：自带成功 toast、失败 toast（`err.message`）、`invalidateQueries`。别手写 `useMutation` 重复这套。
- **错误/loading 态**：用 TanStack Query 的 `isLoading/isError` 渲染，错误消息用后端返回的 `message`（已是用户可读文案），别自己拼。
- **i18n 文案不硬编码**：用 `react-i18next` 的 `useTranslation()` + `t('...')`，key 进 `web/src/i18n`（参考任一 `pages/admin` 页面）。
- **表单校验**就近在提交前做，错误用 toast/inline 提示，沿用现成页面的写法。

## 新增一个页面的顺序

1. 在 `shared/api` 加接口调用；在 `shared/queryKeys.ts` 加 key。
2. （如复杂）在 `shared/hooks` 封装 `useQuery`/`useMutation` hook。
3. 在 `pages/<区域>/` 写页面组件，复用 `shared/components`、`shared/ui`。
4. 在 `app/router`（必要时 `app/layout`）注册路由/入口。
5. i18n 文案进 `web/src/i18n`（若该页面有用户可见文本）。

## 自检

```bash
make dev-frontend     # vite 起前端单独看效果
make lint             # 含 tsc --noEmit + eslint
```
