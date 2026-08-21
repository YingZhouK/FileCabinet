## 完整交付规范

### 分层完成的硬标准

写分层代码（Entity/Repository/Service/Controller）时，**所有层必须一次交付，漏任何一层都算本轮未完成**。Controller 不是"下一步"，是完成条件的组成部分——Service/Logic 写完但 Controller 没写，等于活没干完。

- 正向目标：交付 = 全部相关层 + 编译通过
- 每层结束自带完成检查：这层能否被上一/下一层调用，接口契约是否闭合

### 长任务拆解（预防烂尾）

上下文越长越易在末尾断掉。单个请求只做完整交付的某一段，段与段之间明确宣告已交付/待交付：

- 请求范围过大时，先拆成次序明确的小步（如"Entity+Repository"→"Service"→"Controller"）
- **每段结束时显式宣告**："已完成 X，剩余 Y"——不得静默停在半途等用户追问
- 若长度/上下文受限导致某层（尤其 Controller）无法在本轮交付：**明说"Controller 在下一步生成"，并先交付已完成部分的正确逻辑**，不得直接停止或留下依赖悬空的未接线代码

### 结尾自检

结束回答前检查：本次涉及的分层代码是否每层都交付？Controller 是否遗漏？若有未完成层，宣告清楚而非假设用户会发现。

## 迭代开发流程

### 问题反馈处理原则

用户报告操作/页面/接口有问题时：

1. **先验证，后归因** — 立即复现或查证（日志、接口响应、Network/Console），先假设问题真实存在
2. 验证通过后再说明"不是问题"，并给出用户可自行复核的证据（状态码、日志行、复现步骤）
3. 禁止用"时序问题""操作方式不对"这类未经验证的解释搪塞；确属环境/时序导致，也要先复现出证据链再下结论

### 轨迹记录

每次迭代结束时，将本次改动 checkpoint 到本地 git，防止后续误操作破坏文件：

```bash
git add -A
git commit -m "迭代 checkpoint: <本次做了什么>"
```

**仅添加实际改动的代码文件**，禁止提交无关产物：

- ❌ 编译产物、日志文件、静态资源构建产物、前端本地调试配置、临时脚本——具体名单因项目而异，以项目 CLAUDE.md 为准
- ❌ IDE / 工具元数据（所有项目通用）：`.claude/`, `.codegraph/`, `.playwright-mcp/` 等

提交前用 `git status` 确认 staging 只有本次改动的文件。发现无关文件已暂存时，用 `git restore --staged <path>` 移除。不在磁盘上删除这些文件。

在写 checkpoint message 时可以写中文描述，不必像最终提交那样精简——这是给回溯看的，不是对外交付的。

### 交付契约

最终交付时，将所有 checkpoint **碾平成一条 commit**：

1. 先生成完整 patch（防止合提交后丢了审计痕迹）：
   ```bash
   git diff <首个 checkpoint 的父 commit SHA> HEAD > feature-full.patch
   ```
2. 碾平：
   ```bash
   git reset --soft <首个 checkpoint 的父 commit SHA>
   git commit -m "<模块>: <对外可见的功能交付>"
   ```
3. 提交最终 commit（见上一条规范）。推送目标分支因仓库而异，按项目实际配置执行。

最终 commit message 只能写对外可见的交付，不提开发过程中的琐碎修复和参数调整：
- ✅ 新功能/新接口/新页面
- ✅ 架构变更（如存储方案替换）
- ✅ 影响用户感知的性能优化
- ❌ 代码审查修复（如变量外提、`ShouldBindQuery`）
- ❌ 开发过程中的参数调整（如保留天数、超时时间）
- ❌ 配置文件的临时修改
- ❌ 文档更新（除非该文档本身是交付物）

禁止在 commit 末尾添加 `Co-Authored-By: Claude ...` 行。

### 示例

```bash
# 第一次迭代
git add <本次改动的文件>
git commit -m "迭代 checkpoint: PMM 页面框架搭建"

# 第二次迭代
git add <本次改动的文件>
git commit -m "迭代 checkpoint: PMM 详情页流程进度"

# 最终交付
git diff HEAD~2 HEAD > pmm-feature.patch
git reset --soft HEAD~2
git commit -m "PMM: 新增需求列表与详情页"
```

