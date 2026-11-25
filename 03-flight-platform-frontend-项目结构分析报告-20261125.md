# 飞参判读平台 · 前端项目结构分析报告（20261125 版）

> 本文件为新增文档，首次梳理 `frontend/flight-platform-frontend/` 仓库。仓库是**完整实现并有提交历史**（Electron + Vue 3 桌面端），**不是空仓**。所有路径、包名、版本均已从代码核实；文档与代码冲突时**以代码为准**。

---

## 1. 项目概览

### 1.1 是什么

飞参判读平台的桌面客户端，基于 **Electron 22.3.27 + electron-vite 2.3 + Vue 3.5** 构建，负责飞参数据可视化、ICD 字段表格编辑、图表绘制、报告查看与判读任务管理。

### 1.2 关键定位

- **前端与后端分离部署**：前端只通过 HTTP 连接后端服务，后端（JDK + JAR）**不随安装包分发**。后端地址写在 3 处，必须同步改（见 §4.4 坑 #1）。
- **Win7 硬约束**：必须兼容 Windows 7 SP1 及以上，Electron 锁在 22.3.27（最后一个支持 Win7/8/8.1 的版本；23+ 要求 Win10）。
- **Node 两层**：开发/构建侧用 Node 18.18.0+（`.nvmrc` = `18.20.8`）；Electron 主进程内置运行时是 Node 16.17.1（由 Electron 版本决定，不可升）。二者不可混淆。

### 1.3 平台硬约束：Windows 7

| 项 | 约定 | 说明 |
|---|---|---|
| 操作系统 | Windows 7 SP1 及以上 | 硬约束，Electron 不能升到 23+ |
| Electron | 22.3.27 | 内置 Node 16.17.1 / Chromium 108；已 EOL，不再收安全更新 |
| 开发/构建 Node | 18.18.0+（推荐 18.20.x） | `.nvmrc` = `18.20.8`，`package.json` `engines` 已声明 |
| npm | 9+ | 随 Node 18.18 起附带 |

依赖版本是为「Node 18 能跑 + 兼容 20/22」刻意锁的：**不要随手升级**：

| 包 | 锁定版本 | 原因 |
|---|---|---|
| vite | ^5.4.21 | vite 6/7 要求 Node 20.19+ |
| @vitejs/plugin-vue | ^5.2.4 | 6.x 要求 Node 20.19+，且只配 vite 6/7 |
| electron-vite | ^2.3.0 | 3.x 起解除 vite 5 支持并抬 Node 下限 |
| eslint | ^9.39.5 | 10.x 要求 Node 20.19+ |
| sass | **~1.99.0** | 1.100.0 起 `engines` 变 `>=20.19`，必须锁小版本 |
| electron | **22.3.27** | Win7 兼容的最后一个大版本 |
| electron-builder | 24.13.3 | 25.x 依赖 `minimatch@10`（要求 Node 20+） |

`overrides` 把 rollup 换成 `npm:@rollup/wasm-node@4.62.2` 是 Win7 **强制项**，改它必须删 `package-lock.json` + `node_modules` 后重装。

---

## 2. 技术栈

| 维度 | 选型 |
|---|---|
| 桌面容器 | Electron 22.3.27（主进程 Node 16.17.1） |
| 构建工具 | electron-vite 2.3.0 + Vite 5.4.21 |
| 前端框架 | Vue 3.5.25（**全部 `<script setup>`**，无 Options API） |
| 语言 | **纯 JavaScript**（无 `tsconfig`，只有 `jsconfig.json`） |
| UI 组件库 | Element Plus 2.14.3 |
| 状态管理 | Pinia 4.0.2（setup-style store） |
| 路由 | vue-router 4.6.4（**hash history**，Electron 兼容） |
| 图表 | 自研 Canvas 引擎（`components/drawing/chart/`，**非 ECharts**） |
| 图表库（仅监控页） | ECharts 5（仅 `pages/monitor/cache/CacheMonitorPage.vue` 使用） |
| HTTP 客户端 | axios（封装在 `utils/http.js`） |
| 样式 | Prettier + EditorConfig（2 空格、LF、单引号、无分号、printWidth 100） |
| 打包 | electron-builder 24.13.3（Windows 仅 NSIS，Linux AppImage/deb/snap） |
| 自动更新 | electron-updater 6.3.9（S3 / generic  provider，MinIO 桶 `flight-updates`） |

