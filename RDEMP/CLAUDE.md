# BMC-RDM 开发工作流

---

## 一、开发检查清单

### 新需求启动前

```bash
# 1. 同步前端仓库（保留本地配置）
cd /home/bmc/sd1/CODE/webui-vue-autotest
git pull --rebase
# 确认已有本地配置文件:
#   .env.development — VITE_PROXY_TARGET='http://127.0.0.1:8077'
#   .npmrc — registry=https://registry.npmjs.org

# 2. 同步后端仓库
cd /home/bmc/sd1/CODE/Kits-RDEMP
git pull --no-edit
```

> 前端本地调试配置 (.env.development, .npmrc, .codegraph/) 禁止删除或提交。

### 代码修改后验证（编译 → 启动 → 调接口）

```bash
# 1. 编译
cd /home/bmc/sd1/CODE/Kits-RDEMP
go build -o BMC-RDM .

# 2. 重启（pkill -f 会命中自身 shell 导致 exit 144，改用精确匹配或容忍失败后单独启动）
pkill -x BMC-RDM 2>/dev/null || true   # -x 精确匹配二进制名，避免 exit 144
sleep 1
# ⚠️ 必须剥离代理环境变量启动：http_proxy 会让 Go client 劫持 Jenkins(10.2.110.15:8080) 请求走代理 → 502 → "invalid Location format"
(setsid nohup env -u http_proxy -u https_proxy -u all_proxy -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY ./BMC-RDM > /tmp/bmc-start.log 2>&1 < /dev/null &)

# 3. 等 HTTP 端口就绪——连测试库后启动阶段跑大量后台任务（串口采集退避重试等），
#    要等 8077 真正监听，不要用固定 sleep 4
timeout 60 bash -c 'until ss -tln 2>/dev/null | grep -q :8077; do sleep 2; done'

# 4. 登录拿 Token
TOKEN=$(curl -s -X POST http://127.0.0.1:8077/bmc/rdm/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"eric_zhou","password":"123456"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['accessToken'])")

# 5. 调接口验数据
curl -s http://127.0.0.1:8077/bmc/rdm/<接口> \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{...}' | python3 -m json.tool | head -30
```

> 禁止改完代码直接让用户刷新浏览器看效果。

### 提交代码

```bash
git push origin HEAD:refs/for/master
```

- commit message 写中文，格式: `模块: 描述`
- 只提对外可见的功能交付（新功能、新接口、架构变更、影响用户感知的性能优化）
- 不提开发过程中的琐碎修复和参数调整
- 提交前确认 `configs/config.yaml` 没有临时调试修改混入

---

## 二、环境与配置

### 数据库

| 项目 | 值 |
|------|-----|
| Host | 10.17.152.217 |
| Port | 5432 |
| 账号/密码 | postgres / 123456 |
| 库名 | testdatabase |

> config.yaml 中的 `postgresql` 段默认指向本地 `127.0.0.1:5432`（空库，仅用作启动验证）。本地调试时需要将 host 和 password 改为远程数据库的值。
> 提交代码前必须还原 `configs/config.yaml`，不要提交调试配置到仓库。

### 本地验证

| 项目 | 值 |
|------|-----|
| 后端端口 | 8077 |
| 用户名 | eric_zhou |
| 密码 | 123456 |
| 登录接口 | POST /bmc/rdm/auth/login |
| 鉴权方式 | Bearer Token (Header: Authorization) |

### Artifactory

| 项目 | 值 |
|------|-----|
| Host | 10.32.129.210:8081 |
| 账号/密码 | autotest / At@241106 |

### 生产环境

| 项目 | 值 |
|------|-----|
| 前端 | `http://10.32.129.210:3200/` |
| API | `http://10.32.129.210:3200/api/bmc/rdm/` |
| 登录接口 | `POST /api/bmc/rdm/auth/login` |
| 用户名/密码 | eric_zhou / 123456 |

> 生产环境数据完整，可用于查看页面结构、调用真实 API 获取数据辅助开发。

### 测试环境部署（本机）

前后端都部署在本机，浏览器访问 `http://localhost:8088/`（hash 路由 `/#/`）。

**后端**（连测试库 `10.17.152.217`）：
```bash
cd /home/bmc/sd1/CODE/Kits-RDEMP
# configs/config.yaml 的 postgresql 段改成测试库（见上方「数据库」表，调试期勿还原）
go build -o BMC-RDM .
# ⚠️ 启动必须剥离代理环境变量（http_proxy 劫持 Jenkins 请求 → 502），见「代码修改后验证」
(setsid nohup env -u http_proxy -u https_proxy -u all_proxy -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY ./BMC-RDM > /tmp/bmc-start.log 2>&1 < /dev/null &)
timeout 60 bash -c 'until ss -tln 2>/dev/null | grep -q :8077; do sleep 2; done'
```

**前端**（dev server 连本机后端）：
```bash
cd /home/bmc/sd1/CODE/webui-vue-autotest
# .env.development 的 VITE_PROXY_TARGET 须为 http://127.0.0.1:8077（本机后端；曾误为 10.17.145.120 不可达）
# VITE_SERVER_PORT=8088（8080 被 VSCode server 占用）
(setsid nohup npm run dev > /tmp/webui-test-deploy.log 2>&1 < /dev/null &)
sleep 12
```

验证链路：`curl http://127.0.0.1:8088/` 返回 200，且经前端 proxy 调登录 `POST :8088/api/bmc/rdm/auth/login` 得 `code=00000`。

---

## 三、调试注意事项

### configs/config.yaml 临时修改

本地调试有时需要改 `configs/config.yaml` 的数据库 host/密码指向远程：

- 调试/验证阶段**不要主动还原**，也不要在验证完成后替用户 restore/checkout 该文件 —— 用户可能还要继续验证；除非用户明确要求还原，否则保持指向远程测试库的状态
- 提交代码时：若只是临时改的 host/密码，**不要提交该文件**（`git add` 只加具体代码文件路径，config 保持工作区改动状态、不 stage；必要时 `git restore --staged configs/config.yaml` 移出暂存）
- 禁止用 `git checkout configs/config.yaml` 一类命令替用户还原工作区 —— 会丢用户还在用的调试状态

### updatemachine 配置 (⚠️ 非必要不要设为 false)

```yaml
updatemachine: true  # true=不更新, false=更新
```

`updatemachine: false` 时会启用以下定时任务和功能：

| 影响 | 说明 |
|------|------|
| 光圈消息推送 | 消息真实发送到 Guangquan（`true` 时仅打日志不发送） |
| 机器状态定时更新 | 每 10 分钟执行 `UpdateMachineStatus`，从机器 BMC 拉取在线/离线状态 |
| Zentao 任务同步 | 每 10 分钟执行 `ZentaoUpdateForNeedsAndBugs`，同步 Zentao 需求/Bug 状态 |
| 用户工时同步 | 每 3 分钟执行 `GetAllUserZentaoEffortData`，同步 Zentao 工时数据 |

- 只有明确需要测试以上功能时，才临时设为 `false`
- 测试完成后**必须立即还原**为 `true`
- 可以通过 `tail -f logs/bmcrdm.log | grep -i guangquan` 单独验证光圈日志，不强制关闭

---

## 五、Agent Skills

- **Issue tracker** — Issues tracked as local markdown files under `.scratch/<feature-slug>/`. See `.claude/agents/issue-tracker.md`.
- **Triage labels** — Uses five default triage labels (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `.claude/agents/triage-labels.md`.
- **Domain docs** — Single-context layout: one `CONTEXT.md` + `.claude/adr/` at repo root. See `.claude/agents/domain.md`.
