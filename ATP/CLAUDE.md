
# ATP 自动化测试开发工作流规范 (v2.0)
你是一位资深的 OpenBMC 自动化测试架构师。在生成、重构或优化代码时,必须严格遵守以下"环境-规范-重构"三位一体的终极法则。
---
## 2. Python 代码开发标准 (API & Path)
### 2.1 路径注入与导入规范
所有新生成的 Python 测试代码,必须在文件头部包含以下路径注入逻辑,以确保 `lib` 目录被正确识别:
```python
import os, sys
from robot.libraries.BuiltIn import BuiltIn
# 动态注入 lib 路径以支持跨模块识别
lib_path = os.path.join(BuiltIn().get_variable_value("${EXECDIR}"), "lib")
if lib_path not in sys.path:
    sys.path.insert(0, lib_path)
import robot_utils
from bmc_redfish_utils.bmc_redfish_utils import bmc_redfish_utils
```
### 2.2 配置注释标准化 (Schema 契约)
在定义 `PROFILES` 或类似的配置数组时,**严禁**在字典内部写零散注释。必须在 `PROFILES = [` 的正上方建立一个 **「Schema 定义契约」** 块注释:
```python
# [PROFILES Schema 定义契约]
# - matchers: list[tuple] - 基于 (PROJECT_NAME, CID) 的环境匹配规则(固定字段)。
# - <custom_field>: ... - 根据用例需求自定义的业务配置字段。
PROFILES = [ ... ]
# 获取配置:
cfg = robot_utils.get_matched_profile(PROFILES)
```
**规范要求:**
- **固定逻辑**: 所有配置必须包含 `matchers`,并通过 `robot_utils.get_matched_profile` 进行环境适配。
- **动态扩展**: 除 `matchers` 外,其余字段完全由当前测试用例的业务逻辑决定(如 URL、Locator、Payload 等),严禁在规则中预设不相关的通用字段。
- **协议配置规范**: 涉及到 API 接口配置时,需要固定配置 `protocol` 字段(如 `"rest"` 或 `"redfish"`),这是因为底层通信组件需要根据协议类型实例化对应的会话管理器。
  示例:
  ```python
  {
      "protocol": "redfish",
      "apis": {
          "get_audit_log": {
              "method": "GET",
              "url": "/redfish/v1/Managers/{MANAGER_ID}/LogServices/Oem/Bytedance/AuditLog/Entries"
          }
      }
  }
  ```
---
## 3. 重构与架构核心法则 (Refactoring & Architecture)
### 3.1 架构与封装 (KISS 原则)
- **扁平化函数**: 严禁为了暴露 Robot Keyword 而使用 `class` 包装。直接编写顶层函数,并使用 `@keyword("Name")` 装饰器。
- **高内聚**: 像"性能测试"、"基础测试"应聚合在同一个文件中,避免一个用例一个文件的碎片化。
- **局部化配置**: `PROFILES` 及其关联的数据字典必须维护在具体的 Keyword 函数内部,**严禁**在全局作用域声明,防止跨模块命名冲突。
### 3.2 数据驱动与智能感知
- **智能匹配**: 严禁手动解析 Project/CID。统一使用 `robot_utils.get_matched_profile(PROFILES)` 自动抓取环境变量匹配。
- **无感渲染**: 一些通用的配置(如 `/redfish/v1/Managers/{MANAGER_ID}`)可以不使用硬编码,使用占位符,并调用 `cfg = robot_utils.render_template(raw_cfg, {})`。该函数会自动向 Robot Framework 全局提取同名变量(如 `${MANAGER_ID}`)。
### 3.3 执行引擎:拒绝硬编码睡眠
- **状态机探测**: 严禁使用 `time.sleep()` 进行死板等待。必须使用显式等待机制(如 Selenium 的 `WebDriverWait` 或自定义的状态轮询循环)。
### 3.4 统一通信组件约束
- **IPMI 通信**: 严禁自行拼接 `ipmitool raw`。必须调用 `lib/ipmi_client.robot` 中的:
  - 带内: `Run Inband IPMI Standard Command`
  - 带外: `Run External IPMI Standard Command`
- **Redfish/REST 通信**: 必须使用 `lib/bmc_redfish_utils/bmc_redfish_utils.py` 提供的统一客户端。
  **初始化规范**:
  ```python
  from bmc_redfish_utils.bmc_redfish_utils import bmc_redfish_utils
  # 获取环境变量
  host = BuiltIn().get_variable_value("${OPENBMC_HOST}")
  username = BuiltIn().get_variable_value("${OPENBMC_USERNAME}")
  password = BuiltIn().get_variable_value("${OPENBMC_PASSWORD}")
  https_port = BuiltIn().get_variable_value("${HTTPS_PORT}")
  # 实例化客户端（protocol 从配置中读取）
  mgr = bmc_redfish_utils(host, username, password, port=https_port, protocol=cfg["protocol"])
  mgr.login()
  try:
      # 执行请求（仅支持 GET/POST/PATCH/DELETE，默认 GET）
      resp = mgr.get(url)           # GET 请求
      resp = mgr.post(url, json={}) # POST 请求
      resp = mgr.patch(url, json={})# PATCH 请求
      resp = mgr.delete(url)        # DELETE 请求
  finally:
      try:
          mgr.logout()
      except Exception:
          pass
  ```
  **配置驱动示例**:
  ```python
  {
      "protocol": "redfish",  # 或 "rest"
      "apis": {
          "get_platform": {
              "url": "/platform_oem.json",
              "key_list": ["platform_oem"]
          }
      }
  }
  ```
  - ✅ 正确: 通过配置驱动 protocol 字段，使用 bmc_redfish_utils 统一客户端
  - ❌ 错误: 硬编码协议判断、裸写 Requests、使用旧版 `bmc_redfish.py`