---

## 3. 目录结构

```
frontend/flight-platform-frontend/
├── src/main/                         # Electron 主进程（7 文件）
│   ├── index.js                      # 入口：窗口创建、app 生命周期、APP_ID
│   ├── backend.js                    # 后端可达性探测，BACKEND_ORIGIN = 'http://localhost:8080'
│   ├── menu.js                       # 中文应用菜单
│   ├── updater.js / updateFeed.js    # electron-updater 自动更新（S3 / generic）
│   ├── drawingWindow.js              # 绘图窗口 IPC
│   ├── variableSelectionWindow.js    # 变量选择窗口 IPC
│   ├── redundancyMemberWindow.js     # 冗余成员窗口 IPC
│   └── versionDiffWindow.js          # 版本差异窗口 IPC
├── src/preload/index.js              # 预加载，contextBridge 暴露 window.electron / window.api
├── src/renderer/                     # 渲染进程入口 HTML + CSP
└── src/renderer/src/                 # 所有应用代码
    ├── api/                          # axios 封装，按域分目录
    │   ├── dict.js, login.js, user.js, resourceGrant.js
    │   ├── icd/icdApi.js
    │   ├── graph/{graph,plotVariable,formulaLibrary}.js
    │   ├── judgement/judgement.js
    │   ├── keyword/keyword.js
    │   ├── monitor/* (7 文件)
    │   ├── report/report.js
    │   ├── system/* (7 文件)
    │   ├── task/task.js
    │   └── template/template.js
    ├── components/                   # Vue 3 SFC，全部 <script setup>
    │   ├── layout/                   # MainLayout, Toolbar, TabPanel, FastViewBar …
    │   ├── icd/                      # FieldTable, FieldCellEditor, ICDProjectTree, AsyncDetail, StofDetail, TemplateDetail …
    │   ├── drawing/                  # 工具栏、MessageTree、VariableTable、公式对话框、chart/
    │   ├── PopupWindow/              # 关于/导入/导出/透视/偏好设置/更新/发行说明/快捷键 等对话框
    │   ├── common/, problems/
    │   └── drawing/chart/            # 自研 Canvas 图表引擎（非 ECharts）
    ├── composables/                  # 业务逻辑复用（drawing/, icd/, useHighlight, useTableWindow …）
    ├── config/featureFlags.js        # 性能治理运行时开关（fieldTableReadonlyText, fieldTableWindowSize）
    ├── data/                         # 静态选项列表（icd, layout, perspective, toolbar）
    ├── directives/                   # hasPermi.js, hasRole.js
    ├── pages/                        # 路由页面（hash history）
    │   ├── drawing/DrawingView.vue
    │   ├── icd/IcdWorkspace.vue, AddRedundancyMemberView.vue
    │   ├── login/LoginPage.vue
    │   ├── monitor/, profile/, report/, system/, task/, template/
    │   └── page-shell.css
    ├── router/                       # index.js, routes.js, guards.js
    ├── stores/                       # Pinia store（setup-style）
    │   ├── index.js                  # barrel 导出
    │   ├── useUserStore.js           # token / userInfo / roles / permissions
    │   ├── dictStore.js, keywordStore.js
    │   ├── icd/treeStore.js
    │   ├── layout/panelStore.js, perspectiveStore.js
    │   └── PopupWindow/dialogStore.js
    ├── styles/                       # 7 个 CSS 文件
    ├── utils/                        # http.js, id.js（雪花 ID 安全转换）, auth.js, permission.js …
    └── main.js                       # 应用入口，CSS import 顺序有讲究
```

路径别名（`jsconfig.json` 与 `electron.vite.config.mjs` **两处要一致**）：

| 别名 | 指向 |
|---|---|
| `@` / `@renderer` | `src/renderer/src` |
| `@root` | 仓库根 |
| `@router` | `src/renderer/src/router` |
| `@pages` | `src/renderer/src/pages` |
| `@utils` | `src/renderer/src/utils` |

---

## 4. Electron 约定

### 4.1 主进程与预加载

