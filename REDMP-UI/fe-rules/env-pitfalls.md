# 环境 · pnpm · 踩坑

前端项目环境初始化、pnpm 注意事项、iconify 图标加载失败根因与解法、前端 git 提交踩坑。全部为实测踩坑记录，需要时加载。涉及命令均在 `/home/bmc/sd1/CODE/webui-vue-autotest` 下执行。

## 首次初始化

```bash
# 用 npmjs registry + 本地代理指向，前端通过 Vite proxy 转 /api/* 到后端（去 /api 前缀）
sed -i 's|registry=https://registry.npmmirror.com|registry=https://registry.npmjs.org|' .npmrc
sed -i "s|VITE_PROXY_TARGET='http://[^']*'|VITE_PROXY_TARGET='http://127.0.0.1:8077'|" .env.development
http_proxy=http://10.31.3.34:7892 https_proxy=http://10.31.3.34:7892 pnpm install --shamefully-hoist
pnpm dev --port 8089
```

启动 vite 必须 `<setsid env -u VSCODE_CWD node node_modules/vite/bin/vite.js --port 8089 &`（清 VSCODE_CWD，见 iconify 节）。本地调试配置 `.env.development`、`.npmrc`、`.codegraph/`、`.playwright-mcp/` **禁止删除或提交**。

## pnpm 注意事项

| 问题 | 处理 |
|---|---|
| pnpm store 损坏（重复 "Already up to date"） | `rm -rf ~/.local/share/pnpm/store/v11 && rm -rf node_modules` 后重装 |
| vite build 找不到 `lodash` 等 | 必须加 `--shamefully-hoist` |
| 构建绕过 postinstall 检查 | 用 `node_modules/.bin/vite build` 而非 `pnpm build` |
| onnxruntime-node postinstall 超时 | 不影响编译，仅影响该包功能 |
| 前端 dev 用 `pnpm dev --port 8089`（见 CLAUDE.md 主文件） |

## iconify 图标加载失败（unocss `failed to load icon`）

**现象**：dev 图标不渲染（如 `i-material-symbols:terminal`），vite 报 `[unocss] failed to load icon`；生产构建正常。

**根因链**（三层）：
1. unocss `presetIcons` 优先用本地集合包 `@iconify-json/*`（无则 CDN fetch）
2. **关键**：`VSCODE_CWD` 存在时 unocss 判 `isVSCode=true`，跳过本地 nodeLoader 强制走 CDN —— 即使集合包装了也失败
3. `getIcons()`（`build/index.js` + `src/assets/icons/dynamic-icons.js`）把动态图标塞 safelist，`h()` 渲染的图标类名 unocss 扫不到，只可靠 safelist

**解法**：
```bash
# 1. 装齐集合包（走代理，pnpm 11 无 --no-save，装完 checkout 还原清单）
http_proxy=http://10.31.3.34:7892 https_proxy=http://10.31.3.34:7892 \
  pnpm add -D @iconify-json/material-symbols @iconify-json/mdi \
  @iconify-json/simple-icons @iconify-json/octicons @iconify-json/carbon \
  @iconify-json/mynaui @iconify-json/material-symbols-light \
  --registry=https://registry.npmjs.org
git checkout package.json pnpm-lock.yaml   # 不提交依赖清单

# 2. 重启 vite 必须清掉 VSCODE_CWD（关键！否则本地包不生效）
for pid in $(pgrep -f "vite/bin/vite.js"); do kill $pid 2>/dev/null; done
sleep 2
(setsid env -u VSCODE_CWD node node_modules/vite/bin/vite.js --port 8089 \
  > /tmp/vite-pmm.log 2>&1 < /dev/null &)
```

**验证**：devtools 看图标元素 `mask-image` 有 `data:image/svg+xml` 即成功。

**注意**：集合包不声明进 package.json（下次 `pnpm install` 清掉后需重装）；octicon 集合包名是单数 `@iconify-json/octicon`；npmjs 可 `curl https://registry.npmjs.org/@iconify-json%2F<包名>` 验证；代理 `10.31.3.34:7892` 间歇性不可用（端口通但超时，等恢复）。

## 前端 git 提交踩坑（都发生过）

1. **工作目录陷阱**：默认 Bash 会话在 Kits-RDEMP，操作前端必须显式 `cd /home/bmc/sd1/CODE/webui-vue-autotest`，忘了会报 pathspec 错误或加错到后端仓库。每次前端 git 操作前先 `pwd`
2. **pre-commit hook 超时**：前端 `git commit` 触发 `pnpm lint-staged` 慢则超时 —— 用 `SKIP_SIMPLE_GIT_HOOKS=1 git commit ...` 跳过；超时后先 `ps` 查 commit 是否真创建（`git log --oneline -1`），别盲目重跑
3. **分支切换 stash 污染**：跨分支搬改动用 `git stash` 会整体覆盖目标分支（旧分支未合入的新功能会被带进新分支导致 import 崩溃）。切分支后必须 `git diff --stat` + `git grep` 核对只含本次改动
4. **提交范围**：只 add 本次改动源文件；`.env.development`、`.npmrc`、`.codegraph/`、`.playwright-mcp/`、`*.patch` 一律不提交
5. **前后端分支对应**：新需求前后端拉同名 feature 分支，都基于各自 `origin/master`；前端分支拉取时后端已完成不代表前端也拉了

## 前后端分工

能做后端优先后端，前端只做展示——数据截断/格式化、状态映射、姓名解析由后端加字段返回中文；前端只调接口、渲染表格、绑点击。参数校验任一侧做即可，不必两侧都做（如风扇命令队列 10 条上限仅前端实现，后端不做不算缺陷）。
