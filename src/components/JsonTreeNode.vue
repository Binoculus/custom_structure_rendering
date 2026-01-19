<script>
import Vue from 'vue'

export default Vue.extend({
  name: 'JsonTreeNode',
  props: {
    label: { type: String, default: '' },
    value: { required: true },
    level: { type: Number, required: true },
    path: { type: Array, required: true },
    parentType: { type: String, default: 'root' },
  },
  data() {
    return {
      leafInput: '',
      keyInput: this.label,
      collapsed: false,
    }
  },
  computed: {
    isArray() {
      return Array.isArray(this.value)
    },
    isObject() {
      return typeof this.value === 'object' && this.value !== null && !Array.isArray(this.value)
    },
    isLeaf() {
      return !this.isArray && !this.isObject
    },
    canAddSibling() {
      return this.path.length > 0
    },
    canAddChild() {
      return !this.isLeaf
    },
    canEditKey() {
      return this.parentType === 'object' && this.isLeaf && String(this.label).trim().length > 0
    },
    childEntries() {
      if (this.isArray) {
        return this.value.map((v, idx) => ({ key: idx, value: v }))
      }
      if (this.isObject) {
        return Object.entries(this.value).map(([k, v]) => ({ key: k, value: v }))
      }
      return []
    },
    canToggleCollapse() {
      return !this.isLeaf
    },
    collapsedSummary() {
      if (this.isArray) return `[${this.childEntries.length}]`
      if (this.isObject) return `{${this.childEntries.length}}`
      return ''
    },
    indentStyle() {
      return { marginLeft: `${this.level * 20}px` }
    },
  },
  watch: {
    label: {
      handler(v) {
        this.keyInput = v
      },
      immediate: true,
    },
    value: {
      handler(v) {
        if (this.isLeaf) {
          this.leafInput = v === null ? 'null' : String(v)
        } else {
          this.leafInput = ''
        }
      },
      immediate: true,
    },
  },
  methods: {
    toggleCollapse() {
      if (!this.canToggleCollapse) return
      this.collapsed = !this.collapsed
    },
    parsePrimitiveFromString(raw, original) {
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
    },
    commitLeaf() {
      if (!this.isLeaf) return
      const original = this.value

      if (original === null) {
        if (String(this.leafInput).trim().toLowerCase() === 'null') {
          this.$emit('update-leaf', { path: this.path, value: null })
          return
        }
        this.$emit('update-leaf', { path: this.path, value: this.leafInput })
        return
      }

      this.$emit('update-leaf', {
        path: this.path,
        value: this.parsePrimitiveFromString(String(this.leafInput), original),
      })
    },
    commitKey() {
      if (!this.canEditKey) return
      const newKey = String(this.keyInput).trim()
      if (!newKey) {
        this.keyInput = this.label
        return
      }
      this.$emit('rename-key', { path: this.path, newKey })
    },
    onAddSibling(kind) {
      if (!this.canAddSibling) return
      this.$emit('add-sibling', { path: this.path, kind })
    },
    onAddChild(kind) {
      if (!this.canAddChild) return
      this.$emit('add-child', { path: this.path, kind })
    },
  },
})
</script>

