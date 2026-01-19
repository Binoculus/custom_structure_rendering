<script>
import Vue from 'vue'
import JsonTreeEditor from './components/JsonTreeEditor.vue'

function randomInt(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min
}

function pickOne(list) {
  return list[randomInt(0, list.length - 1)]
}

function makeKey(depth, index) {
  const prefixes = ['字段', '指标', '维度', '属性', '分类', '项目', '条目', '说明']
  const suffixes = ['名称', '数值', '占比', '单位', '年份', '明细', '合计', '备注']
  const prefix = pickOne(prefixes)
  const suffix = pickOne(suffixes)
  const no = String(index).padStart(3, '0')
  return `${prefix}${depth}-${no}-${suffix}`
}

function randomPrimitive() {
  const type = pickOne(['string', 'number', 'boolean', 'null'])
  if (type === 'null') return null
  if (type === 'boolean') return Math.random() < 0.5
  if (type === 'number') {
    const isFloat = Math.random() < 0.35
    const base = randomInt(0, 1000000)
    return isFloat ? Number((base * Math.random()).toFixed(3)) : base
  }
  const samples = [
    '元',
    '万元',
    '2024年',
    '2023年',
    '张三',
    '李四',
    '王五',
    '正常',
    '异常',
    '—',
  ]
  if (Math.random() < 0.6) return pickOne(samples)
  const letters = 'abcdefghijklmnopqrstuvwxyz'
  const len = randomInt(3, 10)
  let s = ''
  for (let i = 0; i < len; i++) s += letters[randomInt(0, letters.length - 1)]
  return s
}

function randomNode(currentDepth, maxDepth) {
  if (currentDepth >= maxDepth) return randomPrimitive()

  const roll = Math.random()
  // 深层更偏向 primitive，避免结构爆炸
  const depthBias = (currentDepth - 1) / (maxDepth - 1)
  const primitiveThreshold = 0.15 + depthBias * 0.35
  if (roll < primitiveThreshold) return randomPrimitive()

  if (roll < primitiveThreshold + 0.4) {
    // object
    const obj = {}
    const count = randomInt(1, currentDepth <= 2 ? 6 : 4)
    for (let i = 1; i <= count; i++) {
      obj[makeKey(currentDepth + 1, i)] = randomNode(currentDepth + 1, maxDepth)
    }
    return obj
  }

  // array
  const arrLen = randomInt(1, currentDepth <= 2 ? 6 : 4)
  const arr = []
  for (let i = 0; i < arrLen; i++) {
    arr.push(randomNode(currentDepth + 1, maxDepth))
  }
  return arr
}

function generateRandomRoot({ topLevelCount = 50, maxDepth = 5 } = {}) {
  // 约定：根节点为第 1 级（object），叶子不超过 maxDepth
  const root = {}
  for (let i = 1; i <= topLevelCount; i++) {
    const key = makeKey(1, i)
    root[key] = randomNode(1, maxDepth)
  }
  return root
}

export default Vue.extend({
  name: 'App',
  components: {
    JsonTreeEditor,
  },
  data() {
    return {
      // 随机模拟：最外层 50 条、最大深度 5 级、结构随机混合
      objectRoot: {},

      // 情况 2：根节点为 array（多层混合：array/object/primitive）
      // arrayRoot: [
      //   {
      //     name: 'root-0',
      //     children: [
      //       { name: 'child-0', value: 1 },
      //       { name: 'child-1', value: 'x', extra: { ok: false } },
      //     ],
      //   },
      //   ['nested-arr', 2, true, null, { deep: [{ k: 'v' }] }],
      // ],
    }
  },
  created() {
    this.regenerate()
  },
  methods: {
    regenerate() {
      this.objectRoot = generateRandomRoot({ topLevelCount: 5, maxDepth: 3 })
    },
    downloadJson(filename, data) {
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
    },
    logCurrent(label, data) {
      console.log(label, data)
    },
  },
})
</script>

<template>
  <div class="page">
    <section class="section">
      <div class="header">
        <h2>随机模拟数据（50 条 / 最大 5 级）</h2>
        <div class="actions">
          <button type="button" @click="regenerate">重新生成</button>
          <button type="button" @click="logCurrent('objectRoot', objectRoot)">获取当前数据</button>
          <button type="button" @click="downloadJson('object-root.json', objectRoot)">导出 JSON</button>
        </div>
      </div>
      <JsonTreeEditor v-model="objectRoot" />

    </section>
    <pre style="text-align: left;">{{ JSON.stringify(objectRoot, null, 2) }}</pre>
  </div>
</template>

<style scoped>
.page {
  padding: 16px;
  display: flex;
}

.section {
  border: 1px solid rgba(0, 0, 0, 0.15);
  border-radius: 8px;
  padding: 12px;
  display: flex;
  flex-direction: column;
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