- 主入口：`package.json` `main` → `./out/main/index.js`（由 `electron-vite build` 产出）；源码 `src/main/index.js`。
- 预加载：`src/preload/index.js`，所有 `BrowserWindow` 通过 `preload` 加载。
- **所有 `BrowserWindow` 都显式设了 `sandbox: false`**（注释写明「关闭沙箱，允许 preload 脚本访问 Node.js API」），这是**刻意的，不要"修"**。
- `contextIsolation` / `nodeIntegration` **不显式设置**，走 Electron 22 默认（`true` / `false`）。
- preload 通过 `contextBridge` 暴露 `window.electron`（来自 `@electron-toolkit/preload`）和 `window.api`（自定义 IPC 订阅/调用）。

### 4.2 IPC 命名

两套并存的命名风格，新增通道按功能域选既有风格，**不要自创第三套**：

| 风格 | 示例 |
|---|---|
| `namespace:verb` 冒号式 | `credentials:get`、`credentials:set`、`credentials:clear`、`dialog:openDirectory`、`image:write-clipboard`、`image:save`、`backend:check-ready`、`app:get-version`、`updater:get-status`、`updater:check`、`updater:download`、`updater:install`、`updater:status` |
| kebab-case | `open-drawing-window`、`drawing-params`、`drawing-view-ready`、`open-variable-selection-window`、`variable-selection-params/ready/confirm/result/closed/applied`、`open-version-diff-window`、`version-diff-params`、`version-diff-ready`、`open-redundancy-member-window`、`redundancy-member-params`、`redundancy-member-view-ready`、`redundancy-member-added`、`datasource-changed` |
| 菜单事件 | `show-about-dialog`、`show-import-wizard`、`show-export-wizard`、`show-perspective-dialog`、`show-preferences`、`show-shortcuts-dialog`、`show-release-notes`、`show-update-dialog`、`open-file`、`backend-ready` |

所有 `ipcMain.handle` 通道通过 `window.api.*` 调用；预加载里的 `subscribe(channel, cb)` 返回一个 unsubscribe 函数。

### 4.3 渲染 → 后端（HTTP，不是 IPC）

渲染进程访问后端走**纯 HTTP（axios）**，不走 IPC。后端地址写在 **3 个地方，必须同步改**：

1. `src/main/backend.js` 的 `BACKEND_ORIGIN = 'http://localhost:8080'`
2. `src/renderer/src/utils/http.js` 的非 DEV 分支 `baseURL`（DEV 分支走 `/api` 代理）
3. `src/renderer/index.html` 的 CSP `connect-src`

Dev 代理在 `electron.vite.config.mjs`：`/api` → `http://localhost:8080`，`changeOrigin: true`，`rewrite` 去掉 `/api` 前缀。

### 4.4 包与构建约定

- `electron-builder.yml` 的 `appId: com.huike.flightplatform` 必须与 `src/main/index.js` 的 `APP_ID` 一致（决定任务栏分组、通知 AUMID、升级识别）。
- Windows 目标**只有 NSIS**（portable 版无法自动更新，已不再产出）。
- **后端不随安装包分发**（JDK + JAR 已移除），`electron-builder.yml` 没有 `extraResources` / `asarUnpack` 后端配置。
- `externalizeDepsPlugin()` 必须保留在 `main` 和 `preload`（electron-vite 2.x 不自动外部化，删了会导致 `keytar.deletePassword is not a function`）。
- `renderer.build.assetsInlineLimit: 0` 是必需的（Chromium 的 `cursor:url()` 不支持 data URI）。
- `npmRebuild: false`（跳过原生模块重新编译）。
- Electron 下载镜像走 npmmirror（`.npmrc` 已配置）。

### 4.5 自动更新

