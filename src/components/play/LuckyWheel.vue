<template>
  <view class="wheel-tool app-card">
    <view class="wheel-head">
      <view class="wheel-copy">
        <text class="wheel-kicker">
          {{ kicker }}
        </text>
        <text class="wheel-title">
          {{ activePreset.title }}
        </text>
        <text class="wheel-subtitle">
          {{ subtitle || '按权重随机，转盘面朝下的那一格就是结果。' }}
        </text>
      </view>
      <view class="wheel-actions">
        <button
          class="wheel-chip"
          :class="{ 'wheel-chip--off': !soundOn }"
          hover-class="none"
          @tap="toggleWheelSound"
          @longpress="showSoundDiagnostics"
        >
          <text>{{ soundOn ? '音效开' : '音效关' }}</text>
        </button>
        <button class="wheel-chip" hover-class="none" @tap="openEditor">
          <Pencil :size="14" />
          <text>{{ customEnabled ? '编辑' : '登录后编辑' }}</text>
        </button>
        <button class="wheel-chip" hover-class="none" @tap="createPreset">
          <Plus :size="14" />
          <text>{{ customEnabled ? '新建' : '登录后新建' }}</text>
        </button>
      </view>
    </view>

    <!-- 转盘切换：放在最上方，一眼就能看到"可以存多个、随时切换" -->
    <view class="wheel-presets">
      <view class="wheel-presets__head">
        <text class="wheel-presets__title">我的转盘</text>
        <text class="wheel-presets__count">{{ wheelList.length }} 个 · 点一下切换</text>
      </view>
      <view class="wheel-presets__list">
        <button
          v-for="preset in wheelList"
          :key="preset.id"
          class="preset-pill"
          :class="{ active: preset.id === activePresetId }"
          hover-class="none"
          @tap="selectPreset(preset.id)"
        >
          <text class="preset-pill__text">{{ preset.title }}</text>
          <text v-if="preset.custom" class="preset-pill__badge">自定义</text>
        </button>
      </view>
    </view>

    <view class="wheel-stage">
      <view class="wheel-shadow" />

      <view class="wheel-tilt">
        <view class="wheel-body">
          <view
            v-for="layer in rimLayers"
            :key="`rim-${layer}`"
            class="wheel-rim-layer"
            :style="{ transform: `translateZ(${layer * -2}rpx)` }"
          />

          <view
            class="wheel-disc"
            :style="discStyle"
          >
            <view class="wheel-face" :style="faceStyle" />
            <view class="wheel-face-gloss" />
            <view class="wheel-face-ring" />

            <view
              v-for="(sector, index) in sectors"
              :key="`sector-${index}`"
              class="wheel-sector"
              :style="{ transform: `rotate(${sector.center}deg)` }"
            >
              <view class="wheel-sector__rail">
                <text
                  class="wheel-sector__label"
                  :class="[sector.flip ? 'is-flip' : '', `is-len-${sector.size}`]"
                >
                  {{ sector.text }}
                </text>
              </view>
            </view>

            <view
              v-for="(peg, index) in pegs"
              :key="`peg-${index}`"
              class="wheel-peg"
              :style="{ transform: `rotate(${peg}deg) translateY(-248rpx)` }"
            />

            <view class="wheel-speed" :style="{ opacity: speedOpacity }" />
          </view>
        </view>

        <view class="wheel-hub">
          <view class="wheel-hub__ring" />
          <view class="wheel-hub__plate">
            <text class="wheel-hub__value">
              {{ resultCard.label || '待转' }}
            </text>
            <button
              class="wheel-hub__button"
              hover-class="none"
              :disabled="spinning || !activePreset.options.length"
              @tap="spin"
            >
              <text>{{ spinning ? '转动中' : '开始转' }}</text>
            </button>
          </view>
        </view>
      </view>

      <view
        class="wheel-pointer"
        :style="{ transform: `rotate(${flapAngle}deg)` }"
      >
        <view class="wheel-pointer__cap" />
        <view class="wheel-pointer__blade" />
      </view>
    </view>

    <view class="wheel-meta">
      <view class="wheel-meta__item">
        <text>{{ activePreset.options.length }}</text>
        <text>个扇区</text>
      </view>
      <view class="wheel-meta__item">
        <text>{{ totalWeight }}</text>
        <text>总权重</text>
      </view>
      <view class="wheel-meta__item">
        <text>{{ wheelList.length }}</text>
        <text>个转盘</text>
      </view>
    </view>

    <view class="wheel-result" :class="{ 'is-visible': !!resultCard.label }">
      <view class="wheel-result__left">
        <text class="wheel-result__label">
          {{ spinning ? '正在减速…' : '本次结果' }}
        </text>
        <text class="wheel-result__value">
          {{ spinning ? '转盘转动中' : (resultCard.label || '还没有结果') }}
        </text>
        <text class="wheel-result__detail">
          {{ spinning ? '指针停下的位置就是答案' : (resultCard.detail || '点转盘中间的按钮开始') }}
        </text>
      </view>
      <button
        v-if="resultCard.label && !spinning"
        class="wheel-result__again"
        hover-class="none"
        @tap="spin"
      >
        再来一次
      </button>
    </view>

    <view class="editor-mask" v-if="showEditor" @tap="showEditor = false">
      <view class="editor-sheet app-card" @tap.stop>
        <view class="editor-head">
          <view>
            <text class="editor-title">
              自定义转盘
            </text>
            <text class="editor-subtitle">
              权重越大，被转到的机会越高。
            </text>
          </view>
          <button class="editor-close" hover-class="none" @tap="showEditor = false">
            <X :size="16" />
          </button>
        </view>

        <view class="editor-field">
          <text>转盘标题</text>
          <input
            v-model="draft.title"
            type="text"
            maxlength="16"
            placeholder="例如：今天吃什么"
            placeholder-class="field-placeholder"
          >
        </view>

        <view
          v-for="(option, index) in draft.options"
          :key="option.key"
          class="editor-row"
        >
          <input
            v-model="option.label"
            class="editor-input"
            type="text"
            maxlength="12"
            placeholder="选项名称"
            placeholder-class="field-placeholder"
          >
          <input
            v-model="option.weight"
            class="editor-weight"
            type="number"
            placeholder="1"
            placeholder-class="field-placeholder"
          >
          <button class="editor-remove" hover-class="none" @tap="removeOption(index)">
            <Trash2 :size="14" />
          </button>
        </view>

        <view class="editor-actions">
          <button class="editor-add" hover-class="none" @tap="addOption">
            <Plus :size="14" />
            <text>增加选项</text>
          </button>
          <button
            v-if="activePreset.custom"
            class="editor-delete"
            hover-class="none"
            @tap="removePreset"
          >
            删除转盘
          </button>
          <button class="editor-save" hover-class="none" @tap="saveDraft">
            保存
          </button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import { Pencil, Plus, Trash2, X } from '@/components/icons'