<template>
  <div :style="indentStyle" class="node-h">
    <div class="row-q">
      <button
        v-if="canToggleCollapse"
        type="button"
        class="twisty"
        :title="collapsed ? '展开' : '折叠'"
        @click="toggleCollapse"
      >
        <span class="twistyIcon" :class="{ collapsed }">▾</span>
      </button>
      <span v-else class="twistyPlaceholder" />

      <template v-if="label">
        <input
          v-if="canEditKey"
          class="labelInput"
          v-model="keyInput"
          @blur="commitKey"
          @keydown.enter.prevent="commitKey"
        />
        <span v-else class="label-k">{{ label }}</span>
      </template>

      <span v-if="collapsed && canToggleCollapse" class="collapsedHint">{{ collapsedSummary }}</span>

      <template v-if="isLeaf">
        <input
          class="input"
          v-model="leafInput"
          @blur="commitLeaf"
        />
      </template>

      <div class="actions">
        <div class="group">
          <span class="groupLabel">同级</span>
          <button type="button" class="btn" :disabled="!canAddSibling" title="同级增加 array" @click="onAddSibling('array')">
            +A
          </button>
          <button type="button" class="btn" :disabled="!canAddSibling" title="同级增加 json(object)" @click="onAddSibling('object')">
            +J
          </button>
          <button type="button" class="btn" :disabled="!canAddSibling" title="同级增加字段/值" @click="onAddSibling('field')">
            +F
          </button>
        </div>

        <div class="group">
          <span class="groupLabel">子级</span>
          <button type="button" class="btn" :disabled="!canAddChild" title="子级增加 array" @click="onAddChild('array')">
            +A
          </button>
          <button type="button" class="btn" :disabled="!canAddChild" title="子级增加 json(object)" @click="onAddChild('object')">
            +J
          </button>
          <button
            type="button"
            class="btn"
            :disabled="!canAddChild"
            :title="canAddChild ? '子级增加字段/值' : '叶子节点不可新增子级'"
            @click="onAddChild('field')"
          >
            +F
          </button>
        </div>
      </div>
    </div>

    <template v-if="!isLeaf && !collapsed">
      <JsonTreeNode
        v-for="child in childEntries"
        :key="String(child.key)"
        :label="isArray ? '' : String(child.key)"
        :value="child.value"
        :level="level + 1"
        :path="[...path, child.key]"
        :parent-type="isArray ? 'array' : 'object'"
        @update-leaf="$emit('update-leaf', $event)"
        @add-sibling="$emit('add-sibling', $event)"
        @add-child="$emit('add-child', $event)"
        @rename-key="$emit('rename-key', $event)"
      />
    </template>
  </div>
</template>

<style scoped>
.node-h {
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji",
    "Segoe UI Emoji";
  color: #111827;
}
.row-q {
  display: flex;
  align-items: center;
  gap: 10px;
  min-height: 30px;
  padding: 4px 8px;
  border-radius: 8px;
  transition: background-color 120ms ease;
}
.row-q:hover {
  background: rgba(17, 24, 39, 0.04);
}

.twisty {
  width: 22px;
  height: 22px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 1px solid rgba(17, 24, 39, 0.12);
  border-radius: 6px;
  background: #fff;
  cursor: pointer;
  padding: 0;
}

.twisty:hover {
  background: rgba(17, 24, 39, 0.04);
}

.twistyIcon {
  display: inline-block;
  line-height: 1;
  transform: rotate(0deg);
  transition: transform 120ms ease;
  font-size: 14px;
  color: rgba(17, 24, 39, 0.8);
}

.twistyIcon.collapsed {
  transform: rotate(-90deg);
}

.twistyPlaceholder {
  width: 22px;
  height: 22px;
  display: inline-block;
}

.collapsedHint {
  font-size: 12px;
  color: rgba(17, 24, 39, 0.7);
  background: rgba(17, 24, 39, 0.04);
  border: 1px solid rgba(17, 24, 39, 0.08);
  border-radius: 999px;
  padding: 2px 8px;
}
.label-k {
  flex: 0 0 160px;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-weight: 600;
}

.labelInput {
  flex: 0 0 160px;
  padding: 6px 10px;
  border-radius: 8px;
  border: 1px solid rgba(17, 24, 39, 0.18);
  background: #fff;
  outline: none;
  font-weight: 600;
}

.labelInput:focus {
  border-color: rgba(59, 130, 246, 0.7);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.18);
}
.input {
  flex: 1;
  max-width: 560px;
  padding: 6px 10px;
  border-radius: 8px;
  border: 1px solid rgba(17, 24, 39, 0.18);
  background: #fff;
  outline: none;
}
.input:focus {
  border-color: rgba(59, 130, 246, 0.7);
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.18);
}

.actions {
  margin-left: auto;
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}

.group {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.groupLabel {
  font-size: 12px;
  color: rgba(17, 24, 39, 0.7);
}

.btn {
  padding: 3px 7px;
  border-radius: 8px;
  border: 1px solid rgba(17, 24, 39, 0.18);
  background: #fff;
  color: #111827;
  cursor: pointer;
}

.btn:hover {
  background: rgba(17, 24, 39, 0.04);
}

.btn:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}
</style>