- `electron-updater`，`autoDownload = false`，`autoInstallOnAppQuit = false`，`autoRunAppAfterInstall = true`。
- 检查频率：10s 延迟初始检查 + 4h 间隔，仅在后端就绪后启动。
- 发布源：S3（MinIO 桶 `flight-updates`，`endpoint http://192.168.2.201:9001`，`acl: null` 必须保留，MinIO 不支持 canned ACL）。
- `dev-app-update.yml` 仅用于 dev，通过 `forceDevUpdateConfig` 启用，指向 `http://127.0.0.1:9000/flight-platform-updates/updates`。
- 环境变量覆盖：`FLIGHT_UPDATE_MODE`、`FLIGHT_UPDATE_URL`、`FLIGHT_UPDATE_BUCKET`、`FLIGHT_UPDATE_ENDPOINT`、`FLIGHT_UPDATE_REGION`、`FLIGHT_UPDATE_PATH`、`FLIGHT_UPDATE_CHANNEL`、`FLIGHT_UPDATE_AUTH`、`FLIGHT_UPDATE_MULTIRANGE`、`FLIGHT_UPDATE_DOWNLOAD_PAGE`。

---

## 5. 代码约定

### 5.1 语言与框架

- **纯 JavaScript**，`find src -name "*.ts"` 返回 0。`.mjs` 用于 11 个需被纯 Node 直接加载的文件（如 `fieldTableModel.mjs`、`requestResponse.mjs`、`snowflakeId.mjs` 等），**不要统一改成 `.js`**。
- **全部 136 个 `.vue` 都用 `<script setup>`**，没有 Options API / `defineComponent`。
- 组件文件 PascalCase，页面组件用 `Page.vue` / `View.vue` / `Workspace.vue` 后缀。
- 路由用 **hash history**（`createWebHashHistory()`），为 Electron 兼容，**不要**换成 history 模式。

### 5.2 格式化与检查

- **Prettier**（`.prettierrc.yaml`）：单引号、**无分号**、`printWidth: 100`、无尾随逗号、LF。
- **EditorConfig**（`.editorconfig`）：2 空格缩进、LF、UTF-8、文件末尾换行、去尾随空格。
- **ESLint**（`eslint.config.mjs`，ESLint 9 flat config）：`@electron-toolkit/eslint-config` + `@electron-toolkit/eslint-config-prettier` + `eslint-plugin-vue` `flat/recommended`；`vue/require-default-prop` 与 `vue/multi-word-component-names` 已关闭。
- 自定义 `no-restricted-syntax` 规则：**拦截 `Number(x.id)` / `parseInt(x.id)`**（参数名以 `id` 结尾，`messageId`、`tabId` 除外），因为雪花 ID 转数字会把相邻几十个 ID **塌缩成同一个值**。用 `@renderer/utils/id` 的 `toIdString` / `normalizeIdList`。
- `git add` 新增 Markdown 文件需 `-f`（`.gitignore` 含 `*.md`，已跟踪的不受影响）。

### 5.3 样式

- 纯 CSS 在 SFC `<style scoped>` 中 + 全局 CSS under `src/renderer/src/styles/`。
- `sass` 是 devDependency 但样式实际为 `.css`。
- `src/renderer/src/main.js` 的 CSS import 顺序有讲究：**必须**按 `element-plus/dist/index.css` → `splitpanes-theme.css` → `normalize.css` → `assets/main.css` → `cjk-typography.css` → `global-select-lock.css`（最后一个必须在组件库 CSS 之后）。

---

## 6. API 层语义（容易读错）

axios 实例在 `src/renderer/src/utils/http.js`：

- `baseURL: import.meta.env.DEV ? '/api' : 'http://localhost:8080'`，`timeout: 10000`。
- 请求拦截器注入 `Authorization: Bearer <token>`（token 从 `localStorage['user-token']`，`utils/auth.js`）。
- 自定义 `transformResponse: parseJsonSafeBigInt`——把 **≥16 位的 JSON 数字重新加引号成字符串**，保证 64 位 long ID 不丢精度。
- 响应处理在 `utils/requestResponse.mjs`：
  - **成功判定不是 HTTP 状态码**：`unwrapResponse(body)` 把 `body.data` **平铺到顶层**（同时保留 `code`/`msg`/`traceId`/`error`），`isFailureEnvelope(body)` 判定「HTTP 200 但 `code !== 200`」也是失败。所以 API 函数拿到的是**业务载荷 + 信封字段并存**（`res.code`、`res.rows`、`res.token`）。
  - **401**：`handleUnauthorized()` 清除 token、提示「登录状态已失效，请重新登录」、跳转到 `Login` 路由并带 `?redirect=`。仅在未重置前触发一次。
  - **401/403 必须停轮询**：`isPollingAccessError(error)` 存在就是为了避免轮询时错误提示刷屏。
  - 抑制全局报错：请求 config 传 `{ skipErrorMessage: true }`。
  - Blob 响应用 `responseType: 'blob'` + `skipErrorMessage: true`。

