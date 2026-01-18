<script setup lang="ts">
import { computed, ref, watch } from 'vue'

defineOptions({
  name: 'JsonTreeNode',
})

export type JsonPrimitive = string | number | boolean | null
export type JsonValue = JsonPrimitive | JsonObject | JsonArray
export type JsonObject = { [key: string]: JsonValue }
export type JsonArray = JsonValue[]

const props = defineProps<{
  label: string
  value: JsonValue
  level: number
  path: Array<string | number>
}>()

const emit = defineEmits<{
  (e: 'update-leaf', payload: { path: Array<string | number>; value: JsonPrimitive }): void
}>()

const isArray = computed(() => Array.isArray(props.value))
const isObject = computed(() => typeof props.value === 'object' && props.value !== null && !Array.isArray(props.value))
const isLeaf = computed(() => !isArray.value && !isObject.value)

const childEntries = computed(() => {
  if (isArray.value) {
    return (props.value as JsonArray).map((v, idx) => ({ key: idx, value: v }))
  }
  if (isObject.value) {
    return Object.entries(props.value as JsonObject).map(([k, v]) => ({ key: k, value: v }))
  }
  return [] as Array<{ key: string | number; value: JsonValue }>
})

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

const leafInput = ref('')

watch(
  () => props.value,
  (v) => {
    if (isLeaf.value) {
      leafInput.value = v === null ? 'null' : String(v)
    } else {
      leafInput.value = ''
    }
  },
  { immediate: true }
)

function commitLeaf() {
  if (!isLeaf.value) return
  const original = props.value as JsonPrimitive

  if (original === null) {
    if (leafInput.value.trim().toLowerCase() === 'null') {
      emit('update-leaf', { path: props.path, value: null })
      return
    }
    emit('update-leaf', { path: props.path, value: leafInput.value })
    return
  }

  emit('update-leaf', {
    path: props.path,
    value: parsePrimitiveFromString(leafInput.value, original),
  })
}

const indentStyle = computed(() => ({ marginLeft: `${props.level * 20}px` }))

const typeHint = computed(() => {
  if (isArray.value) return 'array'
  if (isObject.value) return 'object'
  if (props.value === null) return 'null'
  return typeof props.value
})
</script>

<template>
  <div :style="indentStyle" class="node">
    <div class="row">
      <span class="label">{{ label }}</span>
      <span class="type">({{ typeHint }})</span>

      <template v-if="isLeaf">
        <input
          class="input"
          :value="leafInput"
          @input="leafInput = ($event.target as HTMLInputElement).value"
          @blur="commitLeaf"
        />
      </template>
    </div>

    <template v-if="!isLeaf">
      <JsonTreeNode
        v-for="child in childEntries"
        :key="String(child.key)"
        :label="String(child.key)"
        :value="child.value"
        :level="level + 1"
        :path="[...path, child.key]"
        @update-leaf="emit('update-leaf', $event)"
      />
    </template>
  </div>
</template>

<style scoped>
.node {
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji",
    "Segoe UI Emoji";
}
.row {
  display: flex;
  align-items: center;
  gap: 8px;
  min-height: 28px;
}
.label {
  min-width: 90px;
  font-weight: 600;
}
.type {
  opacity: 0.7;
  font-size: 12px;
}
.input {
  flex: 1;
  max-width: 520px;
  padding: 4px 8px;
}
</style>
