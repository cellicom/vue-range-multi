<script lang="ts" setup generic="T = any, U = RangeValueType<T>">
import { computed, nextTick, ref, watch, defineComponent } from 'vue'
import type { CSSProperties, VNode } from 'vue'

// --- Types ---
export type RangeValueType<T> = number | RangeData<T>
export interface RangeData<T, U = RangeValueType<T>> {
  value: number
  data?: T
  disabled?: boolean
  unremovable?: boolean
  type?: string
  limits?: [number, number]
  renderTop?: RangeRenderFn<T, U>
  renderBottom?: RangeRenderFn<T, U>
}
export type RangeRenderFn<T, U = RangeValueType<T>> = (data: U) => VNode
export type RangeValue<T, U = RangeValueType<T>> = U | U[]
export type RangeProgress = ([number, number] | {
  range: [number, number]
  style?: CSSProperties
  class?: string
})[]
export type RangeMarks = Record<number, string | {
  label: string
  style?: CSSProperties
  class?: string
}>

// --- Utils ---
function swap<T extends object>(arr: T, i: keyof T, j: keyof T): void {
  const temp = arr[i]
  arr[i] = arr[j]
  arr[j] = temp
}

function percentage2value(percentage: number, min: number, max: number, step: number) {
  if (min === undefined || max === undefined || step === undefined)
    return Number.NaN
  if (max <= min)
    return Number.NaN
  if (percentage < 0)
    return min
  if (percentage > 100)
    return max
  const value = min + (max - min) * percentage / 100
  return Math.round(value / step) * step
}

function value2percentage(value: number, min: number, max: number, step: number) {
  if (min === undefined || max === undefined || step === undefined)
    return Number.NaN
  if (max <= min)
    return Number.NaN
  if (value < min)
    return 0
  if (value > max)
    return 100
  const percentage = (value - min) / (max - min) * 100
  const percentageStep = step / (max - min)
  return Math.round(percentage / percentageStep) * percentageStep
}

// --- Render Component ---
const RenderFunction = defineComponent({
  props: { render: Function },
  setup(props) {
    return () => props.render ? props.render() : null
  }
})

// --- Component Props ---
const props = withDefaults(defineProps<{
  modelValue: RangeValue<T, U>
  min?: number
  max?: number
  step?: number
  vertical?: boolean
  thumbLimits?: [number, number]
  addable?: boolean
  addData?: (value: number) => RangeData<T, U>
  limit?: number
  smooth?: boolean
  deduplicate?: boolean
  rangeHighlight?: boolean
  progress?: RangeProgress
  showStops?: boolean | number
  size?: 'small' | 'medium' | 'large'
  thumbType?: 'circle' | 'square' | 'rect'
  thumbSize?: 'small' | 'medium' | 'large'
  renderTop?: RangeRenderFn<T, U>
  renderTopOnActive?: boolean
  renderBottom?: RangeRenderFn<T, U>
  renderBottomOnActive?: boolean
  marks?: RangeMarks
}>(), {
  modelValue: () => [],
  min: 0,
  max: 100,
  step: 1,
  deduplicate: true,
  showStops: 12,
  size: 'small',
  thumbSize: 'medium',
})

const emits = defineEmits<{
  (e: 'update:modelValue', modelValue: typeof props.modelValue): void
  (e: 'change', value: RangeValue<T, U>, thumbValue: U, thumbIndex: number): void
}>()

defineSlots<{
  top?: (props: { data: U }) => any
  bottom?: (props: { data: U }) => any
}>()

// --- Logic ---
const modelType = computed<'number' | 'data' | 'numberList' | 'dataList'>(() => {
  const value = props.modelValue
  if (Array.isArray(value) && typeof value[0] === 'number')
    return 'numberList'
  else if (Array.isArray(value))
    return 'dataList'
  else if (typeof value === 'number')
    return 'number'
  else if (typeof value === 'object')
    return 'data'
  else
    console.warn('[vue-range-multi]: modelValue type not support', value)
  return 'number'
})

const model = computed<RangeData<T, U>[]>({
  get: () => {
    const value = props.modelValue
    if (Array.isArray(value))
      return value.map(item => typeof item === 'number' ? { value: (item as number) } : (item as RangeData<T, U>))
    else
      return [typeof value === 'number' ? { value: (value as number) } : (value as RangeData<T, U>)]
  },
  set: (value) => {
    let res: any
    if (modelType.value === 'number')
      res = value[0].value
    else if (modelType.value === 'data')
      res = value[0]
    else if (modelType.value === 'numberList')
      res = value.map(item => (item.value))
    else
      res = value
    emits('update:modelValue', res)
  },
})

