<script setup lang="ts">
import { ref } from 'vue'
import JsonTreeEditor from './components/JsonTreeEditor.vue'

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

// 情况 1：根节点为 object（多层混合：object/array/primitive）
const objectRoot = ref<any>({
  user: {
    id: 1001,
    name: 'Alice',
    active: true,
    meta: {
      tags: ['a', 'b', 'c'],
      score: 98.6,
    },
  },
  orders: [
    { id: 'o-1', amount: 12.34, items: [{ sku: 'p1', qty: 2 }] },
    { id: 'o-2', amount: 99, items: [] },
  ],
  note: null,
})

// 情况 2：根节点为 array（多层混合：array/object/primitive）
const arrayRoot = ref<any>([
  {
    name: 'root-0',
    children: [
      { name: 'child-0', value: 1 },
      { name: 'child-1', value: 'x', extra: { ok: false } },
    ],
  },
  ['nested-arr', 2, true, null, { deep: [{ k: 'v' }] }],
])

function logCurrent(label: string, data: unknown) {
  // 这里拿到的是“结构不变”的最新数据（object/array 形态保持一致）
  console.log(label, data)
}
</script>

<template>
  <div class="page">
    <section class="section">
      <div class="header">
        <h2>根节点是 Object</h2>
        <div class="actions">
          <button type="button" @click="logCurrent('objectRoot', objectRoot)">获取当前数据</button>
          <button type="button" @click="downloadJson('object-root.json', objectRoot)">导出 JSON</button>
        </div>
      </div>
      <JsonTreeEditor v-model="objectRoot" />
    </section>

    <section class="section">
      <div class="header">
        <h2>根节点是 Array</h2>
        <div class="actions">
          <button type="button" @click="logCurrent('arrayRoot', arrayRoot)">获取当前数据</button>
          <button type="button" @click="downloadJson('array-root.json', arrayRoot)">导出 JSON</button>
        </div>
      </div>
      <JsonTreeEditor v-model="arrayRoot" />
    </section>
  </div>
</template>

<style scoped>
.page {
  padding: 16px;
  display: grid;
  gap: 24px;
}
.section {
  border: 1px solid rgba(0, 0, 0, 0.15);
  border-radius: 8px;
  padding: 12px;
}
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 12px;
}
.actions {
  display: flex;
  gap: 8px;
}
button {
  padding: 6px 10px;
}
</style>