API 模块风格：一函数一端点，`api/` 按域分目录。端点**不带** `/api` 前缀（dev 代理会去掉），如 `/icd/tree`、`/templates`、`/judgement-tasks/page`、`/graph/query`、`/flight-data/list`、`/login`、`/getInfo`。

登录页用 RSA（`utils/jsencrypt.js`）加密密码后再发送。

---

## 7. 状态管理（Pinia）

`src/renderer/src/stores/`，全部 setup-style：

| 文件 | store id | 内容 |
|---|---|---|
| `stores/index.js` | （barrel） | 重导出所有 store |
| `stores/useUserStore.js` | `user` | `token`、`userInfo`、`roles`、`permissions`、`userInfoLoaded`、`isLoggedIn`、`login/getUserInfo/logout/clearUserState` |
| `stores/dictStore.js` | `dict` | `dictionaries` map + `getDict/setDict/removeDict/clearDict` |
| `stores/keywordStore.js` | `keyword` | keyword list、`keywordMap`、`getKeyword/getArgument` |
| `stores/icd/treeStore.js` | `tree` | `refreshKey`、`redundancyRefreshKey`、`highlightGroupId` |
| `stores/layout/panelStore.js` | `panel` | 3 面板 dock 状态（`sidebar`/`main`/`bottom` + `templateEdit`）、`panelStates`、`fastViewBarStates`、`dirtyTabs` Set |
| `stores/layout/perspectiveStore.js` | `perspective` | 透视/布局选项 |
| `stores/PopupWindow/dialogStore.js` | （dialog） | 弹窗对话框状态 |

Pinia **不做持久化**，`router/guards.js` 会在进入受保护路由前重新拉取 `/getInfo`（`ensureUserLoaded`），否则权限为空、`v-hasPermi` 会显示全部。

---

## 8. 飞参判读平台域特有

### 8.1 ICD 字段表格虚拟表（性能敏感区）

性能治理方案已落地，改动前必须读仓库根 `ICD字段表格大数据编辑性能优化-内网修改说明.md`。核心规则：

- 用 Element Plus **`el-table-v2` + `el-auto-resizer`**，只挂载视口行 + 少量缓存行（`cache: 5`）。
- **行高固定 28 px**（`ROW_HEIGHT`），滚动计算依赖它 —— **不要改成自适应行高**。
- **同时只有一个活动单元格编辑器**；编辑态显示文本，点中单元格才实例化编辑器。
- **提交分两步**：先 `commitActiveEditor()` 把草稿写回原字段并 emit（**不发 HTTP**），再由页面的 `saveAll()` 落库。保存按钮必须走 `saveFields()` 包装（`commitActiveEditor()` + `saveAll()`）。
- **切编辑态、切空闲域、重新筛选、切子 Tab、搜索、保存、离开页面前都必须先 commit**。`TemplateDetail.vue` 需要 `flush: 'sync'` 的 `watch(activeChildIndex)`，否则草稿写到新表而不是旧表。
- **旧实现内部一律作废**：`tableRef.value.tableRef`、`visibleData` 切片、`renderedCount`/`loadMoreRows` 倍增、`WINDOW_SIZE`/`EDIT_WINDOW_SIZE`、`scrollIntoView`、`querySelector('.selected-row'/'.highlighted-row')`、`tr` DOM 查找。改用公开方法 `commitActiveEditor()` / `clearSelection()` / `toggleRowExpansion(row, expanded)` / `scrollToField(id)`。
- **`sourceData` prop 对筛选场景必填** —— `flattenFields` 必须返回**原始字段对象**（`row` 指向原对象而非筛选副本），否则编辑看着生效实际没存上。
- **IME 组字期间不能响应 Enter/Esc/Tab**；键盘监听用 `capture` 保证下拉框仍能收到 Enter。
- **性能验收口径：UI 延迟 P95 ≤ 200 ms**（旧实现 1000 字段时 P95 约 1490 ms，新实现 1000/2000/5000 字段均约 35 ms）。
- **影响 6 个调用页面**：`AsyncDetail.vue`、`StofDetail.vue`、`TemplateDetail.vue`、`AddRedundancyMemberView.vue`、`IcdFieldPickerDialog.vue`、`TemplateAddVariableDialog.vue`。
- 改动范围限定在文档列出的文件（2 个新增 + `FieldTable.vue` 整体替换 + 6 个调用页局部改动），**不要混入绘图相关的无关改动**。
- 性能回滚开关在 `config/featureFlags.js`（`fieldTableReadonlyText`、`fieldTableWindowSize: 150`）和 `composables/useTableWindow.js` —— 旧窗口化方案遗留，**其他冗余对话框还在用，按要求保留不要删**。