const allowAdd = computed(() =>
  props.addable
  && (!props.limit || model.value.length < props.limit)
  && !['data', 'number'].includes(modelType.value),
)

const stops = computed(() => {
  const stopsCount = Math.floor((props.max - props.min) / props.step) + 1
  if (props.showStops === true)
    return stopsCount
  else if (typeof props.showStops === 'number')
    return stopsCount > props.showStops ? -1 : stopsCount
  else
    return -1
})

const indexMap = ref<Record<number, number>>({})
const indexMapReversed = computed(() => Object.fromEntries(Object.entries(indexMap.value).map(([k, v]) => [v, Number.parseInt(k)])))

function sort(val: RangeData<T, U>[]) {
  const valMap = val.map((v, i) => ({ v: v.value, i, raw: v }))
  valMap.sort((a, b) => a.v - b.v)
  const changed = valMap.some((v, i) => v.i !== i)
  if (!changed)
    return
  const sortMap = Object.fromEntries(valMap.map((v, i) => [v.i, i]))
  indexMap.value = Object.fromEntries(Object.entries(indexMap.value).map(([k, v]) => [k, sortMap[v]]))
  nextTick(() => model.value = valMap.map(v => v.raw))
}

function getAddableIndex(arr: number[]) {
  [...arr].sort()
  let i = 0
  for (; i < arr.length; i++) {
    if (i !== arr[i])
      return i
  }
  return i
}

watch(model, (val) => {
  const length = Object.keys(indexMap.value).length
  if (val.length > length) {
    for (let i = length; i < val.length; i++)
      indexMap.value[getAddableIndex(Object.keys(indexMap.value).map(key => Number.parseInt(key)))] = i
    sort(val)
  }
  else if (val.length < length) {
    for (let i = length - 1; i >= val.length; i -= 1) {
      const index = indexMapReversed.value[i]
      delete indexMap.value[index]
    }
    sort(val)
  }
}, {
  immediate: true,
})

const trackRef = ref<HTMLElement>()

function getValue(percentage: number) {
  return percentage2value(percentage, props.min, props.max, props.step)
}

function getPercentage(value: number) {
  return value2percentage(value, props.min, props.max, props.step)
}

const current = ref<number>(-1)
const currentPercentage = ref<number>(-1)
let currentPercentageTimer: number

function setCurrentPercentage(val: number) {
  currentPercentage.value = val
  clearTimeout(currentPercentageTimer)
  currentPercentageTimer = window.setTimeout(() => {
    currentPercentage.value = -1
  }, 300)
}

const position = computed(() => Object.fromEntries(Object.entries(indexMap.value).map(
  ([idx, index]) => (props.smooth && current.value === Number.parseInt(idx) && currentPercentage.value > -1)
    ? [idx, currentPercentage.value]
    : [idx, getPercentage(model.value[index].value)],
)))

function onUpdate(percentage: number) {
  setCurrentPercentage(percentage)
  const value = getValue(percentage)
  const modelValue = model.value
  const values = modelValue.map(i => i.value)
  if (props.deduplicate && values.includes(value))
    return
  let index = indexMap.value[current.value]
  const oldValue = values[index]

  const thumbData = modelValue[index]
  const individualLimits = thumbData.limits
  const globalLimits = props.thumbLimits

  if (individualLimits) {
    if (value < individualLimits[0] || value > individualLimits[1]) return
  }
  else if (globalLimits) {
    if (value < globalLimits[0] || value > globalLimits[1]) return
  }

  if (oldValue - value > 0 && index > 0) {
    for (let i = index; i > 0; i -= 1) {
      const prev = values[i - 1]
      if (value < prev) {
        swap(modelValue, i, i - 1)
        swap(indexMap.value, indexMapReversed.value[i], indexMapReversed.value[i - 1])
        index -= 1
      }
    }
  }
  if (oldValue - value < 0 && index < modelValue.length - 1) {
    for (let i = index; i < modelValue.length - 1; i++) {
      const next = values[i + 1]
      if (value > next) {
        swap(modelValue, i, i + 1)
        swap(indexMap.value, indexMapReversed.value[i], indexMapReversed.value[i + 1])
        index += 1
      }
    }
  }
  modelValue[index].value = value
  model.value = modelValue
}

