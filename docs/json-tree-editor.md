# 递归渲染/编辑任意 JSON 树组件（Vue 3 + TS）

本项目里实现了一个“树形递归渲染器 + 叶子节点可编辑”的组件，用来展示/编辑**任意多层**的 JSON-like 数据结构：

- 根节点可以是 **object** 或 **array**
- 任意节点都可能是 **object / array / primitive(null/string/number/boolean)**
- **递归渲染**：每一层缩进 20px
- **叶子节点**（不是 object/array）渲染为输入框，可编辑
- **数据结构不改变**：只替换叶子值，不会把 array 变 object（或反之）
- 编辑完成后，可直接拿到“原样结构”的最新数据（通过 `v-model`）

对应实现文件：

- src/components/JsonTreeEditor.vue
- src/components/JsonTreeNode.vue

---

## 1. 组件职责

### JsonTreeNode（递归节点组件）

- 判断当前节点类型：object / array / leaf
- 对 object/array 展开子节点并继续递归渲染
- 对 leaf 节点渲染 `<input>` 并在提交时向上抛出更新事件（携带 path）

### JsonTreeEditor（编辑器封装组件）

- 接收 `modelValue` 作为整棵树数据（根节点可 object 或 array）
- 监听叶子更新事件（path + 新值）
- 基于 path 对整棵树做一次“**只修改目标叶子的结构保持更新**”
- 通过 `emit('update:modelValue', next)` 把最新数据回传给父组件

---

## 2. 数据类型约束（JsonValue）

组件只保证处理 JSON 兼容的数据（也就是可被 `JSON.stringify()` 正常序列化的结构）：

```ts
export type JsonPrimitive = string | number | boolean | null
export type JsonValue = JsonPrimitive | JsonObject | JsonArray
export type JsonObject = { [key: string]: JsonValue }
export type JsonArray = JsonValue[]
```

说明：

- 如果你的数据里有 `Date`、`Map`、`undefined`、函数等非 JSON 值，当前实现不保证序列化/编辑行为正确。

---

## 3. 关键逻辑：递归渲染（每层缩进 20px）

在 src/components/JsonTreeNode.vue 中：

1) **类型判断**

```ts
const isArray = computed(() => Array.isArray(props.value))
const isObject = computed(
  () => typeof props.value === 'object' && props.value !== null && !Array.isArray(props.value)
)
const isLeaf = computed(() => !isArray.value && !isObject.value)
```

2) **把 object/array 转成统一的 children 列表**

```ts
const childEntries = computed(() => {
  if (isArray.value) {
    return (props.value as JsonArray).map((v, idx) => ({ key: idx, value: v }))
  }
  if (isObject.value) {
    return Object.entries(props.value as JsonObject).map(([k, v]) => ({ key: k, value: v }))
  }
  return []
})
```

3) **缩进 20px**

```ts
const indentStyle = computed(() => ({ marginLeft: `${props.level * 20}px` }))
```

4) **组件自递归**

在 `<script setup>` 下通过 `defineOptions({ name: 'JsonTreeNode' })` 明确组件名，保证模板里 `<JsonTreeNode ... />` 能递归引用自身：

```ts
defineOptions({
  name: 'JsonTreeNode',
})
```

---

## 4. 关键逻辑：叶子节点输入框与提交

### 4.1 叶子节点渲染为输入框

在 template 中：

```vue
<template v-if="isLeaf">
  <input
    class="input"
    :value="leafInput"
    @input="leafInput = ($event.target as HTMLInputElement).value"
    @blur="commitLeaf"
  />
</template>
```

约定：

- 只有当节点是 leaf（不是 object/array）才显示输入框
- 当前实现用 `blur` 作为“编辑完成”时机（你也可以改成 `@change` 或每次 `@input` 即提交）

### 4.2 基于“原值类型”做轻量转换（保持类型尽量稳定）

当前实现的策略是：