### Patch 生成

- 使用原生 `git diff` 或 `rtk proxy git diff`，禁止用 RTK 压缩/裁剪后的 patch
- `git reset --soft` 之前**必须**先 `git diff > patch` 导出完整文件
- **碾平成一笔提交后，中间过程产生的 patch 一律删除、不要保留**。patch 只是碾平前的临时审计产物（含多笔 checkpoint 的 diff），交付完成就失去存在意义；留在工作区只会污染 `git status`、易被误提交。碾平后清理工作区的 `*.patch` 文件（前后端仓库的根目录）

### 将当前分支更新到最新远程分支

用户说「把当前分支更新到最新」（或等价表述）时，含义是：**把当前分支的已提交改动搬运到最新 `origin/<主分支>` 之上**。用 rebase 而非 merge——纯线性历史、改动放最新主分支顶部、无合并提交；两线同基点时 rebase 无冲突。

**通用流程**（每仓库各做一遍，多个仓库互不 merge）：

```bash
# 0. 先厘清：当前分支已提交了哪些改动、基点在哪
git merge-base HEAD origin/<主分支>

# 1. 拉最新远程（只 fetch，别急着 pull——pull 可能因本地未提交改动/分叉冲突）
git fetch origin

# 2. 备份安全网（防不可逆事故），用原生 git diff（见「Patch 生成」节，禁用 RTK 压缩）：
#    a. 当前分支已提交内容（相对基点）
git diff <merge-base> HEAD > /tmp/<功能>-commit.patch
#    b. 工作区未提交改动（若有，如调试配置文件）
git diff <未提交的已跟踪文件> > /tmp/<功能>-wt.patch

# 3. rebase 要求工作区干净——先把工作区未提交的已跟踪文件 stash 隔离（未跟踪目录如 .claude/ 不受影响）
git stash push -m "<功能>-rebase-wt" -- <未提交的已跟踪文件>
git status --short | grep -v '^??'   # 确认已跟踪文件干净

# 4. rebase 到最新主分支
git rebase origin/<主分支>

# 5. 恢复工作区杂项（rebase 不换分支，stash pop 同分支路径匹配，无污染）
git stash pop

# 6. 验证：按仓库声明方式编译 + 测试，确认改动在新 base 上正常、与远程新功能无冲突
# 7. 报告结果；除非用户明确要求，否则不 push（推送目标按项目实际配置）
```

**要点**：
- **先备份再 rebase**：当前分支已提交内容 + 工作区杂项双层 patch 备份，`/tmp` 留存，推送完成前不删。
- **工作区杂项（未提交的二进制/配置/日志/调试文件）必须保留**：rebase 前 stash 隔离，rebase 后 pop 原样恢复，不要丢弃。这些本就不该随分支走，也不阻塞 rebase。
- **rebase 只搬已提交改动**：未跟踪目录不受影响，无需处理。
- **冲突处理**：rebase 中冲突时停下来解决，别 `--abort` 丢弃；同文件不同函数区域通常自动三路合并。
- **主分支名因仓库而异**：`master`/`main` 皆可，以远程实际为准（`git rev-parse --abbrev-ref @{upstream}` 或查看 `origin/*`）。

---

## 新需求开发前：先问分支

**每个新需求启动前，必须先询问用户是否新拉分支，再动手开发**，不可默认在现有分支直接开干：

1. 先向用户确认：这个需求做在哪个分支？
   - 新拉独立 feature 分支（听用户给名，如 `feature/xxx`）
   - 复用现有分支
   - 直接在 master / 当前分支
2. 确认后再开始编码；未确认不生成本功能代码
3. 若多个仓库（前后端）都涉及，分支策略统一确认（通常同名分支两端各建）

---

## 沙箱纪律

### 手术刀原则

修改已有文件时只动业务改动的行，不触碰无关行：