function onDelete() {
  const valueIndex = indexMap.value[current.value]
  const modelValue = model.value
  modelValue.splice(valueIndex, 1)
  model.value = modelValue
  delete indexMap.value[current.value]
  indexMap.value = Object.fromEntries(Object.entries(indexMap.value).map(([idx, index]) => index >= valueIndex ? [idx, index - 1] : [idx, index]))
}

// === Thumb Logic ===
const removable = (index: number) => props.addable && !model.value[index].unremovable
const thumbDeleting = ref(false)
let thumbMoved = false
const activeThumbType = ref<string | undefined>(undefined)

function setTrackCursor(cursor: string | undefined) {
  if (!trackRef?.value) return
  if (cursor) trackRef.value.style.cursor = cursor.replace('cursor-', '')
  else trackRef.value.style.removeProperty('cursor')
}

function shouldDelete(offset: number) {
  if (!trackRef?.value) return false
  const trackRect = trackRef.value.getBoundingClientRect()
  if (props.vertical) {
    const left = trackRect.left - 32
    const right = trackRect.right + 32
    return offset < left || offset > right
  } else {
    const top = trackRect.top - 32
    const bottom = trackRect.bottom + 32
    return offset < top || offset > bottom
  }
}

function onThumbPointerMove(e: PointerEvent) {
  if (current.value === -1 || !trackRef?.value) return
  const index = indexMap.value[current.value]
  if (model.value[index].disabled) return
  
  thumbMoved = true
  const trackRect = trackRef.value.getBoundingClientRect()
  const offset = props.vertical ? e.clientY - trackRect.top : e.clientX - trackRect.left
  const percent = (offset / (props.vertical ? trackRect.height : trackRect.width)) * 100
  
  let p = percent
  if (percent < 0) p = 0
  if (percent > 100) p = 100
  
  if (!Number.isNaN(p)) {
    onUpdate(p)
  }
  
  if (removable(index)) {
    thumbDeleting.value = shouldDelete(props.vertical ? e.clientX : e.clientY)
  }
}

function onThumbPointerUp(e: PointerEvent) {
  window.removeEventListener('pointermove', onThumbPointerMove)
  window.removeEventListener('pointerup', onThumbPointerUp)
  window.removeEventListener('pointercancel', onThumbPointerUp)
  
  if (current.value === -1) return
  const index = indexMap.value[current.value]
  const disabled = model.value[index].disabled

  if (thumbMoved) {
    const modelToEmit = Array.isArray(props.modelValue) ? (props.modelValue as any)[index] : props.modelValue as any
    emits('change', props.modelValue, modelToEmit, index)
    thumbMoved = false
  }
  
  if (e.type === 'pointercancel' || disabled) {
    setTrackCursor(undefined)
    current.value = -1
    return
  }
  
  if (removable(index)) {
    if (shouldDelete(props.vertical ? e.clientX : e.clientY)) {
      thumbDeleting.value = false
      onDelete()
    }
  }
  
  setTrackCursor(undefined)
  current.value = -1
}

async function onThumbPointerDown(e: PointerEvent, idx: number, index: number) {
  e.preventDefault()
  e.stopPropagation()
  
  const target = e.target as HTMLElement
  const isThumbTarget = target.classList.contains('m-range-thumb') || target.classList.contains('m-range-v-thumb')
  
  current.value = idx
  
  const thumbType = model.value[index].type
  if (thumbType) {
    activeThumbType.value = thumbType
  }
  
  if (!isThumbTarget) {
      await nextTick()
      current.value = -1
      return
  }

  const cursorStyle = model.value[index].disabled 
    ? 'cursor-not-allowed' 
    : removable(index) 
      ? 'cursor-move' 
      : props.vertical 
        ? 'cursor-ns-resize' 
        : 'cursor-ew-resize'

  setTrackCursor(cursorStyle)
  thumbMoved = false
  window.addEventListener('pointermove', onThumbPointerMove, { passive: false })
  window.addEventListener('pointerup', onThumbPointerUp)
  window.addEventListener('pointercancel', onThumbPointerUp)
}

