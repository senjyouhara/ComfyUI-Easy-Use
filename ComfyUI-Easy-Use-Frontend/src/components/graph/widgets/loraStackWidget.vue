<template>
  <div class="easyuse-lora-stack-widget" @mousedown.stop @wheel.stop>
    <div class="lsw-header">
      <label class="lsw-global-toggle">
        <input type="checkbox" :checked="globalEnabled" @change="updateGlobalEnabled($event.target.checked)" />
        <span>{{ $t('Enabled') }}</span>
      </label>
    </div>

    <div class="lsw-list">
      <div v-for="(row, index) in rows" :key="row.key" class="lsw-row">
        <button
          class="lsw-toggle-btn"
          :class="{ active: row.enabled }"
          type="button"
          :title="row.enabled ? $t('Enabled') : $t('Disabled')"
          @click="toggleRow(index)"
        >
          <i :class="row.enabled ? 'pi pi-check-circle' : 'pi pi-circle'" />
        </button>

        <select class="lsw-select" :value="row.name" @change="updateRowName(index, $event.target.value)">
          <option v-for="option in loraOptions" :key="option" :value="option">{{ option }}</option>
        </select>

        <input
          class="lsw-weight"
          type="number"
          :value="row.strength"
          step="0.01"
          min="-10"
          max="10"
          @change="updateRowStrength(index, $event.target.value)"
        />

        <button class="lsw-delete-btn" type="button" :title="$t('Delete')" @click="removeRow(index)">
          <i class="pi pi-trash" />
        </button>
      </div>

      <div v-if="rows.length === 0" class="lsw-empty">
        {{ $t('Select to add LoRA') }}
      </div>
    </div>

    <button class="lsw-add-btn" type="button" :disabled="rows.length >= maxRows" @click="addRow">
      <i class="pi pi-plus" />
      <span>{{ $t('Add LoRA') }}</span>
    </button>
  </div>
</template>

<script setup>
import { computed, nextTick, onMounted, onBeforeUnmount, ref } from 'vue'
import { $t } from '@/composable/i18n.js'
import { getWidgetByName, setWidgetValue as setNodeWidgetValue, updateNodeHeight } from '@/composable/node.js'
import { clamp } from '@/composable/utils/widget.js'

const props = defineProps(['widget', 'node'])

const maxRows = computed(() => props.node.widgets?.filter(widget => /^lora_\d+_name$/.test(widget.name)).length ?? 0)
const defaultLoraName = 'None'
const defaultStrength = 1.0
const minStrength = -10
const maxStrength = 10
const globalEnabled = ref(true)
const rows = ref([])
const loraOptions = ref([defaultLoraName])

const toBoolean = (value) => ![false, null, 'False'].includes(value)

const createRow = (index, row = {}) => ({
  key: `lora-row-${index}`,
  enabled: row.enabled ?? true,
  name: row.name ?? defaultLoraName,
  strength: normalizeStrength(row.strength ?? defaultStrength),
})

const getValue = (name, fallback = undefined) => {
  const targetWidget = getWidgetByName(props.node, name)
  return targetWidget ? targetWidget.value : fallback
}

const setWidgetValue = (name, value) => {
  const targetWidget = getWidgetByName(props.node, name)
  if (!targetWidget || targetWidget.value === value) return false

  const widgetIndex = props.node.widgets?.indexOf(targetWidget) ?? -1
  if (widgetIndex < 0) return false

  setNodeWidgetValue(props.node, value, widgetIndex)
  targetWidget.callback?.(value)
  return true
}

const normalizeStrength = (value) => {
  const parsed = Number.parseFloat(value)
  if (Number.isNaN(parsed)) return defaultStrength
  return clamp(parsed, minStrength, maxStrength)
}

const syncStrengthWidgets = (index, strength) => {
  let changed = false
  changed = setWidgetValue(`lora_${index}_strength`, strength) || changed
  changed = setWidgetValue(`lora_${index}_model_strength`, strength) || changed
  changed = setWidgetValue(`lora_${index}_clip_strength`, strength) || changed
  return changed
}

const syncRowWidgets = (index, row) => {
  let changed = false
  changed = setWidgetValue(`lora_${index}_enabled`, row.enabled) || changed
  changed = setWidgetValue(`lora_${index}_name`, row.name) || changed
  changed = syncStrengthWidgets(index, row.strength) || changed
  return changed
}

const resetRowWidgets = (index) => {
  let changed = false
  changed = setWidgetValue(`lora_${index}_enabled`, true) || changed
  changed = setWidgetValue(`lora_${index}_name`, defaultLoraName) || changed
  changed = syncStrengthWidgets(index, defaultStrength) || changed
  return changed
}

