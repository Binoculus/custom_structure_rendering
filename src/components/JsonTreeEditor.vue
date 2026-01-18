<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import JsonTreeNode, { type JsonValue, type JsonPrimitive } from './JsonTreeNode.vue'

const props = defineProps<{
  modelValue: JsonValue
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', value: JsonValue): void
  (e: 'change', value: JsonValue): void
}>()

function cloneJson(v: unknown): unknown {
  return JSON.parse(JSON.stringify(v))
}

// NOTE: JsonValue 是递归类型，直接用于 ref<T> 在某些 TS 配置下会触发“推导过深”。
// 这里用 any 作为内部存储类型；对外仍保持 JsonValue 结构（v-model 传入/传出）。
const localValue = ref<any>(cloneJson(props.modelValue))

watch(
  () => props.modelValue,
  (v) => {
    localValue.value = cloneJson(v)
  },
  { deep: true }
)

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

function onUpdateLeaf(payload: { path: Array<string | number>; value: JsonPrimitive }) {
  const next = cloneJson(localValue.value) as JsonValue
  setAtPath(next, payload.path, payload.value)
  localValue.value = next
  emit('update:modelValue', next)
  emit('change', next)
}

const rootLabel = computed(() => (Array.isArray(localValue.value) ? 'root[]' : 'root{}'))

defineExpose({
  getValue: () => localValue.value as JsonValue,
})
</script>

<template>
  <div>
    <JsonTreeNode :label="rootLabel" :value="localValue" :level="0" :path="[]" @update-leaf="onUpdateLeaf" />
  </div>
</template>
