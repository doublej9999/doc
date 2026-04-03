# form-create 动态表单教程

这份文档用一个“可落地”的方式讲 form-create 动态表单怎么做，重点覆盖：

- 基础表单搭建
- 常用组件配置
- 外部接口加载数据
- 选人组件通过 API 获取人员列表
- 表格/子表单示例
- 编辑回显与提交数据处理

以下示例以 **Vue 3 + form-create** 为主，接口请求使用 `axios`。

---

## 1. 什么是 form-create

form-create 是一个通过 **JSON 规则配置生成表单** 的动态表单方案。你不用手写大量 `<el-form-item>`，而是通过规则数组来描述表单结构。

适合场景：

- 后台管理系统动态表单
- 流程表单
- 问卷/配置型页面
- 不同业务场景下动态切换字段

核心思路：

- 用 `rule` 定义字段
- 用 `option` 定义表单整体配置
- 用接口动态加载选项、默认值、联动数据
- 最后统一提交表单值

---

## 2. 安装

按你的 UI 组件库安装对应版本。这里以 Element Plus 为例：

```bash
npm install @form-create/element-ui@next axios
```

如果你的项目已经集成了 form-create，对应跳过即可。

在页面中引入：

```vue
<template>
  <form-create
    v-model:api="fApi"
    :rule="rule"
    :option="option"
  />
</template>

<script setup>
import { ref } from 'vue'

const fApi = ref(null)
const rule = ref([])
const option = ref({
  submitBtn: {
    text: '提交'
  },
  resetBtn: true
})
</script>
```

---

## 3. 最基础的动态表单

先看一个最小例子：

```vue
<script setup>
import { ref } from 'vue'

const fApi = ref(null)

const rule = ref([
  {
    type: 'input',
    field: 'name',
    title: '姓名',
    value: '',
    props: {
      placeholder: '请输入姓名',
      clearable: true
    }
  },
  {
    type: 'input',
    field: 'mobile',
    title: '手机号',
    value: '',
    props: {
      placeholder: '请输入手机号'
    }
  },
  {
    type: 'select',
    field: 'gender',
    title: '性别',
    value: '',
    options: [
      { label: '男', value: 'male' },
      { label: '女', value: 'female' }
    ]
  }
])

const option = ref({
  form: {
    labelWidth: '100px'
  },
  submitBtn: {
    text: '提交'
  },
  onSubmit(formData) {
    console.log('提交数据：', formData)
  }
})
</script>
```

提交时 `formData` 大概长这样：

```js
{
  name: '张三',
  mobile: '13800138000',
  gender: 'male'
}
```

---

## 4. 常用组件示例

下面把项目里最常见的组件过一遍。

### 4.1 输入框 input

```js
{
  type: 'input',
  field: 'title',
  title: '标题',
  value: '',
  props: {
    placeholder: '请输入标题',
    maxlength: 50,
    showWordLimit: true
  },
  validate: [
    { required: true, message: '标题必填', trigger: 'blur' }
  ]
}
```

### 4.2 多行文本 textarea

```js
{
  type: 'input',
  field: 'remark',
  title: '备注',
  value: '',
  props: {
    type: 'textarea',
    rows: 4,
    placeholder: '请输入备注'
  }
}
```

### 4.3 数字输入 inputNumber

```js
{
  type: 'inputNumber',
  field: 'age',
  title: '年龄',
  value: 18,
  props: {
    min: 0,
    max: 120,
    precision: 0
  }
}
```

### 4.4 单选 radio

```js
{
  type: 'radio',
  field: 'status',
  title: '状态',
  value: 1,
  options: [
    { label: '启用', value: 1 },
    { label: '禁用', value: 0 }
  ]
}
```

### 4.5 多选 checkbox

```js
{
  type: 'checkbox',
  field: 'hobby',
  title: '兴趣爱好',
  value: [],
  options: [
    { label: '阅读', value: 'read' },
    { label: '运动', value: 'sport' },
    { label: '音乐', value: 'music' }
  ]
}
```

### 4.6 下拉选择 select

```js
{
  type: 'select',
  field: 'deptId',
  title: '所属部门',
  value: '',
  options: [],
  props: {
    placeholder: '请选择部门',
    filterable: true,
    clearable: true
  }
}
```