const buildRowsFromWidgets = () => {
  const count = Number(getValue('num_loras', 0)) || 0
  const mode = getValue('mode', 'simple')
  const nextRows = []

  for (let index = 1; index <= count; index += 1) {
    const strengthValue = mode === 'advanced'
      ? getValue(`lora_${index}_model_strength`, getValue(`lora_${index}_strength`, defaultStrength))
      : getValue(`lora_${index}_strength`, defaultStrength)

    nextRows.push(createRow(index, {
      enabled: toBoolean(getValue(`lora_${index}_enabled`, true)),
      name: getValue(`lora_${index}_name`, defaultLoraName) || defaultLoraName,
      strength: normalizeStrength(strengthValue),
    }))
  }

  rows.value = nextRows
  globalEnabled.value = toBoolean(getValue('toggle', true))
  loraOptions.value = getWidgetByName(props.node, 'lora_1_name')?.options?.values || [defaultLoraName]
}

const syncRowsToWidgets = ({ updateHeight = false } = {}) => {
  props.node.easyUseLoraStackSyncing = true
  let changed = false
  changed = setWidgetValue('toggle', globalEnabled.value) || changed
  changed = setWidgetValue('num_loras', rows.value.length) || changed

  for (let index = 1; index <= maxRows.value; index += 1) {
    const row = rows.value[index - 1]
    changed = (row ? syncRowWidgets(index, row) : resetRowWidgets(index)) || changed
  }

  if (!changed && !updateHeight) {
    props.node.easyUseLoraStackSyncing = false
    return
  }

  nextTick(() => {
    props.node.easyUseLoraStackSyncing = false
    if (updateHeight) updateNodeHeight(props.node)
  })
}

const updateGlobalEnabled = (value) => {
  globalEnabled.value = value
  syncRowsToWidgets()
}

const toggleRow = (index) => {
  rows.value = rows.value.map((row, rowIndex) => rowIndex === index ? { ...row, enabled: !row.enabled } : row)
  syncRowsToWidgets()
}

const updateRowName = (index, value) => {
  rows.value = rows.value.map((row, rowIndex) => rowIndex === index ? { ...row, name: value } : row)
  syncRowsToWidgets()
}

const updateRowStrength = (index, value) => {
  const strength = normalizeStrength(value)
  rows.value = rows.value.map((row, rowIndex) => rowIndex === index ? { ...row, strength } : row)
  syncRowsToWidgets()
}

const addRow = () => {
  if (rows.value.length >= maxRows.value) return
  rows.value = [...rows.value, createRow(rows.value.length + 1)]
  syncRowsToWidgets({ updateHeight: true })
}

const removeRow = (index) => {
  rows.value = rows.value
    .filter((_, rowIndex) => rowIndex !== index)
    .map((row, rowIndex) => createRow(rowIndex + 1, row))
  syncRowsToWidgets({ updateHeight: true })
}

onMounted(() => {
  props.node.easyUseLoraStackRefresh = () => {
    if (props.node.easyUseLoraStackSyncing) return
    buildRowsFromWidgets()
  }

  setTimeout(() => {
    buildRowsFromWidgets()
    syncRowsToWidgets({ updateHeight: true })
  }, 1)
})

onBeforeUnmount(() => {
  delete props.node.easyUseLoraStackRefresh
  delete props.node.easyUseLoraStackSyncing
})
</script>

<style scoped>
.easyuse-lora-stack-widget {
  display: flex;
  flex-direction: column;
  gap: 8px;
  width: 100%;
  height: 100%;
  padding: 8px;
  color: var(--input-text);
}

.lsw-header {
  display: flex;
  justify-content: flex-start;
}

.lsw-global-toggle {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
}

.lsw-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.lsw-row {
  display: grid;
  grid-template-columns: 32px minmax(0, 1fr) 76px 32px;
  gap: 6px;
  align-items: center;
}

.lsw-toggle-btn,
.lsw-delete-btn,
.lsw-add-btn {
  border: 1px solid var(--border-color);
  background: var(--comfy-input-bg);
  color: var(--input-text);
  border-radius: 6px;
  cursor: pointer;
}

.lsw-toggle-btn,
.lsw-delete-btn {
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.lsw-toggle-btn.active {
  color: var(--theme-color);
}

.lsw-select,
.lsw-weight {
  width: 100%;
  min-height: 32px;
  border: 1px solid var(--border-color);
  background: var(--comfy-input-bg);
  color: var(--input-text);
  border-radius: 6px;
  padding: 0 8px;
}

.lsw-weight {
  text-align: right;
}

.lsw-add-btn {
  min-height: 34px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.lsw-add-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.lsw-empty {
  padding: 8px;
  border: 1px dashed var(--border-color);
  border-radius: 6px;
  color: var(--descrip-text);
  font-size: 12px;
  text-align: center;
}
</style>
