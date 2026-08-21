# webui-vue-autotest 前端开发

Vue3 + Vite 前端仓库（webui-vue-autotest）开发规范。本文件常驻上下文；明细规则走指针按需加载，保持主文件精简。

## 核心编码规则（必须遵守）

- **script setup 区块顺序**：`store` → `route` → `const` → `hooks` → `refs` → `reactive` → `computed` → `watch` → `lifecycle` → `methods`；区块注释用 `/* 名称 */`
- **注释**：一律中文；vue 页面 js 内用 `//`（store 的 JSDoc 例外）；每个变量/函数上方加中文注释说明用途
- **命名**：表格列 `*Columns`、分页 `*Pagination`、方法 `handle*`/`on*`/`get*`
- **无 TypeScript**：纯 JavaScript，不加类型注解
- **组件命名**：script 引入 naive 用 CamelCase（`NTag`、`NButton`）；template 强制 kebab-case（`n-tag`、`n-button`）
- **API 归属**：请求统一放进 `src/store/modules/{module}.js`（用 `request` + pinia `defineStore`），页面组件不直接发请求
- **复用优先**：先查 `src/components/`、`src/composables/`、`src/utils/dict.js` 有无现成（CommonPage / MeCrud / MeQueryItem / MeModal / AppCard）

### 前端与后端分工（能后端做就后端做）

前端尽量只做展示：数据截断/格式化、状态映射、姓名解析 → 后端加字段返回中文；前端只调接口、渲染表格、绑点击。参数校验任一侧做即可，不必两侧都做。

## 指针（按需加载，不常驻）

| 需要什么 | 去读 |
|---|---|
| naive 组件清单 / API 风格 / store 模板 | `.claude/fe-rules/components-api.md` |
| UnoCSS 样式速查 / 图标 / 字典函数 / getAuth 权限模板 | `.claude/fe-rules/style-dict-auth.md` |
| 项目前端环境（初始化/启动/pnpm）/ iconify 踩坑 / 前端 git 提交踩坑 | `.claude/fe-rules/env-pitfalls.md` |
| 前端 commit message 格式 | `.clinerules/commit-message.md` |

## 环境速查

- 路径 `/home/bmc/sd1/CODE/webui-vue-autotest`（独立 git 仓库，操作前先 `pwd` 确认，别在后端仓库误操作）
- 启动：`pnpm dev --port 8089`；本地调试配置 `.env.development`（`VITE_PROXY_TARGET='http://127.0.0.1:8077'`）、`.npmrc`（`registry=https://registry.npmjs.org`）**禁止删除或提交**
- 新需求前后端拉**同名** feature 分支，都基于各自 `origin/master`

## 常用命令

```bash
pnpm dev        # 启动
pnpm lint:fix   # 代码修复
node_modules/.bin/vite build   # 构建（绕过 postinstall 检查）
```