- **不改变文件元数据** — 换行符（CRLF/LF）、缩进风格、尾部空白、编码格式、末尾空行一律保持原样
- **不重写文件** — diff 不应出现整个文件被重写的假象
- 批量操作前，先用 `git show HEAD:<path>` 确认基准版本的精确内容，再逐行追加/插入

### 文件恢复

当编辑文件失败后，禁止用 `git checkout HEAD -- .` 或 `git checkout HEAD -- <dir>` 清理工作区——这会连带删除其他文件的未暂存修改。

有 checkpoint 兜底时，直接丢弃当前工作区恢复到最后一次 checkpoint：

```bash
git checkout -- .           # 放弃所有未暂存修改
git reset --hard HEAD       # 回到最近一次 commit（即上次 checkpoint）
```

没有 checkpoint 时（刚改了几行但没来得及 commit）：
- 恢复单个文件：`git checkout HEAD -- <具体文件路径>`
- 逐块修复：用 `Edit` 工具

---

## 模块化开发

按职责拆分文件，避免把功能全部塞进一个文件，导致后续重构困难。

### 文件规模红线

| 语言 | 警戒线 | 说明 |
|------|--------|------|
| Go（controller/service 层） | 单文件 > 400 行 | 超过即考虑按职责拆文件（同包跨文件调用零成本，编译期保证正确） |
| Vue（页面组件） | 单文件 > 800 行 | 超过即拆子组件（列表/详情/表单独立成组件，props/emit 传递状态） |
| JS/TS（composable/store） | 单文件 > 300 行 | 超过即按功能拆分模块 |

### 拆分原则

- **按职责拆**：一个文件只做一类事（如 controller 按 Token/代理/过滤/映射/权限拆），不按"顺眼"拆
- **同包优先**：Go 同包跨文件调用/包级变量共享零成本——拆文件是搬家不是重构，风险低
- **包级状态集中**：跨函数共享的缓存/锁放同一文件（或明确归属），避免散落
- **新增代码从第一天拆**：新模块先规划文件结构再写，不写大再拆（拆的成本永远高于一开始分好）
- **停用页/死代码直接删**：不保留"以后可能用"的重复逻辑，删除自然消解重复

### 什么时候不拆

- 功能已交付稳定、近期不再频繁改动时，拆是纯重构无功能收益——记 TODO 等稳定后专项拆
- 拆组件（Vue）风险高于拆文件（Go）——Vue 组件间状态传递拆错是运行时错误，先出拆解方案确认再动

---

## 网络代理

安装系统包、npm 包、pip 包等走代理：

```bash
http_proxy=http://10.31.3.34:7892 https_proxy=http://10.31.3.34:7892 <command>
```

apt 代理已配在 `/etc/apt/apt.conf.d/proxy.conf`：

```
Acquire::http::Proxy "http://10.31.3.34:7892";
Acquire::https::Proxy "http://10.31.3.34:7892";
```

---

## 前端 pnpm 环境（webui-vue-autotest 踩坑）

### 重装 node_modules 会全线 build 崩——靠全 hoist 兜底

该前端项目**不声明多个直接依赖却 import 它们**（`lodash` 27+ 处、`highlight.js`、`@iconify/utils` 等），历史上一直靠 `shamefully-hoist` 把它们 hoist 到顶层才可解析。**一旦重装 node_modules，这些未声明依赖全挂**，build 报 `Cannot find package X` / `Rollup failed to resolve import`。

- **pnpm 11 不再读取 `.npmrc` 的 `shamefully-hoist`**（启动即 `npm warn Unknown project config "shamefully-hoist"`）。旧 `.npmrc` 那条是死的。
- 恢复可 build 布局需在**本地未跟踪文件** `pnpm-workspace.yaml` 写 `shamefullyHoist: true`（workspace 配置位）+ `allowBuilds` 对 esbuild / simple-git-hooks / vue-demi / core-js / es5-ext 置 `true`（pnpm 10+/11 默认忽略 build scripts，否则 esbuild 二进制缺失、git hook 不装）。
- 根治方向：把缺失依赖补进 `package.json`（`lodash`、`@iconify/utils` 等），但 `pnpm-lock.yaml` 会大 churn，交付前与用户确认是否纳入。

