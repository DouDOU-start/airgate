# 后台前端页面

技术栈：React 19 + Vite + TanStack Query + Tailwind + `@doudou-start/airgate-theme`。

## 第 0 步：参照现有页面

优先阅读 `web/src/pages/admin/` 的列表/表单页，沿用其结构。

## 三层落点

| 层 | 位置 | 职责 |
|---|---|---|
| 页面 | `web/src/pages/{admin,user,setup}/` | 路由页面组件 |
| 复用 | `web/src/shared/{api,hooks,components,ui,columns,types}` | API 封装、查询 hook、通用组件、表格列定义 |
| 装配 | `web/src/app/{router,providers,layout}` | 路由注册、Provider、布局 |

## 数据流约定

- 服务端状态用 TanStack Query，勿用 `useState + fetch`
- query key 统一定义于 `shared/queryKeys.ts`，勿散落字面量
- HTTP 经 `shared/api` 封装（已含 `ApiError`、401 自动刷新、i18n 消息），勿裸 `fetch`/axios
- 变更操作用 `useCrudMutation`（`shared/hooks`），内置 toast 与 `invalidateQueries`
- 样式用 Tailwind + theme，勿引入新 UI 库或硬编码主题色
- i18n 文案不硬编码：`useTranslation()` + `t('...')`，key 置于 `web/src/i18n`

## 新增页面顺序

1. `shared/api` 加接口调用；`shared/queryKeys.ts` 加 key
2. （如复杂）`shared/hooks` 封装 `useQuery` / `useMutation` hook
3. `pages/<区域>/` 编写页面组件，复用 `shared/components`、`shared/ui`
4. `app/router`（必要时 `app/layout`）注册路由/入口
5. 用户可见文本的 i18n key 置于 `web/src/i18n`

## 路由守卫

- `withSetupCheck()` 包裹路由，检测初始化状态
- `checkAdmin()` 强制管理员路由权限
- API Key 会话只能访问用量页面（受限模式），不继承 admin/user 权限
- 路由须匹配后端分组的鉴权层级（admin 页 → adminGroup，user 页 → userGroup）

## 路由懒加载

- `lazyWithPreload()` 模式：懒加载组件 + 空闲时预加载
- `ADMIN_IDLE_PRELOADS` / `USER_IDLE_PRELOADS` 列表控制预加载优先级
- 空闲检测：`requestIdleCallback` + 2.5s 超时降级到 `setTimeout(500ms)`

## 插件前端加载

- 插件前端为单 `index.js` ESM bundle，react/react-dom/react-i18next 作为 external 共享
- 加载器将 bare import 重写到 `window.__airgate_shared` 映射
- CSS 并行加载，id=`plugin-css-{pluginId}`
- **主题样式清理**：加载插件时移除旧 SDK 注入的 `theme-vars` 样式节点，防止颜色冲突
- 每个插件缓存一份，支持缓存失效通知

## 插件前端注册表

插件通过 platform 键注册 5 种组件：

| 注册类型 | 用途 |
|---|---|
| `platformIcon` | 平台图标 |
| `accountIdentity` | 账号身份展示 |
| `usageMetricDetail` | 用量指标详情 |
| `usageModelMeta` | 模型元信息 |
| `usageCostDetail` | 成本详情 |

- platform 键统一小写
- 注册支持 pub/sub，版本号递增触发重渲染
