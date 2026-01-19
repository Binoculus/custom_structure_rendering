# custom_structure_rendering（JsonTreeEditor）

一个用于**递归渲染 + 可编辑**“任意深度 / 不确定结构”的 JSON 树组件（JSON Tree Viewer / JSON Tree Editor）。

适用场景：后端返回结构不固定的数据、动态表单/动态配置、任意嵌套 object/array 的可视化与在线编辑。

## 关键词（便于检索）

**中文**：JSON 树、树形渲染、递归渲染、递归组件、任意结构渲染、不确定数据结构、后端返回数据、动态 schema、树形编辑器、JSON 编辑器、JSON 查看器、叶子节点编辑、深层嵌套对象、深层嵌套数组、路径更新、结构保持、展开折叠、同级新增、子级新增、重命名 key、导出 JSON。

**English**: JSON tree editor, JSON tree viewer, recursive rendering, recursive component, dynamic schema, unknown JSON structure, nested object, nested array, editable leaf nodes, tree view, expand/collapse, add sibling, add child, rename key, immutable-ish update, path-based update, Vue 2 JSON editor, Vite + Vue2.

## 项目目标

- 处理后端返回的**不确定数据结构**（object/array/primitive 混合、任意深度）
- 将数据结构渲染为**树形（tree view）**
- 允许编辑叶子值，同时**不破坏原有结构形态**（array 仍为 array、object 仍为 object）
- 让每层节点都可控制（展开/折叠、同级/子级新增、重命名 key 等）

## 功能特性

- **根节点可为 object 或 array**
- **节点类型自动识别**：object / array / primitive（null/string/number/boolean）
- **递归渲染**：每一层缩进 20px
- **叶子节点可编辑**：leaf 渲染为输入框，blur/Enter 提交
- **类型尽量保持稳定**：number/boolean 会尝试按原类型解析，不合法回退原值
- **结构保持更新**：只替换目标叶子值，不会把 array 变 object（或反之）
- **path 定位更新**：用 `path: (string | number)[]` 精确定位深层字段
- **展开/折叠**：支持折叠 object/array 子树
- **节点操作**：同级新增、子级新增（array/object/field），对象 key 重命名
- **导出 JSON**：Demo 中提供下载 JSON 文件示例（Blob + a.download）

## 技术栈

- Vue 2.7.x（Options API）
- Vite（@vitejs/plugin-vue2）
- JavaScript（ESM）

## 快速开始

推荐使用 pnpm（仓库包含 pnpm-lock.yaml）：

```bash
pnpm i
pnpm dev
```

或使用 npm：

```bash
npm i
npm run dev
```

构建与预览：

```bash
pnpm build
pnpm preview
```

## 使用方式（组件接入）

组件文件：

- `src/components/JsonTreeEditor.vue`
- `src/components/JsonTreeNode.vue`

父组件中通过 `v-model` 绑定整棵 JSON 树：

```vue
<template>
	<JsonTreeEditor v-model="data" @change="onChange" />
</template>

<script>
import JsonTreeEditor from './components/JsonTreeEditor.vue'

export default {
	components: { JsonTreeEditor },
	data() {
		return {
			data: {
				user: { name: 'Alice', ok: true, score: 12.5 },
				list: [1, 2, { deep: null }],
			},
		}
	},
	methods: {
		onChange(next) {
			// next 就是“保持结构”的最新数据
			console.log('changed:', next)
		},
	},
}
</script>
```

## 组件 API（对外）

### Props

- `modelValue: any`：整棵树数据（建议为 JSON 可序列化结构）

### Events

- `update:modelValue(next)`：v-model 回传
- `change(next)`：每次提交变更时触发

## 数据约束与注意事项

- 当前实现以“JSON 兼容数据”为目标（可被 `JSON.stringify()` 正常序列化）。
- 如果数据包含 `Date` / `Map` / `undefined` / 函数等非 JSON 值，编辑与 clone 行为不保证正确。
- 组件内部为避免直接 mutate 外部引用，会做一次 `JSON.parse(JSON.stringify(...))` 的深拷贝。

## 文档

- 设计与关键逻辑说明：`docs/json-tree-editor.md`

## 时间节点 / Roadmap

- 2026.01.16：完成基本逻辑（递归渲染、叶子编辑、path 更新、结构保持）
- 2026.x.x：封装成可复用工具包（组件化发布、样式/主题、类型与更多编辑策略）