### 重装前必须停 dev server

dev server（`npm run dev`，8088）占用 node_modules，pnpm 要删 modules 目录重建时无 TTY 会 `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY` 中止。先 `kill <8088 pid>` 再重装，装完重启 dev。

### commit 的 pre-commit hook 会触发 pnpm

前端 `.git/hooks/pre-commit` = simple-git-hooks → `pnpm lint-staged`（`.lintstagedrc` 为 `{"*": "eslint --fix"}`）。node_modules 有漂移时 hook 会先跑 `pnpm install`——别被它带偏，先确认 node_modules 完整。

lint-staged 全库跑会爆**历史既有 lint 债**（如 auto-release/index.vue 的 `autoReleaseAdd no-undef`、CssSyntaxError `Unclosed block`），非本轮引入。要提交自己改动的干净文件时，用 `git commit --no-verify` 绕过（前提：改的文件单独 `npx eslint` 已确认干净）；全库 lint 债另报用户，不顺手修。

---

@RTK.md

<!-- CODEGRAPH_START -->
## CodeGraph

This project has a CodeGraph MCP server (`codegraph_*` tools) configured. CodeGraph is a tree-sitter-parsed knowledge graph of every symbol, edge, and file. Reads are sub-millisecond and return structural information grep cannot.

### When to prefer codegraph over native search

Use codegraph for **structural** questions — what calls what, what would break, where is X defined, what is X's signature. Use native grep/read only for **literal text** queries (string contents, comments, log messages) or after you already have a specific file open.

| Question | Tool |
|---|---|
| "Where is X defined?" / "Find symbol named X" | `codegraph_search` |
| "What calls function Y?" | `codegraph_callers` |
| "What does Y call?" | `codegraph_callees` |
| "How does X reach/become Y? / trace the flow from X to Y" | `codegraph_trace` (one call = the whole path, incl. callback/React/JSX dynamic hops) |
| "What would break if I changed Z?" | `codegraph_impact` |
| "Show me Y's signature / source / docstring" | `codegraph_node` |
| "Give me focused context for a task/area" | `codegraph_context` |
| "See several related symbols' source at once" | `codegraph_explore` |
| "What files exist under path/" | `codegraph_files` |
| "Is the index healthy?" | `codegraph_status` |

### Rules of thumb

- **Answer directly — don't delegate exploration.** For "how does X work" / architecture questions, answer with 2-3 codegraph calls: `codegraph_context` first, then ONE `codegraph_explore` for the source of the symbols it surfaces. For a specific **flow** ("how does X reach Y") start with `codegraph_trace` from→to — one call returns the whole path with dynamic hops bridged — then ONE `codegraph_explore` for the bodies; don't rebuild the path with `codegraph_search` + `codegraph_callers`. Codegraph IS the pre-built index, so spawning a separate file-reading sub-task/agent — or running a grep + read loop — repeats work codegraph already did and costs more for the same answer.
- **Trust codegraph results.** They come from a full AST parse. Do NOT re-verify them with grep — that's slower, less accurate, and wastes context.
- **Don't grep first** when looking up a symbol by name. `codegraph_search` is faster and returns kind + location + signature in one call.
- **Don't chain `codegraph_search` + `codegraph_node`** when you just want context — `codegraph_context` is one call.
- **Don't loop `codegraph_node` over many symbols** — one `codegraph_explore` call returns several symbols' source grouped in a single capped call, while each separate node/Read call re-reads the whole context and costs far more.
- **Index lag — check the staleness banner, don't guess a wait.** When a codegraph response starts with "⚠️ Some files referenced below were edited since the last index sync…", the listed files are pending re-index — Read those specific files for accurate content. Files NOT in that banner are fresh and codegraph is authoritative for them. `codegraph_status` also lists pending files under "Pending sync".

### If `.codegraph/` doesn't exist

The MCP server returns "not initialized." Ask the user: *"I notice this project doesn't have CodeGraph initialized. Want me to run `codegraph init -i` to build the index?"*
<!-- CODEGRAPH_END -->
