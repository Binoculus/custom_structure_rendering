<script>
import Vue from 'vue'
import JsonTreeNode from './JsonTreeNode.vue'

function cloneJson(v) {
  return JSON.parse(JSON.stringify(v))
}

export default Vue.extend({
  name: 'JsonTreeEditor',
  model: {
    prop: 'modelValue',
    event: 'update:modelValue',
  },
  components: {
    JsonTreeNode,
  },
  props: {
    modelValue: {
      required: true,
    },
  },
  data() {
    return {
      localValue: cloneJson(this.modelValue),
    }
  },
  watch: {
    modelValue: {
      handler(v) {
        this.localValue = cloneJson(v)
      },
      deep: true,
    },
  },
  computed: {
    rootIsArray() {
      return Array.isArray(this.localValue)
    },
    rootIsObject() {
      return typeof this.localValue === 'object' && this.localValue !== null && !Array.isArray(this.localValue)
    },
    rootIsLeaf() {
      return !this.rootIsArray && !this.rootIsObject
    },
    rootEntries() {
      if (this.rootIsArray) {
        return this.localValue.map((v, idx) => ({ key: idx, value: v }))
      }
      if (this.rootIsObject) {
        return Object.entries(this.localValue).map(([k, v]) => ({ key: k, value: v }))
      }
      return []
    },
  },
  methods: {
    getValue() {
      return this.localValue
    },
    commitNext(next) {
      this.localValue = next
      this.$emit('update:modelValue', next)
      this.$emit('change', next)
    },
    setAtPath(root, path, value) {
      if (path.length === 0) return

      let cursor = root
      for (let i = 0; i < path.length - 1; i++) {
        const key = path[i]
        if (key === undefined) return
        cursor = cursor[key]
      }

      const lastKey = path[path.length - 1]
      if (lastKey === undefined) return
      cursor[lastKey] = value
    },
    onUpdateLeaf(payload) {
      if (payload.path.length === 0) {
        const next = payload.value
        this.commitNext(next)
        return
      }

      const next = cloneJson(this.localValue)
      this.setAtPath(next, payload.path, payload.value)
      this.commitNext(next)
    },
    isPlainObject(v) {
      return typeof v === 'object' && v !== null && !Array.isArray(v)
    },
    getAtPath(root, path) {
      let cursor = root
      for (const key of path) {
        cursor = cursor?.[key]
      }
      return cursor
    },
    getParentAtPath(root, path) {
      if (path.length === 0) return { parent: undefined, key: undefined }
      let parent = root
      for (let i = 0; i < path.length - 1; i++) {
        parent = parent?.[path[i]]
      }
      return { parent, key: path[path.length - 1] }
    },
    uniqueKey(obj, prefix = 'newKey') {
      let i = 1
      while (Object.prototype.hasOwnProperty.call(obj, `${prefix}${i}`)) i++
      return `${prefix}${i}`
    },
    valueFromKind(kind) {
      if (kind === 'array') return []
      if (kind === 'object') return {}
      return null
    },
    onAddChild(payload) {
      const next = cloneJson(this.localValue)
      const target = payload.path.length === 0 ? next : this.getAtPath(next, payload.path)

      if (Array.isArray(target)) {
        target.push(this.valueFromKind(payload.kind))
        this.commitNext(next)
        return
      }

      if (this.isPlainObject(target)) {
        const key = this.uniqueKey(target)
        target[key] = this.valueFromKind(payload.kind)
        this.commitNext(next)
        return
      }
    },
    onAddSibling(payload) {
      if (payload.path.length === 0) return

      const next = cloneJson(this.localValue)
      const { parent, key } = this.getParentAtPath(next, payload.path)
      if (parent === undefined || key === undefined) return

      if (Array.isArray(parent)) {
        const idx = typeof key === 'number' ? key : Number(key)
        if (!Number.isInteger(idx)) return
        parent.splice(idx + 1, 0, this.valueFromKind(payload.kind))
        this.commitNext(next)
        return
      }

      if (this.isPlainObject(parent)) {
        const newK = this.uniqueKey(parent)
        parent[newK] = this.valueFromKind(payload.kind)
        this.commitNext(next)
        return
      }
    },
    onRenameKey(payload) {
      if (payload.path.length === 0) return

      const next = cloneJson(this.localValue)
      const { parent, key } = this.getParentAtPath(next, payload.path)
      if (!this.isPlainObject(parent)) return
      if (typeof key !== 'string') return

      const oldKey = key
      const newKey = payload.newKey.trim()
      if (!newKey || newKey === oldKey) return
      if (Object.prototype.hasOwnProperty.call(parent, newKey)) {
        return
      }

      parent[newKey] = parent[oldKey]
      delete parent[oldKey]
      this.commitNext(next)
    },
  },
})
</script>

<template>
  <div>
    <template v-if="rootIsLeaf">
      <JsonTreeNode
        label=""
        :value="localValue"
        :level="0"
        :path="[]"
        parent-type="root"
        @update-leaf="onUpdateLeaf"
        @add-child="onAddChild"
        @add-sibling="onAddSibling"
        @rename-key="onRenameKey"
      />
    </template>
    <template v-else>
      <JsonTreeNode
        v-for="child in rootEntries"
        :key="String(child.key)"
        :label="rootIsArray ? '' : String(child.key)"
        :value="child.value"
        :level="0"
        :path="[child.key]"
        :parent-type="rootIsArray ? 'array' : 'object'"
        @update-leaf="onUpdateLeaf"
        @add-child="onAddChild"
        @add-sibling="onAddSibling"
        @rename-key="onRenameKey"
      />
    </template>
  </div>
</template>