### 4.7 日期选择 datePicker

```js
{
  type: 'datePicker',
  field: 'entryDate',
  title: '入职日期',
  value: '',
  props: {
    type: 'date',
    valueFormat: 'YYYY-MM-DD',
    placeholder: '请选择日期'
  }
}
```

### 4.8 时间范围 daterange

```js
{
  type: 'datePicker',
  field: 'period',
  title: '有效期',
  value: [],
  props: {
    type: 'daterange',
    startPlaceholder: '开始日期',
    endPlaceholder: '结束日期',
    valueFormat: 'YYYY-MM-DD'
  }
}
```

### 4.9 开关 switch

```js
{
  type: 'switch',
  field: 'enabled',
  title: '是否启用',
  value: true
}
```

### 4.10 上传 upload

```js
{
  type: 'upload',
  field: 'files',
  title: '附件',
  value: [],
  props: {
    action: '/api/upload',
    limit: 3,
    multiple: true
  }
}
```

---

## 5. 表单校验

form-create 可以直接在规则上配置校验：

```js
{
  type: 'input',
  field: 'email',
  title: '邮箱',
  value: '',
  validate: [
    { required: true, message: '邮箱不能为空', trigger: 'blur' },
    { type: 'email', message: '邮箱格式不正确', trigger: 'blur' }
  ]
}
```

手机号自定义校验：

```js
{
  type: 'input',
  field: 'mobile',
  title: '手机号',
  value: '',
  validate: [
    { required: true, message: '手机号必填', trigger: 'blur' },
    {
      validator: (rule, value, callback) => {
        if (!/^1\d{10}$/.test(value)) {
          callback(new Error('手机号格式不正确'))
        } else {
          callback()
        }
      },
      trigger: 'blur'
    }
  ]
}
```

---

## 6. 外部接口加载数据

动态表单最常见的需求，不是写死 options，而是从外部接口获取。

比如：

- 部门列表
- 字典项
- 城市列表
- 用户列表
- 详情回显数据

下面给一个完整示例。

### 6.1 请求部门列表并填充下拉框

```vue
<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const fApi = ref(null)

const rule = ref([
  {
    type: 'select',
    field: 'deptId',
    title: '所属部门',
    value: '',
    options: [],
    props: {
      placeholder: '请选择部门',
      filterable: true,
      clearable: true
    }
  }
])

const option = ref({
  form: { labelWidth: '100px' },
  submitBtn: true
})

async function loadDeptOptions() {
  const res = await axios.get('/api/dept/list')
  const list = res.data?.data || []

  const options = list.map(item => ({
    label: item.deptName,
    value: item.deptId
  }))

  const deptRule = rule.value.find(item => item.field === 'deptId')
  if (deptRule) {
    deptRule.options = options
  }
}

onMounted(() => {
  loadDeptOptions()
})
</script>
```

如果你需要在表单渲染后再更新，也可以通过 `fApi` 操作字段。

例如：

```js
fApi.value.updateRule('deptId', {
  options
})
```

---

## 7. 级联联动：根据部门加载人员

这是很常见的场景：

- 先选部门
- 再加载该部门下的人员

```vue
<script setup>
import { ref } from 'vue'
import axios from 'axios'

const fApi = ref(null)

const rule = ref([
  {
    type: 'select',
    field: 'deptId',
    title: '部门',
    value: '',
    options: [],
    props: {
      placeholder: '请选择部门',
      filterable: true,
      clearable: true
    },
    on: {
      change: async (value) => {
        if (!value) {
          fApi.value.setValue('userId', '')
          fApi.value.updateRule('userId', { options: [] })
          return
        }

        const res = await axios.get('/api/user/list', {
          params: { deptId: value }
        })

        const userOptions = (res.data?.data || []).map(user => ({
          label: user.name,
          value: user.id
        }))

        fApi.value.setValue('userId', '')
        fApi.value.updateRule('userId', {
          options: userOptions
        })
      }
    }
  },
  {
    type: 'select',
    field: 'userId',
    title: '人员',
    value: '',
    options: [],
    props: {
      placeholder: '请选择人员',
      filterable: true,
      clearable: true
    }
  }
])
</script>
```