---
## 4. 代码质量与日志规范
### 4.1 纯净代码与透明化
- **剔除默认值**: 调用 `get_variable_value` 时,**不要**自作聪明加默认值(如 `"root"`)。依赖缺失应立即抛出异常,拒绝静默失败。
- **精准变量名**: 获取变量时使用全局精确匹配名(如 `${username}`),不要重命名为难看的别名。
- **注释修剪**: 删除解释"代码执行逻辑"的注释(如"拿到配置"),只保留解释"业务流派异常"或"字典约定"的注释。
### 4.2 列表寻址与解析
- **解析真理**: 针对列表寻址,必须通过 `robot_utils.get_dict_value` 配合过滤器字典(如 `{"Key": "Value"}`)进行精准定位。
### 4.3 纯业务日志面具 (`step`)
- **非上下文管理器**: `robot_utils.step` 是普通打印函数,**不准**作为 `with step(...):` 使用。
- **文案规范**:
  - ❌ 错: `step("Sending JSON to URL...")`
  - ✅ 对: `step("Sending BMC restart command")` (业务驱动)
---
## 5. Robot 用例层元数据规范
### 5.1 文档 (Documentation)
`[Documentation]` 必须遵循 Markdown/List 模块格式,使用**英文半角标点**,严禁中文全角标点。
```robotframework
[Documentation]
...    | ----------------------------------------------------------
...    | 【元数据】
...    | 用例作者: 周颖
...    | 测试说明: 验证系统重启后的SEL日志记录
...    | ----------------------------------------------------------
...    | 【测试步骤】
...    | 1. 查询系统当前 SEL 总条数, 预期结果1
...    | 2. 下发专业动作指令, 预期结果2
...    | ----------------------------------------------------------
...    | 【预期结果】
...    | 1. 在180s阈值内系统完成上线自检
...    | 2. SEL 条数增加且包含重启记录
```
### 5.2 标签 (Tags)
- **绝对一致**: `[Tags]` 必须与用例名称完全一致,空格替换为下划线。
- 示例: 用例 `Verify Add And Clear SEL` -> 标签 `[Tags]  Verify_Add_And_Clear_SEL`
---
## 6. 异常处理与健壮性规范
### 6.1 EAFP 原则
- **底层函数**: 失败直接抛出异常,严禁返回错误字典。
- **上层捕获**: 通过 `try-except` 捕获异常,区分预期失败与系统故障。
### 6.2 拒绝静默失败
- **严禁空 except**: 严禁使用空的 `try...except: pass`。关键业务中异常应向上抛出或记录明确日志。
- **JSON 解析安全**: `resp.json()` 必须包裹在 `try-except` 中捕获 `ValueError`。
### 6.3 严格空值判断
- **禁止隐式判断**: 严禁使用 `if not result:`,应使用 `if result is None:`。
---
## 8. 接口设计与复用规范
### 8.1 参数最小化
- **核心接口**: 仅接收必要配置字典和执行参数,内部自治完成分析器创建、登录等流程。
### 8.2 优先复用
- **统一工具**: 优先检查并复用项目已有通用工具(如 `get_dict_value`、`execute_bmc_request`、`render_template`),避免重复造轮子。
- **配置驱动**: 在进行代码重构时,应优先使用项目提供的统一工具函数替代手动的反射调用或分散的请求逻辑。
---
## 9. 错误消息与日志编写规范
### 9.1 用词准确
- **技术术语**: 避免使用生造或文学化词汇(如"裂化"、"破发"),需使用清晰、直白、技术准确的描述(如"断言失败"、"索引越界"、"类型不匹配")。
### 9.2 极致简洁
- **核心信息**: 验证失败时**仅**打印核心关键信息,**禁止**打印额外的上下文信息或构建复杂的路径字符串。
### 9.3 避免冗余
- **统一处理**: 对于重复出现的错误抛出逻辑,应提取为统一的辅助函数或处理机制,避免在多处重复编写相同的 `raise` 语句。

---
## 10. Git 提交规范

### 10.1 "rebase" 定义
当我说 **rebase** 时，意思是使用 `git rebase -i` 进行交互式变基操作（通常是 `git rebase -i HEAD~N` 合并提交），**不是** `git reset --soft` + 重新 commit。远程已经存在的提交记录，必须通过 rebase 合并/整理，不能 soft reset 重来。

### 10.2 提交前确认 commit message
commit 前必须把 message 展示给用户确认后再提交。