import {
  deleteWheelPreset,
  getWheelPresets,
  saveWheelPreset,
  spinWheel
} from '@/utils/play-data'
import {
  getSoundDiagnostics,
  isSoundOn,
  playWheelTick,
  prepareGameSound,
  releaseGameSound,
  toggleSoundOn,
  unlockGameSound
} from '@/utils/game-sound'

const props = defineProps({
  scope: {
    type: String,
    default: 'guest'
  },
  /** 游客不给自定义：转盘是免费玩法里唯一带编辑器的，登录后才开放 */
  customEnabled: {
    type: Boolean,
    default: true
  },
  kicker: {
    type: String,
    default: 'LUCKY WHEEL'
  },
  subtitle: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['result', 'audio'])

/*
 * 扇区配色：同样是六个色相，但做了两件事——
 * 1) 相邻扇区一定冷暖交替（粉→青→橙→紫→金→玫红），不会出现两块相近的挨在一起；
 * 2) 明度整体压一档，去掉荧光感，配白字更稳。
 */
const palette = ['#E9547A', '#3FB0A0', '#F0884A', '#7C68D8', '#E4B03E', '#D96FAE']
const rimLayers = 10

const wheelList = ref([])
const activePresetId = ref('')
const rotation = ref(0)
const spinning = ref(false)
const flapAngle = ref(0)
const spinSpeed = ref(0)
const soundOn = ref(isSoundOn())
const resultCard = ref({})
const showEditor = ref(false)
const draft = reactive({ id: '', title: '', options: [] })

let rafTimer = null
let flapVelocity = 0
let lastPeg = 0
let lastTickAt = 0
let pendingResult = null

const activePreset = computed(() => {
  return wheelList.value.find((item) => item.id === activePresetId.value) || wheelList.value[0] || {
    id: '',
    title: '还没有转盘',
    options: []
  }
})

const totalWeight = computed(() =>
  (activePreset.value.options || []).reduce((sum, item) => sum + Math.max(Number(item.weight || 1), 1), 0)
)

/**
 * 权重决定扇区角度，角度决定视觉体积，和真实的抽奖盘一致。
 */
const sectors = computed(() => {
  const options = activePreset.value.options || []
  if (!options.length) return []
  const total = totalWeight.value || options.length
  let cursor = 0
  return options.map((option) => {
    const weight = Math.max(Number(option.weight || 1), 1)
    const angle = (weight / total) * 360
    const start = cursor
    cursor += angle
    const center = start + angle / 2
    const text = String(option.label || '')
    return {
      start,
      angle,
      center,
      text: text.length > 7 ? `${text.slice(0, 6)}…` : text,
      size: Math.min(text.length, 4),
      flip: center > 90 && center < 270
    }
  })
})

const faceStyle = computed(() => {
  const list = sectors.value
  if (!list.length) {
    return {
      background: 'conic-gradient(from 0deg, #ffe9f0 0deg 360deg)'
    }
  }
  const stops = list.map((sector, index) => {
    const color = palette[index % palette.length]
    return `${color} ${sector.start}deg ${sector.start + sector.angle}deg`
  })
  return {
    background: `conic-gradient(from 0deg, ${stops.join(', ')})`
  }
})

const discStyle = computed(() => ({
  transform: `translateZ(6rpx) rotate(${rotation.value}deg)`
}))

const pegs = computed(() => {
  const list = sectors.value
  if (!list.length) return []
  return list.map((sector) => sector.start)
})

const speedOpacity = computed(() =>
  Math.min(0.42, Math.max(0, spinSpeed.value) * 0.42).toFixed(3)
)

function refreshPresets () {
  wheelList.value = getWheelPresets(props.scope)
  if (!activePresetId.value || !wheelList.value.some((item) => item.id === activePresetId.value)) {
    activePresetId.value = wheelList.value[0]?.id || ''
  }
}

function selectPreset (id) {
  if (spinning.value) return
  activePresetId.value = id
  resultCard.value = {}
}

function hapticTick () {
  const now = Date.now()
  if (now - lastTickAt < 90) return
  lastTickAt = now
  // #ifdef APP-PLUS
  try {
    uni.vibrateShort({ fail: () => {} })
  } catch (error) {
    // 部分机型不支持震动，忽略即可
  }
  // #endif
}

function stopLoop () {
  if (rafTimer) {
    clearInterval(rafTimer)
    rafTimer = null
  }
}

/**
 * 用时间缓动驱动旋转：先加速冲出，再长距离减速，最后带一点回弹。
 * 定时器方案在 H5 / App / 小程序上表现一致。
 */
function runSpin (from, to, duration) {
  const startAt = Date.now()
  let previousAngle = from
  let previousAt = startAt

  stopLoop()
  rafTimer = setInterval(() => {
    const now = Date.now()
    const elapsed = now - startAt
    const progress = Math.min(elapsed / duration, 1)
    const eased = 1 - Math.pow(1 - progress, 4.1)
    const back = progress > 0.86
      ? Math.sin((progress - 0.86) / 0.14 * Math.PI) * 0.9
      : 0
    rotation.value = from + (to - from) * eased + back

    const delta = Math.abs(rotation.value - previousAngle)
    const dt = Math.max(now - previousAt, 12) / 1000
    spinSpeed.value = Math.min(1, (delta / dt) / 1400)
    previousAngle = rotation.value
    previousAt = now

    const pegCount = Math.max(pegs.value.length, 1)
    const step = 360 / pegCount
    const currentPeg = Math.floor((((rotation.value % 360) + 360) % 360) / step)
    if (currentPeg !== lastPeg) {
      lastPeg = currentPeg
      flapVelocity = -260
      // 指针扫过一颗钉子：一声「咔哒」，转速越快越实
      playWheelTick(spinSpeed.value)
      hapticTick()
    }
    flapVelocity += (-flapAngle.value * 900 - flapVelocity * 16) * dt
    flapAngle.value = Math.max(-22, Math.min(12, flapAngle.value + flapVelocity * dt))

    if (progress >= 1) {
      stopLoop()
      spinning.value = false
      spinSpeed.value = 0
      flapAngle.value = 0
      flapVelocity = 0
      finish()
    }
  }, 16)
}

function finish () {
  const result = pendingResult
  pendingResult = null
  if (!result) return
  resultCard.value = {
    label: result.label,
    detail: `${result.presetTitle} · ${result.optionCount} 个选项`,
    record: result.record
  }
  emit('result', result)
  const label = result.label || ''
  if (label) {
    uni.showToast({ title: `转到 ${label}`, icon: 'none' })
  }
}

function spin () {
  if (spinning.value) return
  const options = activePreset.value.options || []
  if (options.length < 2) {
    uni.showToast({ title: '至少需要 2 个选项', icon: 'none' })
    return
  }

  emit('audio')
  // 音效必须在这次点按里预热，不然后面每一声都会被浏览器/系统拦掉
  unlockGameSound()
  const result = spinWheel(props.scope, activePreset.value.id)
  const sector = sectors.value[result.index] || sectors.value[0]
  const current = rotation.value
  const base = ((current % 360) + 360) % 360
  // 指针在正上方，扇区中心转到 0° 时命中；再加一点随机偏移，避免每次都停在正中。
  const offset = (Math.random() - 0.5) * sector.angle * 0.62
  const turns = 6 + Math.floor(Math.random() * 3)
  const target = current + (turns * 360) + (((360 - sector.center - offset) - base) % 360 + 360) % 360

  spinning.value = true
  resultCard.value = {}
  pendingResult = result
  lastPeg = Math.floor((base % 360) / (360 / Math.max(pegs.value.length, 1)))
  runSpin(current, target, 4200 + Math.random() * 900)
}

/** 音效开关：关掉就一声不响，打开立刻给一声音，方便确认效果 */
function toggleWheelSound () {
  soundOn.value = toggleSoundOn()
  if (soundOn.value) {
    unlockGameSound()
    playWheelTick(0.7)
  }
}

/** 长按音效按钮：把音效引擎的状态弹出来，方便在真机上排查为什么没声 */
function showSoundDiagnostics () {
  uni.showModal({
    title: '音效状态',
    content: getSoundDiagnostics(),
    showCancel: false,
    confirmText: '知道了',
    confirmColor: '#ff5d8f'
  })
}

function openEditor () {
  if (!props.customEnabled) {
    uni.showModal({
      title: '登录后可以自定义',
      content: '游客可以用现成的转盘，登录后就能自己加转盘、改选项和权重。',
      confirmText: '去登录',
      confirmColor: '#ff5d8f',
      success: (res) => {
        if (res.confirm) uni.navigateTo({ url: '/pages/login/index?from=wheel' })
      }
    })
    return
  }
  draft.id = activePreset.value.id || ''
  draft.title = activePreset.value.title || ''
  draft.options = (activePreset.value.options || []).map((option) => ({
    key: `${option.id}-${Math.random().toString(16).slice(2, 6)}`,
    id: option.id,
    label: option.label,
    weight: String(Math.max(Number(option.weight || 1), 1))
  }))
  showEditor.value = true
}

function createPreset () {
  if (spinning.value) return
  if (!props.customEnabled) {
    openEditor()
    return
  }
  draft.id = ''
  draft.title = ''
  draft.options = [
    { key: `new-${Date.now()}-1`, label: '', weight: '1' },
    { key: `new-${Date.now()}-2`, label: '', weight: '1' }
  ]
  showEditor.value = true
}

function addOption () {
  draft.options.push({
    key: `opt-${Date.now()}-${Math.random().toString(16).slice(2, 6)}`,
    label: '',
    weight: '1'
  })
}

function removeOption (index) {
  draft.options.splice(index, 1)
}

function saveDraft () {
  const options = draft.options
    .map((option) => ({
      id: option.id || `wopt-${Date.now()}-${Math.random().toString(16).slice(2, 6)}`,
      label: String(option.label || '').trim(),
      weight: Math.max(Number(option.weight || 1), 1)
    }))
    .filter((option) => option.label)

  if (!String(draft.title || '').trim()) {
    uni.showToast({ title: '请先填写标题', icon: 'none' })
    return
  }
  if (options.length < 2) {
    uni.showToast({ title: '至少需要 2 个选项', icon: 'none' })
    return
  }

  const saved = saveWheelPreset(props.scope, {
    id: draft.id || `wheel-${Date.now()}`,
    title: draft.title,
    options
  })
  refreshPresets()
  activePresetId.value = saved.id
  showEditor.value = false
  resultCard.value = {}
  uni.showToast({ title: '已保存到本机', icon: 'none' })
}

function removePreset () {
  const preset = activePreset.value
  if (!preset.custom) return
  uni.showModal({
    title: '删除转盘',
    content: `确定删除「${preset.title}」吗？`,
    confirmColor: '#c2453f',
    success: (res) => {
      if (!res.confirm) return
      deleteWheelPreset(props.scope, preset.id)
      showEditor.value = false
      resultCard.value = {}
      refreshPresets()
    }
  })
}

watch(() => props.scope, refreshPresets, { immediate: true })

onBeforeUnmount(() => {
  stopLoop()
  spinning.value = false
  releaseGameSound()
})

onMounted(() => {
  // 进页面就把音频池建好、文件预加载，省得第一声要等
  prepareGameSound()
})

defineExpose({ spin, refreshPresets })
</script>

<style lang="scss" scoped>
.wheel-tool {
  padding: 24rpx;
  background: linear-gradient(180deg, #fffdf9 0%, #fbf4e8 100%);
}

.wheel-head {
  display: flex;
  /*
   * 必须能换行：右边两个胶囊按钮在窄屏上放不下时会横向溢出，
   * 整块头部被撑宽（实测 390 屏下溢出 110px，按钮看着就是错位的）。
   */
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 16rpx;
}

.wheel-copy {
  min-width: 0;
  flex: 1;
}

.wheel-kicker {
  display: block;
  color: rgba(178, 130, 62, 0.95);
  font-size: 17rpx;
  font-weight: 800;
  letter-spacing: 4rpx;
}

.wheel-title {
  display: block;
  margin-top: 8rpx;
  color: #241a15;
  font-size: 38rpx;
  font-weight: 900;
  letter-spacing: 3rpx;
  line-height: 1.1;
}

.wheel-subtitle {
  display: block;
  margin-top: 8rpx;
  color: rgba(88, 70, 58, 0.6);
  font-size: 21rpx;
  line-height: 1.5;
}

.wheel-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 8rpx;
  align-items: flex-start;
  flex: 0 1 auto;
  min-width: 0;
}

.wheel-chip,
.editor-add,
.editor-delete {
  display: inline-flex;
  align-items: center;
  gap: 4rpx;
  min-height: 52rpx;
  padding: 0 14rpx;
  border: 1rpx solid rgba(178, 130, 62, 0.22);
  border-radius: 999rpx;
  background: rgba(255, 255, 255, 0.72);
  color: rgba(120, 88, 52, 0.9);
  font-size: 19rpx;
  font-weight: 800;
}

/* 音效关掉时弱化一点，一眼能看出状态 */
.wheel-chip--off {
  border-color: rgba(178, 130, 62, 0.14);
  background: rgba(255, 255, 255, 0.46);
  color: rgba(120, 88, 52, 0.55);
}

/* ---------- 3D 舞台 ---------- */

.wheel-stage {
  position: relative;
  height: 640rpx;
  margin-top: 12rpx;
  perspective: 1500rpx;
  perspective-origin: 50% 34%;
}

.wheel-shadow {
  position: absolute;
  left: 50%;
  bottom: 34rpx;
  width: 540rpx;
  height: 96rpx;
  margin-left: -270rpx;
  border-radius: 50%;
  background: radial-gradient(50% 50% at 50% 50%, rgba(166, 68, 104, 0.34), rgba(166, 68, 104, 0) 72%);
  filter: blur(6rpx);
}

.wheel-tilt {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 620rpx;
  height: 620rpx;
  /* 上下左右都要各减一半（620/2=310），原来纵向只减了 300，盘子整体偏低 5px */
  margin: -310rpx 0 0 -310rpx;
  transform-style: preserve-3d;
  transform: rotateX(20deg);
}

.wheel-body {
  position: absolute;
  inset: 0;
  transform-style: preserve-3d;
}

.wheel-rim-layer {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  /* 换成连续的金色金属渐变，替掉原来重复的"糖果条纹"刻面 */
  background: linear-gradient(
    160deg,
    #FBF0D2 0%,
    #E9CE8E 26%,
    #C9A04A 52%,
    #F3E3B8 76%,
    #D8B464 100%
  );
  box-shadow: inset 0 0 0 2rpx rgba(255, 255, 255, 0.45);
}

.wheel-rim-layer:first-child {
  box-shadow:
    inset 0 0 0 3rpx rgba(255, 255, 255, 0.55),
    inset 0 -6rpx 14rpx rgba(120, 80, 20, 0.2),
    0 0 26rpx rgba(206, 152, 47, 0.32);
}

.wheel-disc {
  position: absolute;
  left: 30rpx;
  top: 30rpx;
  width: 560rpx;
  height: 560rpx;
  border-radius: 50%;
  /*
   * 每个扇形都是「整块正方形转一个角度」，旋转后的外接矩形会超出圆盘、
   * 甚至超出屏幕（实测窄屏右边多出 36px），页面就跟着能被左右拖。
   * 圆盘自己裁掉就好：标签和钉子都在圆内，视觉上没有任何损失。
   */
  overflow: hidden;
  transform-style: preserve-3d;
  will-change: transform;
}

.wheel-face {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  box-shadow: inset 0 0 0 4rpx rgba(255, 255, 255, 0.55);
}

.wheel-face-gloss {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  background:
    radial-gradient(120% 90% at 26% 12%, rgba(255, 255, 255, 0.62), rgba(255, 255, 255, 0) 46%),
    radial-gradient(100% 100% at 70% 96%, rgba(38, 16, 30, 0.22), rgba(38, 16, 30, 0) 58%);
  pointer-events: none;
}

.wheel-face-ring {
  position: absolute;
  inset: 22rpx;
  border-radius: 50%;
  border: 2rpx solid rgba(255, 255, 255, 0.32);
  pointer-events: none;
}

.wheel-speed {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  background: repeating-conic-gradient(
    from 0deg,
    rgba(255, 255, 255, 0.5) 0deg 2deg,
    rgba(255, 255, 255, 0) 2deg 16deg
  );
  pointer-events: none;
}

.wheel-sector {
  position: absolute;
  inset: 0;
}

.wheel-sector__rail {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 50%;
  height: 63%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.wheel-sector__label {
  color: #fff;
  font-weight: 900;
  letter-spacing: 1rpx;
  white-space: nowrap;
  transform: rotate(-90deg);
  text-shadow: 0 2rpx 6rpx rgba(60, 12, 32, 0.34);
}

.wheel-sector__label.is-flip {
  transform: rotate(90deg);
}

.wheel-sector__label.is-len-1 {
  font-size: 34rpx;
}

.wheel-sector__label.is-len-2 {
  font-size: 30rpx;
}

.wheel-sector__label.is-len-3 {
  font-size: 26rpx;
}

.wheel-sector__label.is-len-4 {
  font-size: 22rpx;
}

.wheel-peg {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 18rpx;
  height: 18rpx;
  margin: -9rpx 0 0 -9rpx;
  border-radius: 50%;
  background: radial-gradient(60% 60% at 34% 30%, #fffdf6, #d8c08a 62%, #8d7132 100%);
  box-shadow:
    0 3rpx 5rpx rgba(52, 26, 12, 0.42),
    inset 0 -2rpx 3rpx rgba(90, 62, 16, 0.5);
}

.wheel-hub {
  position: absolute;
  left: 50%;
  top: 50%;
  width: 236rpx;
  height: 236rpx;
  margin: -118rpx 0 0 -118rpx;
  transform: translateZ(20rpx);
  transform-style: preserve-3d;
}

.wheel-hub__ring {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  background: linear-gradient(
    160deg,
    #FCF2D8 0%,
    #EBD49B 38%,
    #C9A04A 68%,
    #F1E1B6 100%
  );
  box-shadow:
    0 8rpx 18rpx rgba(120, 62, 26, 0.3),
    inset 0 0 0 3rpx rgba(255, 255, 255, 0.55);
}

.wheel-hub__plate {
  position: absolute;
  inset: 16rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: radial-gradient(80% 80% at 32% 24%, #ffffff, #fff7ea 58%, #f5e2c2 100%);
  box-shadow: inset 0 4rpx 12rpx rgba(255, 255, 255, 0.9), inset 0 -6rpx 14rpx rgba(186, 140, 70, 0.2);
}

.wheel-hub__value {
  display: block;
  max-width: 160rpx;
  color: var(--color-text);
  font-size: 24rpx;
  font-weight: 900;
  text-align: center;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.wheel-hub__button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 56rpx;
  margin-top: 8rpx;
  padding: 0 20rpx;
  border: 1rpx solid rgba(232, 196, 132, 0.5);
  border-radius: 999rpx;
  background: linear-gradient(180deg, #c2453f 0%, #a8322c 100%);
  color: #f8ebd2;
  font-size: 22rpx;
  font-weight: 900;
  box-shadow: 0 10rpx 20rpx rgba(120, 60, 40, 0.26);
}

.wheel-hub__button[disabled] {
  opacity: 0.6;
}

.wheel-pointer {
  position: absolute;
  left: 50%;
  top: 26rpx;
  width: 46rpx;
  height: 150rpx;
  margin-left: -23rpx;
  transform-origin: 50% 8rpx;
  z-index: 6;
}

.wheel-pointer__cap {
  width: 46rpx;
  height: 46rpx;
  border-radius: 50%;
  background: radial-gradient(60% 60% at 32% 28%, #ffd9a0, #d8542c 62%, #9c2f16 100%);
  box-shadow: 0 6rpx 12rpx rgba(120, 40, 12, 0.4), inset 0 -3rpx 6rpx rgba(90, 24, 8, 0.5);
}

.wheel-pointer__blade {
  width: 0;
  height: 0;
  margin: -4rpx auto 0;
  border-left: 15rpx solid transparent;
  border-right: 15rpx solid transparent;
  border-top: 96rpx solid #e2542c;
  /* 先描一圈白边再投影，指针从盘面上"浮"起来，不再糊在一起 */
  filter:
    drop-shadow(0 0 3rpx rgba(255, 255, 255, 0.9))
    drop-shadow(0 8rpx 10rpx rgba(120, 40, 12, 0.3));
}

/* ---------- 信息区 ---------- */

.wheel-meta {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10rpx;
  margin-top: 12rpx;
}

.wheel-meta__item {
  padding: 14rpx 10rpx;
  border-radius: 16rpx;
  border: 1rpx solid rgba(178, 130, 62, 0.12);
  background: rgba(206, 158, 84, 0.07);
  text-align: center;
}

.wheel-meta__item text {
  display: block;
}

.wheel-meta__item text:first-child {
  color: #241a15;
  font-size: 28rpx;
  font-weight: 900;
}

.wheel-meta__item text:last-child {
  margin-top: 3rpx;
  color: rgba(88, 70, 58, 0.56);
  font-size: 18rpx;
}

.wheel-result {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16rpx;
  margin-top: 18rpx;
  padding: 20rpx;
  border: 1rpx solid rgba(178, 130, 62, 0.18);
  border-radius: 18rpx;
  background: rgba(206, 158, 84, 0.08);
}

.wheel-result__left {
  min-width: 0;
  flex: 1;
}

.wheel-result__label {
  display: block;
  color: rgba(88, 70, 58, 0.56);
  font-size: 19rpx;
  letter-spacing: 2rpx;
}

.wheel-result__value {
  display: block;
  margin-top: 6rpx;
  color: #241a15;
  font-size: 32rpx;
  font-weight: 900;
}

.wheel-result__detail {
  display: block;
  margin-top: 4rpx;
  color: rgba(88, 70, 58, 0.6);
  font-size: 20rpx;
}

.wheel-result__again {
  flex: 0 0 auto;
  min-height: 64rpx;
  padding: 0 24rpx;
  border-radius: 999rpx;
  background: linear-gradient(180deg, #c2453f 0%, #a8322c 100%);
  color: #f8ebd2;
  font-size: 22rpx;
  font-weight: 900;
}

.wheel-presets {
  margin-top: 18rpx;
  padding: 14rpx 16rpx 16rpx;
  border: 1rpx solid rgba(178, 130, 62, 0.16);
  border-radius: 20rpx;
  background: rgba(255, 255, 255, 0.72);
}

.wheel-presets__head {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 12rpx;
}

.wheel-presets__title {
  color: #241a15;
  font-size: 22rpx;
  font-weight: 900;
  letter-spacing: 2rpx;
}

.wheel-presets__count {
  color: rgba(88, 70, 58, 0.52);
  font-size: 18rpx;
}

.wheel-presets__list {
  display: flex;
  flex-wrap: wrap;
  gap: 10rpx;
  margin-top: 12rpx;
}

.preset-pill {
  display: inline-flex;
  align-items: center;
  gap: 6rpx;
  min-height: 56rpx;
  padding: 0 18rpx;
  border: 1rpx solid rgba(178, 130, 62, 0.2);
  border-radius: 999rpx;
  background: #fffdf9;
  color: rgba(88, 70, 58, 0.78);
  font-size: 21rpx;
  font-weight: 800;
}

.preset-pill.active {
  border-color: transparent;
  background: linear-gradient(180deg, #c2453f 0%, #a8322c 100%);
  color: #f8ebd2;
}

.preset-pill__badge {
  padding: 2rpx 8rpx;
  border-radius: 999rpx;
  background: rgba(224, 87, 126, 0.14);
  color: #b23a5c;
  font-size: 16rpx;
  font-weight: 900;
}

.preset-pill.active .preset-pill__badge {
  background: rgba(255, 255, 255, 0.22);
  color: #f8ebd2;
}

/* ---------- 编辑面板 ---------- */

.editor-mask {
  position: fixed;
  inset: 0;
  z-index: 1200;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  background: rgba(24, 18, 22, 0.42);
}

.editor-sheet {
  width: 100%;
  max-width: 900rpx;
  /* vh 在 iOS 上含地址栏，弹层会顶出去；dvh 跟着可视区变，旧浏览器退回 vh */
  max-height: 82vh;
  max-height: 82dvh;
  padding: 22rpx 22rpx calc(22rpx + env(safe-area-inset-bottom));
  border-radius: 20rpx 20rpx 0 0;
  overflow: auto;
}

.editor-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16rpx;
}

.editor-title {
  display: block;
  color: #241a15;
  font-size: 30rpx;
  font-weight: 900;
}

.editor-subtitle {
  display: block;
  margin-top: 6rpx;
  color: rgba(88, 70, 58, 0.6);
  font-size: 20rpx;
}

.editor-close {
  width: 56rpx;
  height: 56rpx;
  border-radius: 50%;
  background: rgba(178, 130, 62, 0.1);
  color: rgba(120, 88, 52, 0.9);
}

.editor-field {
  display: flex;
  flex-direction: column;
  gap: 8rpx;
  margin-top: 16rpx;
}

.editor-field text {
  color: rgba(88, 70, 58, 0.6);
  font-size: 20rpx;
  font-weight: 800;
  letter-spacing: 2rpx;
}

.editor-field input,
.editor-input,
.editor-weight {
  width: 100%;
  height: 78rpx;
  padding: 0 18rpx;
  border: 1rpx solid rgba(178, 130, 62, 0.22);
  border-radius: 16rpx;
  background: #fffdf9;
  color: #241a15;
  font-size: 24rpx;
}

.editor-row {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 120rpx 64rpx;
  gap: 10rpx;
  align-items: center;
  margin-top: 10rpx;
}

.editor-remove {
  width: 64rpx;
  height: 64rpx;
  border-radius: 16rpx;
  background: rgba(163, 39, 32, 0.08);
  color: #A3271F;
}

.editor-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10rpx;
  margin-top: 18rpx;
}

.editor-add {
  justify-content: center;
}

.editor-delete {
  background: rgba(88, 70, 58, 0.08);
  color: rgba(88, 70, 58, 0.7);
}

.editor-save {
  flex: 0 0 auto;
  min-width: 180rpx;
  min-height: 56rpx;
  padding: 0 26rpx;
  border: 1rpx solid rgba(232, 196, 132, 0.5);
  border-radius: 999rpx;
  background: linear-gradient(180deg, #c2453f 0%, #a8322c 100%);
  color: #f8ebd2;
  font-size: 23rpx;
  font-weight: 900;
  letter-spacing: 4rpx;
}
</style>
