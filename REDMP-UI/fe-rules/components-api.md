# 组件 · API · Store

前端开发 reference：naive 组件清单、API 风格、store 模板。需要时加载。

## 组件使用

优先复用已有组件（表 2.2）；naive UI 一律 template 用 kebab-case、script 用 CamelCase。

| 组件 | 路径 | 用途 |
|---|---|---|
| CommonPage | `@/components/common/CommonPage.vue` | 页面容器 |
| MeCrud | `@/components/me/crud/index.vue` | CRUD 表格组件 |
| MeQueryItem | `@/components/me/crud/QueryItem.vue` | 查询条件项 |
| MeModal | `@/components/me/modal/Index.vue` | 弹窗组件 |
| AppCard | `@/components/common/AppCard.vue` | 卡片容器 |

### Naive UI 组件（template kebab-case）

n-form/n-form-item · n-input · n-select · n-data-table · n-button · n-modal · n-grid/n-gi · n-radio-group/n-radio-button · n-date-picker · n-switch · n-text · n-tag · n-card · n-descriptions/n-descriptions-item · n-tabs/n-tab-pane · n-timeline/n-timeline-item · n-avatar · n-tree-select · n-upload · n-progress · n-spin · n-empty · n-alert · n-tooltip · n-popconfirm · n-dropdown · n-pagination · n-flex · n-scrollbar · n-number-animation

## API 风格

统一：

| 操作 | 方法 | 路径 | 说明 |
|---|---|---|---|
| 列表查询 | POST | `/{module}/filter` | 分页查询 |
| 获取详情 | GET | `/{module}/getbyid/{id}` | 单条 |
| 创建 | POST | `/{module}/create` | 新增 |
| 更新 | PUT | `/{module}/update/{id}` | |
| 删除 | DELETE | `/{module}/delete/{id}` | |
| 列表不分页 | GET | `/{module}/getall` | |
| 审核 | POST | `/{module}/audit/{id}` | 会记录操作日志 |

## Store 模板

API 请求统一放 `src/store/modules/{module}.js`：

```javascript
import { request } from '@/utils'
import { defineStore } from 'pinia'

export const use{Module}Store = defineStore('{module}', {
  state: () => ({ list: [], detail: null, workLog: [], loading: false }),

  actions: {
    /** 获取列表（分页） @param {object} params @returns {Promise<{pageData,total}>} */
    async getList(params) {
      this.loading = true
      try {
        const res = await request.post('{module}/filter', params)
        this.list = res.data?.pageData || []
        return res.data
      } finally { this.loading = false }
    },

    /** 获取列表不分页 @returns {Promise<Array>} */
    async getAll() {
      const res = await request.get('{module}/getall')
      return res.data || []
    },

    /** 获取详情 @returns {Promise<object>} */
    async getDetail(id) {
      const res = await request.get(`{module}/getbyid/${id}`)
      this.detail = res.data
      this.workLog = res.data?.work_log || []
      return res.data
    },

    /** 创建 @returns {Promise<object>} */
    async add(data) { return request.post('{module}/create', data) },

    /** 更新 @returns {Promise<object>} */
    async update(id, data) { return request.put(`{module}/update/${id}`, data) },

    /** 删除 @returns {Promise<object>} */
    async delete(id) { return request.delete(`{module}/delete/${id}`) },

    /** 审核（记录操作日志） */
    async audit(id, operationLog) { return request.post(`{module}/audit/${id}`, operationLog) },
  },
})
```
