<template>
  <div class="easyuse-lora-stack-widget" @mousedown.stop @wheel.stop>
    <div class="lsw-header">
      <div class="lsw-global-toggle">
        <button
          class="lsw-toggle-switch"
          :class="{ active: globalEnabled }"
          type="button"
          :title="globalEnabled ? $t('Enabled') : $t('Disabled')"
          @click="updateGlobalEnabled(!globalEnabled)"
        >
          <span class="lsw-toggle-thumb" />
        </button>
        <span>{{ globalEnabled ? $t('Enabled') : $t('Disabled') }}</span>
      </div>
      <span class="lsw-strength-label">{{ $t('Strength') }}</span>
      <span class="lsw-header-spacer" />
    </div>

    <div class="lsw-list">
      <div v-for="(row, index) in rows" :key="row.key" class="lsw-row">
        <div class="lsw-pill" :class="{ disabled: !row.enabled }">
          <button
            class="lsw-toggle-switch lsw-row-toggle"
            :class="{ active: row.enabled }"
            type="button"
            :title="row.enabled ? $t('Enabled') : $t('Disabled')"
            @click="toggleRow(index)"
          >
            <span class="lsw-toggle-thumb" />
          </button>

          <select class="lsw-select" :value="row.name" @change="updateRowName(index, $event.target.value)">
            <option v-for="option in loraOptions" :key="option" :value="option">{{ option }}</option>
          </select>

          <div class="lsw-weight-control">
            <button class="lsw-step-btn" type="button" @click="stepRowStrength(index, -1)">
              <i class="pi pi-angle-left" />
            </button>
            <button :title="formatStrength(row.strength)" class="lsw-weight-display" type="button" @click="openRowStrengthPrompt(index, $event)">
              {{ formatStrength(row.strength) }}
            </button>
            <button class="lsw-step-btn" type="button" @click="stepRowStrength(index, 1)">
              <i class="pi pi-angle-right" />
            </button>
          </div>

          <button class="lsw-delete-btn" type="button" :title="$t('Delete')" @click="removeRow(index)">
            <i class="pi pi-trash" />
          </button>

        </div>

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
const strengthStep = 0.01
const strengthPrecision = 2
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

const getGraphCanvas = () => props.node.graph?.list_of_graphcanvas?.[0] || window.LGraphCanvas?.active_canvas || null

const normalizeStrength = (value) => {
  const parsed = Number.parseFloat(value)
  if (Number.isNaN(parsed)) return defaultStrength
  return Number(clamp(parsed, minStrength, maxStrength).toFixed(strengthPrecision))
}

const formatStrength = (value) => normalizeStrength(value).toFixed(strengthPrecision)

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

const stepRowStrength = (index, direction) => {
  const row = rows.value[index]
  if (!row) return
  updateRowStrength(index, row.strength + (direction * strengthStep))
}

const openRowStrengthPrompt = (index, event) => {
  const row = rows.value[index]
  const canvas = getGraphCanvas()
  if (!row || !canvas?.prompt) return

  canvas.prompt($t('Strength'), formatStrength(row.strength), (value) => {
    if (value === null) return
    updateRowStrength(index, value)
  }, event)
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
  display: grid;
  grid-template-columns: minmax(0, 1fr) 80px 28px;
  gap: 6px;
  align-items: center;
}

.lsw-global-toggle {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
  font-size: 12px;
  line-height: 12px;
  color: var(--descrip-text);
}

.lsw-strength-label {
  text-align: center;
  font-size: 12px;
  color: var(--descrip-text);
}

.lsw-header-spacer {
  display: block;
}

.lsw-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.lsw-row {
  display: grid;
  grid-template-columns: minmax(0, 1fr);
  gap: 6px;
  align-items: center;
}

.lsw-pill {
  display: grid;
  grid-template-columns: 34px minmax(0, 1fr) 80px 28px;
  gap: 8px;
  align-items: center;
  min-height: 28px;
  padding: 3px 8px 3px 6px;
  border: 1px solid var(--border-color);
  background: var(--comfy-input-bg);
  border-radius: 999px;
}

.lsw-pill.disabled {
  opacity: 0.58;
}

.lsw-toggle-switch {
  position: relative;
  width: 30px;
  height: 18px;
  border: 1px solid var(--border-color);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  cursor: pointer;
  padding: 0;
  transition: background 0.2s ease, border-color 0.2s ease;
}

.lsw-toggle-switch.active {
  background: rgba(110, 160, 255, 0.22);
  border-color: rgba(110, 160, 255, 0.45);
}

.lsw-toggle-thumb {
  position: absolute;
  top: 1px;
  left: 1px;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #8e8e8e;
  transition: transform 0.2s ease, background 0.2s ease;
}

.lsw-toggle-switch.active .lsw-toggle-thumb {
  transform: translateX(12px);
  background: #9ebcff;
}

.lsw-row-toggle {
  justify-self: center;
}

.lsw-select {
  width: 100%;
  min-width: 0;
  min-height: 28px;
  border: 0;
  outline: 0;
  appearance: none;
  background: var(--comfy-input-bg);
  color: var(--input-text);
  font-size: 12px;
  padding: 0 2px;
}

.lsw-weight-control {
  display: grid;
  grid-template-columns: 24px minmax(0, 1fr) 24px;
  align-items: center;
  min-height: 26px;
  overflow: hidden;
}

.lsw-step-btn,
.lsw-weight-display,
.lsw-delete-btn,
.lsw-add-btn {
  border: 0;
  background: transparent;
  color: var(--input-text);
}

.lsw-step-btn,
.lsw-weight-display,
.lsw-delete-btn {
  min-height: 26px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.lsw-step-btn {
  font-size: 12px;
}

.lsw-weight-display {
  min-width: 0;
  padding: 0 4px;
  text-align: center;
  font-variant-numeric: tabular-nums;
}

.lsw-delete-btn {
  width: 28px;
  height: 28px;
  border: 0;
  border-radius: 999px;
  background: transparent;
  opacity: 0.72;
}

.lsw-delete-btn:hover,
.lsw-step-btn:hover,
.lsw-weight-display:hover,
.lsw-add-btn:hover,
.lsw-toggle-switch:hover {
  opacity: 1;
}

.lsw-add-btn {
  min-height: 34px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  border: 1px solid var(--border-color);
  background: var(--comfy-input-bg);
  border-radius: 8px;
  cursor: pointer;
}

.lsw-add-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.lsw-empty {
  padding: 8px;
  border: 1px dashed var(--border-color);
  border-radius: 8px;
  color: var(--descrip-text);
  font-size: 12px;
  text-align: center;
}
</style>