---

## 8. 选人组件：通过 API 请求人员列表

很多业务里“选人”不是简单下拉框，而是支持搜索、远程加载、多选、回显。最简单的实现方式，是基于 `select` 做一个 **远程搜索选人组件**。

### 8.1 单选选人（远程搜索）

```js
{
  type: 'select',
  field: 'assigneeId',
  title: '负责人',
  value: '',
  options: [],
  props: {
    placeholder: '请输入姓名搜索负责人',
    filterable: true,
    remote: true,
    reserveKeyword: true,
    clearable: true,
    remoteMethod: async (keyword) => {
      const res = await axios.get('/api/user/search', {
        params: {
          keyword
        }
      })

      const options = (res.data?.data || []).map(user => ({
        label: `${user.name} / ${user.mobile} / ${user.deptName}`,
        value: user.id
      }))

      fApi.value.updateRule('assigneeId', { options })
    }
  },
  validate: [
    { required: true, message: '请选择负责人', trigger: 'change' }
  ]
}
```

接口返回建议：

```json
{
  "code": 200,
  "data": [
    {
      "id": "u1001",
      "name": "张三",
      "mobile": "13800138000",
      "deptName": "技术部"
    },
    {
      "id": "u1002",
      "name": "李四",
      "mobile": "13800138001",
      "deptName": "产品部"
    }
  ]
}
```

### 8.2 多选选人

```js
{
  type: 'select',
  field: 'memberIds',
  title: '参与人员',
  value: [],
  options: [],
  props: {
    placeholder: '请输入姓名搜索参与人员',
    filterable: true,
    remote: true,
    multiple: true,
    collapseTags: true,
    clearable: true,
    remoteMethod: async (keyword) => {
      const res = await axios.get('/api/user/search', {
        params: { keyword }
      })

      const options = (res.data?.data || []).map(user => ({
        label: `${user.name}（${user.deptName}）`,
        value: user.id
      }))

      fApi.value.updateRule('memberIds', { options })
    }
  }
}
```

### 8.3 页面初始化时默认拉取一次人员

为了避免用户没输入前没有数据，可以在页面加载后先请求一批热门/最近联系人：

```js
async function loadInitialUsers() {
  const res = await axios.get('/api/user/recent')
  const options = (res.data?.data || []).map(user => ({
    label: `${user.name}（${user.deptName}）`,
    value: user.id
  }))

  fApi.value.updateRule('assigneeId', { options })
  fApi.value.updateRule('memberIds', { options })
}
```

### 8.4 编辑回显：已知 userId，显示用户名称

编辑页经常只有 `userId`，但是选人控件需要显示 label。这时要单独请求详情接口。

```js
async function loadDetail(id) {
  const res = await axios.get(`/api/form/detail/${id}`)
  const detail = res.data?.data

  fApi.value.setValue('assigneeId', detail.assigneeId)
}

async function loadAssigneeOption(userId) {
  const res = await axios.get(`/api/user/detail/${userId}`)
  const user = res.data?.data

  fApi.value.updateRule('assigneeId', {
    options: [
      {
        label: `${user.name} / ${user.mobile} / ${user.deptName}`,
        value: user.id
      }
    ]
  })
}
```

建议顺序：

1. 先请求详情
2. 再请求负责人详情，补充 options
3. 最后 setValue 回显

这样界面显示最稳定。

---

## 9. 外部接口字典统一封装

项目里往往有很多字典项，比如：

- 性别
- 状态
- 审批类型
- 业务分类

建议统一封装成方法：

```js
import axios from 'axios'

export async function getDictOptions(dictType) {
  const res = await axios.get('/api/dict/options', {
    params: { dictType }
  })

  return (res.data?.data || []).map(item => ({
    label: item.label,
    value: item.value
  }))
}
```

使用方式：

```js
const genderOptions = await getDictOptions('gender')
fApi.value.updateRule('gender', { options: genderOptions })
```

这样表单里所有 select/radio/checkbox 都能复用。

---

## 10. 表格表单示例：明细行录入