// === Track Event Logic ===
let addTiming = false
function addThumb(e: MouseEvent) {
  if (!addTiming) return
  addTiming = false
  if (!trackRef?.value || !allowAdd.value) return
  const trackRect = trackRef.value.getBoundingClientRect()
  const offset = props.vertical ? e.clientY - trackRect.top : e.clientX - trackRect.left
  const percent = offset / (props.vertical ? trackRect.height : trackRect.width) * 100
  const value = getValue(percent)
  
  if (model.value.some(item => item.value === value)) return
  
  const modelValue = model.value
  if (modelType.value === 'numberList') {
    modelValue.push({ value })
    model.value = modelValue
  }
  else if (modelType.value === 'dataList') {
    modelValue.push(props.addData ? props.addData(value) : { value })
    model.value = modelValue
  }
}
</script>

<template>
  <div
    class="m-range-theme m-range"
    :class="[
      `m-range-${vertical ? 'v-' : ''}${size}`, 
      `m-range-${vertical ? 'v-' : ''}thumb-${thumbSize}`,
      activeThumbType ? `m-range-${activeThumbType}` : ''
    ]"
  >
    <div
      ref="trackRef"
      :class="[vertical ? 'm-range-v-track' : 'm-range-track', { 'cursor-copy': allowAdd }]"
      @pointerdown="addTiming = true"
      @pointerleave="addTiming = false"
      @pointerup.prevent="addThumb"
    >
      <div v-show="rangeHighlight && model.length >= 2" class="m-range-highlight-container">
        <div
          :class="vertical ? 'm-range-v-highlight' : 'm-range-highlight'"
          :style="vertical
            ? { top: `${Math.min(...Object.values(position))}%`, bottom: `${100 - Math.max(...Object.values(position))}%` }
            : { left: `${Math.min(...Object.values(position))}%`, right: `${100 - Math.max(...Object.values(position))}%` }"
        />
      </div>
      
      <div v-if="progress?.length" class="m-range-progress-container">
        <template v-for="(bar, idx) in progress" :key="idx">
          <div
            v-if="Array.isArray(bar)"
            :class="vertical ? 'm-range-v-progress' : 'm-range-progress'"
            :style="vertical
              ? { top: `${getPercentage(bar[0])}%`, bottom: `${100 - getPercentage(bar[1])}%` }
              : { left: `${getPercentage(bar[0])}%`, right: `${100 - getPercentage(bar[1])}%` }"
          />
          <div
            v-else
            :class="[vertical ? 'm-range-v-progress' : 'm-range-progress', bar.class]"
            :style="vertical
              ? { top: `${getPercentage(bar.range[0])}%`, bottom: `${100 - getPercentage(bar.range[1])}%`, ...bar.style }
              : { left: `${getPercentage(bar.range[0])}%`, right: `${100 - getPercentage(bar.range[1])}%`, ...bar.style }"
          />
        </template>
      </div>
      
      <div v-if="stops > 0" class="m-range-points-area">
        <div :class="vertical ? 'm-range-v-points-container' : 'm-range-points-container'">
          <div v-for="index in stops" :key="index" class="m-range-points" />
        </div>
      </div>
      
      <div v-if="marks" :class="vertical ? 'm-range-v-marks' : 'm-range-marks'">
        <template v-for="(mark, key) in marks" :key="key">
          <div
            v-if="typeof mark === 'string'"
            :class="vertical ? 'm-range-v-mark-item' : 'm-range-mark-item'"
            :style="vertical ? { top: `${getPercentage(Number(key))}%` } : { left: `${getPercentage(Number(key))}%` }"
          >
            {{ mark }}
          </div>
          <div
            v-else
            :class="[vertical ? 'm-range-v-mark-item' : 'm-range-mark-item', mark.class]"
            :style="vertical ? { top: `${getPercentage(Number(key))}%` } : { left: `${getPercentage(Number(key))}%`, ...mark.style }"
          >
            {{ mark.label }}
          </div>
        </template>
      </div>

      <!-- Thumbs -->
      <div
        v-for="(index, idx) in indexMap"
        :key="idx"
        :class="[
          vertical ? 'm-range-v-thumb' : 'm-range-thumb',
          {
            'm-range-thumb-active': current === Number(idx),
            'op-20': removable(index) && thumbDeleting && current === Number(idx),
            'cursor-not-allowed': model[index].disabled,
            'cursor-move': !model[index].disabled && removable(index),
            'cursor-ns-resize': !model[index].disabled && !removable(index) && vertical,
            'cursor-ew-resize': !model[index].disabled && !removable(index) && !vertical,
          },
          `m-range-${vertical ? 'v-' : ''}thumb-${thumbType || (size === 'large' ? 'rect' : 'circle')}`,
          `m-range-${vertical ? 'v-' : ''}thumb-${thumbSize}`,
          model[index].type ? `m-range-${model[index].type}` : ''
        ]"
        :style="vertical ? { top: `${position[idx] || 0}%` } : { left: `${position[idx] || 0}%` }"
        @pointerdown="onThumbPointerDown($event, Number(idx), index)"
      >
        <Transition name="fade">
          <div v-if="!renderTopOnActive || current === Number(idx)" class="cursor-default" :class="vertical ? 'm-range-v-transition-container' : 'm-range-transition-container'">
            <div :class="vertical ? 'm-range-v-thumb-top-container' : 'm-range-thumb-top-container'">
              <RenderFunction v-if="model[index].renderTop || renderTop" :render="() => (model[index].renderTop || renderTop)?.((['data', 'dataList'].includes(modelType) ? model[index] : model[index].value) as U)" />
              <slot v-else name="top" :data="(['data', 'dataList'].includes(modelType) ? model[index] : model[index].value) as U" />
            </div>
          </div>
        </Transition>
        <Transition name="fade">
          <div v-if="!renderBottomOnActive || current === Number(idx)" class="cursor-default" :class="vertical ? 'm-range-v-transition-container' : 'm-range-transition-container'">
            <div :class="vertical ? 'm-range-v-thumb-bottom-container' : 'm-range-thumb-bottom-container'">
              <RenderFunction v-if="model[index].renderBottom || renderBottom" :render="() => (model[index].renderBottom || renderBottom)?.((['data', 'dataList'].includes(modelType) ? model[index] : model[index].value) as U)" />
              <slot v-else name="bottom" :data="(['data', 'dataList'].includes(modelType) ? model[index] : model[index].value) as U" />
            </div>
          </div>
        </Transition>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Theme Variables */