### 8.2 图表引擎（自研 Canvas，非 ECharts）

- 引擎位于 `components/drawing/chart/`（`CanvasChart.js` + `core/` + `session/` + `presets/flight/`）。
- 刻度数学**只在** `core/tick.js`；格式化在 `core/format.js`；标签在 `core/labels.js`；系列颜色在 `core/series.js`。
- 业务无关配置在 `core/config.js`；飞参 profile 注入在 `presets/flight/profile.js`（通道 `CHA/CHB/CHC/CHD`，X 轴 `拍数`/`时间`，Y 轴 `解析数据`/`原始数据`）。
- 离散量（`bit*`/`group*` 前缀或 `bool` 且 `bitSize > 0`）按 bit 展开成多条曲线，y-range 固定 `[-1, start+size]`。
- 系列标题：messageId `2629536` → `[blockName]name`，否则 `[HEX(messageId)]name`。
- **`chart/` 目录必须保持自包含** —— `npm run check:chart-portable` 会用 esbuild 打包校验，目录里出现 `@renderer` / `@utils` / `@/` / `api/graph` / `useFieldSelection` / `localStorage` 就失败。
- **静态渲染单文件**：`npm run build:chart` 产出 `chart-dist/chart-renderer.js`（IIFE，`window.FlightChart.renderStatic`），**要手工复制到后端仓库的 `node-renderer/src/public/chart-renderer.js`**。目标 `chrome108`、不压缩。改前端图表**必须**同步后端那份，否则后端渲染的还是旧版。
- README 说的入口 `src/renderer/src/composables/drawing/staticChartRenderer.js` **不存在**，真实入口是 `components/drawing/chart/static/renderStatic.js`（README 已过时）。

### 8.3 性能治理开关

`config/featureFlags.js` 只放「可安全回滚」的运行时开关，默认值均已实测：

| 开关 | 含义 |
|---|---|
| `fieldTableReadonlyText: true` | 只读态单元格渲染纯文本，不再实例化 `el-input` / `el-select` |
| `fieldTableWindowSize: 150` | 首屏只渲染该行数，滚动到底部再追加 |

出问题时改这里即可回退到旧行为，无需回滚代码。

---

## 9. 非明显坑（改动前必读）

