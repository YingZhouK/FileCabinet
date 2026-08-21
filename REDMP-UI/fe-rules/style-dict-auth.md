# 样式 · 字典 · 权限

前端 reference：UnoCSS 样式速查、常用图标、字典函数格式、getAuth 权限模板。需要时加载。

## UnoCSS 样式速查

| 元素 | 类 | 说明 |
|---|---|---|
| 卡片圆角 | `rounded-24` | 24px |
| 按钮图标间距 | `mr-4` | 4px 右边距 |
| 按钮间距 | `mr-12` | |
| 栅格间距 | `x-gap="24"` | 24px 列间距 |
| 卡片头部图标 | `h-20 w-20 rounded-6` | |
| 统计数字 | `text-30 font-bold font-din` | 30px 加粗等宽 |
| 进度条卡片 | `rounded-14 shadow-[0_4px_12px_rgba(0,0,0,0.15)]` | |
| 边框 | `border-1 border-light_border dark:border-dark_border` | 响应式 |
| 统计卡片内边距 | `px-24 py-16` | |
| 快捷入口图标 | `h-48 w-48 rounded-14` | |
| 项目首字母 | `h-28 w-28 rounded-8` | |
| 滚动区域 | `max-h-60vh` | |
| 垂直布局 | `n-flex vertical` | |

## 常用图标

| 用途 | 类名 |
|---|---|
| 编辑 | `i-material-symbols:edit-outline` |
| 取消 | `i-material-symbols:cancel-outline` |
| 保存 | `i-material-symbols:save-outline` |
| 刷新 | `i-material-symbols:refresh` |
| 图表 | `i-material-symbols:bar-chart` |
| 文件夹 | `i-material-symbols:folder-open` |
| 列表 | `i-material-symbols:list` |
| 信息 | `i-material-symbols:info` |
| 标记 | `i-material-symbols:flag` |
| 搜索 | `i-material-symbols:search` |
| 警告 | `i-carbon:warning` |
| 对勾 | `i-carbon:checkmark` |
| 时间 | `i-carbon:time` |
| 合并 | `i-carbon:merge` |
| 加号 | `i-fe:plus` |

## 字典函数格式

放 `src/utils/dict.js` 末尾新增（不删改已有）；`type=0` 返回 options 供下拉，`type=1` 返回 `[label, tag, color?, btnNames?, roleBtnNames?]`：

```javascript
export function {DICT_NAME}(type = 0, val) {
  const options = [
    { label: '选项1', value: 1, tag: 'success', color: '#xxx', btnNames: ['Edit', 'Cancel'], roleBtnNames: ['Review'] },
  ]
  if (type === 0) return options
  if (type === 1) {
    if (isNullOrWhitespace(val)) return ['暂无', 'default', null, ['', ''], ['', '']]
    const f = options.find(x => x.value === val)
    return [f?.label || '暂无', f?.tag || 'default', f?.color, f?.btnNames || ['', ''], f?.roleBtnNames || ['', '']]
  }
}
```

选项属性：`label` 显示 / `value` 值 / `tag` naive tag 类型 / `color` 自定义色 / `btnNames` 创建人按钮 / `roleBtnNames` 审批人按钮 / `sort` 可选排序。

按钮名：Edit/Cancel/Submit/Delete（创建人）；Review/Approve/Reject（审批人）。

## 权限模板 getAuth

```javascript
import { useUserStore } from '@/store'
export function getAuth(row, modalAction) {
  const isAdd = modalAction === 'add'
  const userStore = useUserStore()
  const isCreator = row.create_user_id === userStore.user_id
  const isHandler = row.handler_id === userStore.user_id
  const isReviewer = row.reviewer_ids?.includes(userStore.user_id)
  if (modalAction === 'actions') return { isCreator, isHandler, isReviewer }
  if (isAdd) return { enable: true, isCreator, isHandler, isReviewer }
  // 按 row.status 与角色返回各字段 enable / 整体 enable
  if ([状态A, 状态B].includes(row.status)) {
    if (isCreator) return { field1: true, field2: true, enable: false, isCreator, isHandler, isReviewer }
  }
  if (isReviewer && row.status === 审批通过状态) {
    return { field3: true, field4: true, enable: false, isCreator, isHandler, isReviewer }
  }
  return { enable: false, isCreator, isHandler, isReviewer }
}
```

返回属性：`enable` 整体可编辑 / `isCreator` 创建人 / `isHandler` 处理人 / `isReviewer` 审批人 / `{field}` 各字段可编辑。