.m-range-theme {
  --c-primary: #409EFF;
  --c-fill: #E4E7ED;
  --c-fill-stop: #F5F5F5;
  --c-fill-thumb: #fff;
  --c-drop: var(--c-primary);
}
.dark .m-range-theme {
  --c-primary: #3070ED;
  --c-fill: #444;
  --c-fill-stop: #555;
  --c-fill-thumb: #333;
}

/* Base & Sizes */
.m-range {
  min-height: 0.25rem;
  min-width: 0.25rem;
  box-sizing: content-box;
}

.m-range-small { height: 0.5rem; --m-range-h: 0.5rem; }
.m-range-medium { height: 1rem; --m-range-h: 1rem; }
.m-range-large { height: 2rem; --m-range-h: 2rem; }

.m-range-v-small { width: 0.5rem; --m-range-w: 0.5rem; }
.m-range-v-medium { width: 1rem; --m-range-w: 1rem; }
.m-range-v-large { width: 2rem; --m-range-w: 2rem; }

/* Track */
.m-range-track {
  position: relative;
  height: 100%;
  background-color: var(--c-fill);
  user-select: none;
  border-radius: 9999px;
}
.m-range-v-track {
  position: relative;
  width: 100%;
  height: 100%;
  background-color: var(--c-fill);
  user-select: none;
  border-radius: 9999px;
}

/* Highlight & Progress */
.m-range-highlight-container, .m-range-progress-container {
  position: absolute;
  height: 100%;
  width: 100%;
  overflow: hidden;
  border-radius: inherit;
}
.m-range-highlight, .m-range-progress {
  height: 100%;
  background-color: var(--c-primary);
  position: absolute;
}
.m-range-v-highlight, .m-range-v-progress {
  width: 100%;
  background-color: var(--c-primary);
  position: absolute;
}