| # | 坑 | 影响 | 处置 |
|---|---|---|---|
| 1 | **后端地址 3 处必须同步**：`src/main/backend.js`、`src/renderer/src/utils/http.js`、`src/renderer/index.html` CSP | 改一处不改另一处 ⇒ 打包后连不上后端 | 三处一起改 |
| 2 | **雪花 ID 不能转数字**：`icd_field.id` 等超出 JS 安全整数范围，`Number()` 会把相邻几十个 ID 塌缩成同一个值 | ESLint 拦截 `Number(x.id)` / `parseInt(x.id)`（`messageId`、`tabId` 除外） | 用 `@renderer/utils/id` 的 `toIdString` / `normalizeIdList` |
| 3 | **`.gitignore` 含 `*.md`**：新增 Markdown 默认不被跟踪 | 需 `git add -f` 才能提交 | 仓库根 ICD 性能说明文档已因此被忽略 |
| 4 | **`overrides` rollup wasm-node 是 Win7 强制项**：改它必须删 `package-lock.json` + `node_modules` 重装 | 原生 rollup 在 Win7 上加载不了 | 不要随手改 |
| 5 | **`externalizeDepsPlugin()` 不能删**：electron-vite 2.x 不自动外部化 | 删了 `keytar.deletePassword is not a function` | 保留 |
| 6 | **`@element-plus/icons-vue` 未声明在 `package.json`**：仅靠 element-plus 传递依赖提升解析 | 很脆，升级可能断 | 不要轻易动 `main.js` 的 import |
| 7 | **`echarts-gl`、`dockview-vue` 声明但未用**：ECharts 只在监控缓存页用 | 无用依赖 | 不要随手删（可能被其他代码间接依赖） |
| 8 | **路由 hash history**：Electron 兼容所需 | 换成 history 模式会在 Electron 里出错 | 保留 hash |
| 9 | **无 `.env` 文件，但 `import.meta.env.VITE_APP_BASE_API` 被读取**：会是 `undefined` | `useUserStore.js:75`、`UserProfilePage.vue:95` 的头像 URL 可能异常 | 需要时自行补充 `.env` |
| 10 | **全局选择锁 / 复制剪切守卫**是故意禁止编辑区外选中文本 | 不要当 bug 修 | 保留 |
| 11 | **权限判定语义**：`checkPermi` / `checkRole` 对空权限集合返回 `false`，仅精确码匹配（`*:*:*` 是 admin），**不支持 `icd:*` 前缀通配** | 权限控制可能比预期更严格 | 按精确码编写 |
| 12 | **Linux 支持残缺**：自动更新只对 AppImage 有效，deb/snap/Windows portable 降级为「手动下载」 | 跨平台发布时注意 | 只保证 Windows NSIS |

---

## 10. 现有文档

| 文档 | 说明 |
|---|---|
| `README.md` | 环境要求（Node 18 开发/构建 + Win7 运行）、版本锁定原因、`externalizeDepsPlugin` 不能删、Win7 开发环境额外处理、IDE 推荐、构建命令、后端静态图渲染单文件 |
| `ICD字段表格大数据编辑性能优化-内网修改说明.md` | ICD 字段表格性能优化完整说明（9 个业务必需文件 + 6 个验证文件 + 附录完整代码 + SHA-256 校验） |
| `scripts/tests/icd-table/README.md` | ICD UI 回归测试工具说明与性能记录 |
| `changelog.json` | 发行说明，显示在「关于/发行说明」对话框 |

> `docs/` 下不属于前端仓库，前端相关结构分析请参阅本文；后端结构分析请参阅 `02-*`；协作工作流程标准请参阅 `01-*`。

---

## 11. 验证命令（cwd = `frontend/flight-platform-frontend/`）

```bash
npm run dev                    # electron-vite dev
npm run build                  # electron-vite build
npm run lint                   # eslint --cache .
npm run format                 # prettier --write .
npm run test:icd               # node --test，ICD 字段表格模型
npm run test:icd-ui            # ICD 表格 UI 回归脚本
npm run check:chart-portable   # 图表目录自包含性校验
npm run build:chart            # 构建后端静态图渲染单文件 chart-renderer.js（需同步到后端 node-renderer）
npm run debug                  # dev + 远程调试端口 9222
npm run build:win              # electron-builder 打包（--publish never）
npm run release:win            # 打包并上传（--publish always）
```

> **没有 `npm test`，也没有 `npm run typecheck`**（纯 JS 项目，`lint` 即静态检查）。README 里写了 `npm run build:mac`，但 `package.json` 里**没有这个脚本**。

---

## 附录 A. 前端与后端交互边界

| 边界 | 说明 |
|---|---|
| 渲染 → 后端 | axios HTTP，`http://localhost:8080`，3 处地址需同步 |
| 渲染 → 主进程 | IPC via `window.api.*`，两套命名风格（冒号式 / kebab-case） |
| 主进程 → 系统凭据 | keytar（Windows 凭据管理器 / macOS Keychain / Linux Secret Service），密码不写入 localStorage |
| 前端 → Node 渲染服务 | 不直接通信，经后端 `ChartRenderService` REST 调用 `node-renderer.base-url` |
| 前端 → 自动更新 | electron-updater 从 S3/MinIO 拉取 `latest.yml`，桶 `flight-updates`，`acl: null` |
| 图表单文件 | 前端 `npm run build:chart` → 手工复制到后端 `node-renderer/src/public/chart-renderer.js` |
