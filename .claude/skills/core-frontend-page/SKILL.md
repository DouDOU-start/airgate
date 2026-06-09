---
name: core-frontend-page
description: 在 airgate-core 后台前端（web/，React 19 + Vite + TanStack Query + Tailwind）新增或修改页面、列表、表单时使用。规定 pages/shared/app 三层落点、TanStack Query + queryKeys + shared/api 数据流约定与 theme 用法，避免自创目录或绕开统一封装。
---

# core 后台前端页面开发

技术栈：React 19 + Vite + TanStack Query + Tailwind + `@doudou-start/airgate-theme`。
按三层落点与现有页面开发，勿自创目录结构。

## 第 0 步：参照现有页面

完整阅读一个同形态现有页面作模板，优先 `web/src/pages/admin/` 的列表/表单页，沿用其结构。

## 三层落点

| 层 | 位置 | 职责 |
|---|---|---|
| 页面 | `web/src/pages/{admin,user,setup}/` | 路由页面组件 |
| 复用 | `web/src/shared/{api,hooks,components,ui,columns,types}` | API 封装、查询 hook、通用组件、表格列定义 |
| 装配 | `web/src/app/{router,providers,layout}` | 路由注册、Provider、布局 |

## 数据流约定

- 服务端状态用 TanStack Query，勿用 `useState + fetch`。
- query key 统一定义于 `shared/queryKeys.ts`，勿散落字面量。
- HTTP 经 `shared/api` 封装，勿直接 `fetch` / 裸 axios。
- 可复用 UI 置于 `shared/components` / `shared/ui`，表格列置于 `shared/columns`。
- 样式用 Tailwind + theme，勿引入新 UI 库或硬编码主题色。

## 编码约定（高频）

- HTTP 统一经 `shared/api/client.ts`：已封装 `ApiError`、401 自动刷新重试、网络错误 i18n 消息；勿绕过裸 `fetch`/axios，否则丢失全局鉴权与错误处理。
- 变更操作用 `useCrudMutation`（`shared/hooks`）：内置成功/失败 toast 与 `invalidateQueries`；勿手写 `useMutation` 重复。
- 错误/loading：用 TanStack Query 的 `isLoading/isError`，错误消息取后端返回的 `message`（已为用户可读文案）。
- i18n 文案不硬编码：`react-i18next` 的 `useTranslation()` + `t('...')`，key 置于 `web/src/i18n`。
- 表单校验于提交前就近执行，错误经 toast / inline 提示。

## 新增页面顺序

1. `shared/api` 加接口调用；`shared/queryKeys.ts` 加 key。
2. （如复杂）`shared/hooks` 封装 `useQuery` / `useMutation` hook。
3. `pages/<区域>/` 编写页面组件，复用 `shared/components`、`shared/ui`。
4. `app/router`（必要时 `app/layout`）注册路由/入口。
5. 用户可见文本的 i18n key 置于 `web/src/i18n`。

## 自检

```bash
make dev-frontend     # vite 预览
make lint             # tsc --noEmit + eslint
```