很多动态表单不只是“普通字段”，还会有“商品明细”“费用明细”“子任务明细”这种表格型输入。

在 form-create 里常见做法有两种：

1. 用 **子表单/数组组件** 管理一组结构化数据
2. 用自定义组件承载可编辑表格

这里先给一个最常用的“明细数组”示例。

### 10.1 明细表单数据结构

比如采购申请明细：

```js
[
  {
    itemName: '显示器',
    qty: 2,
    price: 1200
  },
  {
    itemName: '键盘',
    qty: 3,
    price: 300
  }
]
```

### 10.2 子表单示例（明细表）

```js
{
  type: 'group',
  field: 'items',
  title: '采购明细',
  value: [
    {
      itemName: '',
      qty: 1,
      price: 0
    }
  ],
  props: {
    rule: [
      {
        type: 'input',
        field: 'itemName',
        title: '物品名称',
        props: {
          placeholder: '请输入物品名称'
        },
        validate: [
          { required: true, message: '物品名称必填', trigger: 'blur' }
        ]
      },
      {
        type: 'inputNumber',
        field: 'qty',
        title: '数量',
        value: 1,
        props: {
          min: 1
        }
      },
      {
        type: 'inputNumber',
        field: 'price',
        title: '单价',
        value: 0,
        props: {
          min: 0,
          precision: 2
        }
      }
    ],
    min: 1,
    max: 20,
    addBtnText: '新增明细'
  }
}
```

提交后得到：

```js
{
  items: [
    {
      itemName: '显示器',
      qty: 2,
      price: 1200
    },
    {
      itemName: '键盘',
      qty: 3,
      price: 300
    }
  ]
}
```

### 10.3 计算总金额

如果需要根据明细计算总金额，可以在提交前统一处理：

```js
onSubmit(formData) {
  const totalAmount = (formData.items || []).reduce((sum, item) => {
    return sum + Number(item.qty || 0) * Number(item.price || 0)
  }, 0)

  const payload = {
    ...formData,
    totalAmount
  }

  console.log('最终提交：', payload)
}
```

---

## 11. 可编辑表格方案示例

如果你的需求是“看起来像 table 一样一行一行编辑”，通常建议：

- 用 form-create 管主表单
- 用自定义组件管明细表格
- 最后作为一个字段值回传给表单

比如封装一个 `EditableTable` 组件，然后在 form-create 里这样写：

```js
{
  type: 'component',
  field: 'expenseList',
  title: '费用明细',
  value: [],
  props: {
    is: 'EditableTable',
    columns: [
      { title: '费用项', dataIndex: 'feeName' },
      { title: '金额', dataIndex: 'amount' },
      { title: '备注', dataIndex: 'remark' }
    ]
  }
}
```

`EditableTable` 组件的职责：

- 支持新增/删除行
- 单元格编辑
- 对外通过 `modelValue` / `update:modelValue` 回传数据

这样做的好处是：

- 表格交互更灵活
- 不会把复杂表格逻辑塞进 rule 配置里
- 明细字段仍然能跟主表单一起提交

---

## 12. 完整示例：请假申请表单

这个例子把前面的点串起来：

- 部门从接口获取
- 负责人从 API 远程搜索
- 抄送人支持多选
- 请假类型从字典接口获取
- 明细/附件一起提交