- 原值是 number：输入转 `Number(raw)`，不合法则回退为原值
- 原值是 boolean：仅接受 `true/false`（忽略大小写），不合法则回退
- 原值是 string：保持 string
- 原值是 null：输入为 `null`（字符串）时仍视为 null，否则当作 string

核心代码：

```ts
function parsePrimitiveFromString(raw: string, original: JsonPrimitive): JsonPrimitive {
  if (original === null) {
    return raw
  }
  switch (typeof original) {
    case 'number': {
      const n = Number(raw)
      return Number.isFinite(n) ? n : original
    }
    case 'boolean': {
      const lower = raw.trim().toLowerCase()
      if (lower === 'true') return true
      if (lower === 'false') return false
      return original
    }
    case 'string':
    default:
      return raw
  }
}
```

提交时向上抛出事件：

```ts
emit('update-leaf', { path: props.path, value: nextValue })
```

---

## 5. 关键逻辑：用 path 定位并更新叶子（结构不变）

### 5.1 为什么用 path

每个节点都有一个 path（例如 `['user','meta','score']` 或 `[1,'deep',0,'k']`），这样子组件不需要知道“整棵树”在哪里，只要抛出：

- path：目标叶子位置
- value：新值

父层统一接管“如何更新树”。

### 5.2 setAtPath：定位到最后一级 key 并赋值

在 src/components/JsonTreeEditor.vue：

```ts
function setAtPath(root: JsonValue, path: Array<string | number>, value: JsonPrimitive) {
  if (path.length === 0) return

  let cursor: any = root
  for (let i = 0; i < path.length - 1; i++) {
    const key = path[i]
    if (key === undefined) return
    cursor = cursor[key]
  }

  const lastKey = path[path.length - 1]
  if (lastKey === undefined) return
  cursor[lastKey] = value
}
```

这个函数的关键点：

- 只沿着现有结构向下走，不会创建新节点
- 最终只替换叶子值：**不会改动 object/array 的形态**

---

## 6. 关键逻辑：如何保证编辑后能拿到“原样结构”的数据

### 6.1 v-model 回传整棵树

叶子变更后，会构造 `next` 并触发：

```ts
emit('update:modelValue', next)
emit('change', next)
```

父组件（例如 App）可以直接拿 `v-model` 对应的变量，它就是最新数据：

```vue
<JsonTreeEditor v-model="objectRoot" />
```

### 6.2 关于“不能改变原始数据结构”的说明

- 本实现保证“结构形态不变”：原来是 array 的地方仍是 array，原来是 object 的地方仍是 object
- 叶子节点的值会被替换（这就是编辑的目的）

### 6.3 为什么内部用了 clone（以及 TS 类型说明）

为了避免直接 mutate 父组件传入的对象引用（以及更容易触发视图更新），编辑器内部会对数据做一次 `JSON.stringify/parse` 的 clone。

另外，`JsonValue` 是递归类型，在某些 TypeScript 配置下把它作为 `ref<JsonValue>` 的泛型会触发“Type instantiation is excessively deep”。
因此本实现把内部的 `ref` 存储类型收敛为 `any`，但对外输入/输出仍遵循 `JsonValue` 约束。

---

## 7. Demo：导出为 JSON 文件

在 App 里用 `Blob + a.download` 导出：

```ts
function downloadJson(filename: string, data: unknown) {
  const json = JSON.stringify(data, null, 2)
  const blob = new Blob([json], { type: 'application/json;charset=utf-8' })
  const url = URL.createObjectURL(blob)

  const a = document.createElement('a')
  a.href = url
  a.download = filename
  document.body.appendChild(a)
  a.click()
  a.remove()
  URL.revokeObjectURL(url)
}
```

---

## 8. 可选增强（如果你需要我继续改）

- 提交时机：改成实时提交（每次 input 都更新）/ 或增加“确认”按钮
- 更完整的类型转换策略：比如允许 number 输入空字符串变成 null，或支持显式输入 `"null"/"true"/"false"` 强制转换
- 展开/折叠节点（目前实现是全展开）