/* Points */
.m-range-points-area {
  position: absolute;
  height: 100%;
  width: 100%;
  border-radius: inherit;
  overflow: hidden;
}
.m-range-points-container {
  position: absolute;
  height: 100%;
  left: -3px;
  right: -3px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.m-range-v-points-container {
  position: absolute;
  width: 100%;
  top: -3px;
  bottom: -3px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
}
.m-range-points {
  width: 6px;
  height: 6px;
  border-radius: 3px;
  background-color: var(--c-fill-stop);
}

/* Marks */
.m-range-marks {
  position: absolute;
  height: 100%;
  width: 100%;
  left: 0;
  top: 0;
  transform: translateY(calc(var(--m-range-h) + var(--m-range-thumb-h)));
}
.m-range-mark-item {
  position: absolute;
  top: 0;
  transform: translateX(-50%);
}
.m-range-v-marks {
  position: absolute;
  height: 100%;
  width: 100%;
  left: 0;
  top: 0;
  transform: translateX(calc(var(--m-range-w) + var(--m-range-thumb-w)));
}
.m-range-v-mark-item {
  position: absolute;
  left: 0;
  transform: translateY(-50%);
}

/* Thumb */
.m-range-thumb, .m-range-v-thumb {
  border: 2px solid var(--c-primary);
  touch-action: none;
  position: absolute;
  background-color: var(--c-fill-thumb);
  transform-origin: center;
  transition-property: opacity, transform;
  transition-duration: 150ms;
}
.m-range-thumb {
  transform: translateX(-50%);
}
.m-range-v-thumb {
  transform: translateY(-50%);
}

.m-range-thumb-circle { width: calc(var(--m-range-h) + var(--m-range-thumb-h) * 2); border-radius: 9999px; }
.m-range-thumb-square { width: calc(var(--m-range-h) + var(--m-range-thumb-h) * 2); }
.m-range-thumb-rect { width: 0.75rem; border-radius: 9999px; }

.m-range-v-thumb-circle { height: calc(var(--m-range-w) + var(--m-range-thumb-w) * 2); border-radius: 9999px; }
.m-range-v-thumb-square { height: calc(var(--m-range-w) + var(--m-range-thumb-w) * 2); }
.m-range-v-thumb-rect { height: 0.75rem; border-radius: 9999px; }

.m-range-thumb-small { top: 0px; bottom: 0px; --m-range-thumb-h: 0rem; }
.m-range-thumb-medium { top: -0.25rem; bottom: -0.25rem; --m-range-thumb-h: 0.25rem; }
.m-range-thumb-large { top: -0.5rem; bottom: -0.5rem; --m-range-thumb-h: 0.5rem; }

.m-range-v-thumb-small { left: 0px; right: 0px; --m-range-thumb-w: 0rem; }
.m-range-v-thumb-medium { left: -0.25rem; right: -0.25rem; --m-range-thumb-w: 0.25rem; }
.m-range-v-thumb-large { left: -0.5rem; right: -0.5rem; --m-range-thumb-w: 0.5rem; }

.m-range-thumb-active { z-index: 1; filter: drop-shadow(0.1rem 0.15rem 0.25rem var(--c-drop)); }

/* Slots Transitions */
.m-range-transition-container { position: absolute; height: 100%; width: 0; left: 50%; }
.m-range-thumb-top-container { position: absolute; left: 50%; top: 0; transform: translateX(-50%) translateY(calc(-100% - 4px)); }
.m-range-thumb-bottom-container { position: absolute; left: 50%; bottom: 0; transform: translateX(-50%) translateY(calc(100% + 4px)); }

.m-range-v-transition-container { position: absolute; width: 100%; height: 0; top: 50%; }
.m-range-v-thumb-top-container { position: absolute; top: 50%; left: 0; transform: translateY(-50%) translateX(calc(-100% - 4px)); }
.m-range-v-thumb-bottom-container { position: absolute; top: 50%; right: 0; transform: translateY(-50%) translateX(calc(100% + 4px)); }

/* Cursors & Utilities */
.cursor-copy { cursor: copy; }
.cursor-move { cursor: move; }
.cursor-not-allowed { cursor: not-allowed; }
.cursor-ns-resize { cursor: ns-resize; }
.cursor-ew-resize { cursor: ew-resize; }
.cursor-default { cursor: default; }
.op-20 { opacity: 0.2; }

/* Animations */
.fade-enter-active { animation: fade 0.3s; }
.fade-leave-active { animation: fade 0.3s reverse; }
@keyframes fade {
  from { opacity: 0; transform: scale(0.5); }
  to { opacity: 1; transform: scale(1); }
}
</style>