```vue
<template>
  <form-create
    v-model:api="fApi"
    :rule="rule"
    :option="option"
  />
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const fApi = ref(null)

const rule = ref([
  {
    type: 'input',
    field: 'title',
    title: '申请标题',
    value: '',
    props: {
      placeholder: '请输入申请标题'
    },
    validate: [
      { required: true, message: '申请标题必填', trigger: 'blur' }
    ]
  },
  {
    type: 'select',
    field: 'deptId',
    title: '所属部门',
    value: '',
    options: [],
    props: {
      placeholder: '请选择部门',
      filterable: true,
      clearable: true
    },
    validate: [
      { required: true, message: '请选择部门', trigger: 'change' }
    ]
  },
  {
    type: 'select',
    field: 'leaveType',
    title: '请假类型',
    value: '',
    options: [],
    props: {
      placeholder: '请选择请假类型'
    }
  },
  {
    type: 'select',
    field: 'assigneeId',
    title: '审批人',
    value: '',
    options: [],
    props: {
      placeholder: '请输入姓名搜索审批人',
      filterable: true,
      remote: true,
      clearable: true,
      remoteMethod: async (keyword) => {
        const res = await axios.get('/api/user/search', {
          params: { keyword }
        })
        const options = (res.data?.data || []).map(user => ({
          label: `${user.name} / ${user.deptName}`,
          value: user.id
        }))
        fApi.value.updateRule('assigneeId', { options })
      }
    },
    validate: [
      { required: true, message: '请选择审批人', trigger: 'change' }
    ]
  },
  {
    type: 'select',
    field: 'ccUserIds',
    title: '抄送人',
    value: [],
    options: [],
    props: {
      placeholder: '请输入姓名搜索抄送人',
      filterable: true,
      remote: true,
      multiple: true,
      collapseTags: true,
      clearable: true,
      remoteMethod: async (keyword) => {
        const res = await axios.get('/api/user/search', {
          params: { keyword }
        })
        const options = (res.data?.data || []).map(user => ({
          label: `${user.name} / ${user.deptName}`,
          value: user.id
        }))
        fApi.value.updateRule('ccUserIds', { options })
      }
    }
  },
  {
    type: 'datePicker',
    field: 'leaveRange',
    title: '请假时间',
    value: [],
    props: {
      type: 'datetimerange',
      startPlaceholder: '开始时间',
      endPlaceholder: '结束时间',
      valueFormat: 'YYYY-MM-DD HH:mm:ss'
    },
    validate: [
      { required: true, message: '请选择请假时间', trigger: 'change' }
    ]
  },
  {
    type: 'input',
    field: 'reason',
    title: '请假原因',
    value: '',
    props: {
      type: 'textarea',
      rows: 4,
      placeholder: '请输入请假原因'
    },
    validate: [
      { required: true, message: '请假原因必填', trigger: 'blur' }
    ]
  },
  {
    type: 'upload',
    field: 'attachments',
    title: '附件',
    value: [],
    props: {
      action: '/api/upload',
      multiple: true,
      limit: 5
    }
  },
  {
    type: 'group',
    field: 'details',
    title: '补充明细',
    value: [
      {
        date: '',
        hours: 8,
        remark: ''
      }
    ],
    props: {
      rule: [
        {
          type: 'datePicker',
          field: 'date',
          title: '日期',
          props: {
            type: 'date',
            valueFormat: 'YYYY-MM-DD'
          }
        },
        {
          type: 'inputNumber',
          field: 'hours',
          title: '请假小时数',
          value: 8,
          props: {
            min: 1,
            max: 24
          }
        },
        {
          type: 'input',
          field: 'remark',
          title: '备注',
          props: {
            placeholder: '请输入备注'
          }
        }
      ],
      addBtnText: '新增明细'
    }
  }
])

const option = ref({
  form: {
    labelWidth: '110px'
  },
  submitBtn: {
    text: '提交申请'
  },
  resetBtn: true,
  async onSubmit(formData) {
    await axios.post('/api/leave/apply', formData)
    console.log('提交成功')
  }
})

async function loadDeptOptions() {
  const res = await axios.get('/api/dept/list')
  const options = (res.data?.data || []).map(item => ({
    label: item.deptName,
    value: item.deptId
  }))
  fApi.value.updateRule('deptId', { options })
}

async function loadLeaveTypeOptions() {
  const res = await axios.get('/api/dict/options', {
    params: { dictType: 'leave_type' }
  })
  const options = (res.data?.data || []).map(item => ({
    label: item.label,
    value: item.value
  }))
  fApi.value.updateRule('leaveType', { options })
}

onMounted(async () => {
  await loadDeptOptions()
  await loadLeaveTypeOptions()
})
</script>
```

---

## 13. 编辑场景：详情回显

新增和编辑通常共用一个表单页面。编辑时要做两件事：

1. 加载详情数据
2. 处理依赖 options 的字段回显

```js
async function loadFormDetail(id) {
  const res = await axios.get(`/api/leave/detail/${id}`)
  const detail = res.data?.data

  if (!detail) return

  // 先补充审批人选项，避免只有 value 没有 label
  if (detail.assigneeId) {
    const userRes = await axios.get(`/api/user/detail/${detail.assigneeId}`)
    const user = userRes.data?.data
    fApi.value.updateRule('assigneeId', {
      options: [
        {
          label: `${user.name} / ${user.deptName}`,
          value: user.id
        }
      ]
    })
  }

  fApi.value.setValue(detail)
}
```

如果是多选抄送人，同理先查详情列表，再写入 options。

---

## 14. 提交前的数据转换

真实项目里，前端展示值和后端入参往往不完全一致，所以建议在提交时做一层转换。

例如：

```js
async function submitForm(formData) {
  const payload = {
    ...formData,
    startTime: formData.leaveRange?.[0] || '',
    endTime: formData.leaveRange?.[1] || ''
  }

  delete payload.leaveRange

  await axios.post('/api/leave/apply', payload)
}
```

再比如上传组件返回的是文件对象数组，但后端只要 URL 列表：

```js
payload.attachmentUrls = (formData.attachments || []).map(file => file.url)
```

---

## 15. 实战建议

### 15.1 规则和请求分开管理

建议把表单 `rule` 和接口请求拆开：

- `form-rules.js`：只放规则模板
- `api.js`：只放接口请求
- 页面里负责初始化和联动

这样更容易维护。

### 15.2 字典统一封装

凡是 select/radio/checkbox 的选项，都尽量走统一字典接口，不要每个页面各写一遍。

### 15.3 选人组件尽量支持远程搜索

人员数据通常很多，不适合一次性全部加载。建议：

- 默认加载最近联系人 / 常用联系人
- 输入关键字后远程搜索
- 编辑时根据 id 单独回显名称

### 15.4 复杂表格建议自定义组件

如果明细表涉及：

- 单元格校验
- 合计行
- 行内按钮
- 批量导入
- 行拖拽排序

这类需求建议直接封装可编辑表格组件，不要硬把所有逻辑塞进基础 rule。

---

## 16. 一份更推荐的目录结构

```bash
src/
├── api/
│   ├── user.js
│   ├── dept.js
│   └── dict.js
├── views/
│   └── leave/
│       ├── form.vue
│       └── form-rules.js
└── components/
    └── EditableTable.vue
```

例如用户接口：

```js
// api/user.js
import axios from 'axios'

export function searchUsers(keyword) {
  return axios.get('/api/user/search', {
    params: { keyword }
  })
}

export function getUserDetail(id) {
  return axios.get(`/api/user/detail/${id}`)
}
```

---

## 17. 总结

如果你要做一个 form-create 动态表单，推荐你按这个顺序实现：

1. 先定义基础 `rule`
2. 再补充表单校验
3. 把下拉、字典、选人数据改成接口加载
4. 用联动处理部门 -> 人员这类关系
5. 子表单/可编辑表格处理明细数据
6. 提交前统一做 payload 转换
7. 编辑页补齐 options 再做回显

一句话总结：

**form-create 适合“配置驱动表单”，而外部数据加载、选人搜索、表格明细，是业务项目里最核心的三块。**

---

## 18. 可直接复用的选人组件规则片段

最后给一个最常复制的版本：

```js
{
  type: 'select',
  field: 'userId',
  title: '选择人员',
  value: '',
  options: [],
  props: {
    placeholder: '请输入姓名/手机号搜索人员',
    filterable: true,
    remote: true,
    clearable: true,
    remoteMethod: async (keyword) => {
      const res = await axios.get('/api/user/search', {
        params: { keyword }
      })
      const options = (res.data?.data || []).map(user => ({
        label: `${user.name} / ${user.mobile} / ${user.deptName}`,
        value: user.id
      }))
      fApi.value.updateRule('userId', { options })
    }
  },
  validate: [
    { required: true, message: '请选择人员', trigger: 'change' }
  ]
}
```

如果你愿意，我下一步可以继续给你补两份内容：

1. **form-create + Element Plus 的完整页面源码版**
2. **带“弹窗选人树 + 表格回显”的企业级选人组件方案**
