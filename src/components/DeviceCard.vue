<template>
  <el-card 
    class="device" 
    :class="{ 
      'offline': device.data.online === false,
      'is-busy': busy,
      'size-half': device.size === 'half',
      'mode-lock': device.uiMode === 'door_lock',
      'mode-ac': device.uiMode === 'ac_unit'
    }"
    v-show="!device.customHidden || showHidden"
    @mouseenter="checkMarquee"
  >
    <!-- Hover Tools Pill -->
    <div class="card-tools">
      <div class="tool-pill">
        <div class="edit-btn" @click.stop="$emit('edit', device)" title="Edit">
          <i class="fa-solid fa-gear"></i>
        </div>
        <div class="hide-btn" @click.stop="$emit('hide', device)" title="Hide">
          <i class="fa-solid fa-eye-slash"></i>
        </div>
        <div class="delete-btn" @click.stop="$emit('delete', device)" title="Delete">
          <i class="fa-solid fa-trash-can"></i>
        </div>
        <div class="drag-handle" title="Drag">
          <i class="fa-solid fa-grip-vertical"></i>
        </div>
      </div>
    </div>

    <!-- Temperature Badge -->
    <div v-if="temperature && !isSensor && device.uiMode !== 'ac_unit' && device.size !== 'half'" class="temp-tag">
      <i class="fa-solid fa-temperature-half"></i>
      <span>{{ temperature }}°C</span>
    </div>

    <div class="device-inner">
      <!-- Specialized UI: AC Unit -->
      <template v-if="device.uiMode === 'ac_unit'">
        <div class="ac-layout">
          <div class="ac-info">
             <span class="device-name marquee-target">{{ device.customName || device.name }}</span>
             <div class="ac-temp-display">
                <span class="ac-val">{{ device.data.temperature || 24 }}</span>
                <span class="ac-unit">°C</span>
             </div>
          </div>
          <div class="ac-controls">
             <el-button circle size="small" class="ac-btn" @click.stop="onToggle">
                <i :class="['fa-solid', device.data.state ? 'fa-power-off' : 'fa-power-off']"></i>
             </el-button>
             <div class="ac-adjust">
                <el-button circle size="small" @click.stop="adjustTemp(-1)"><i class="fa-solid fa-minus"></i></el-button>
                <el-button circle size="small" @click.stop="adjustTemp(1)"><i class="fa-solid fa-plus"></i></el-button>
             </div>
          </div>
        </div>
      </template>

      <!-- Specialized UI: Door Lock -->
      <template v-else-if="device.uiMode === 'door_lock'">
        <div class="lock-layout">
           <div class="lock-visual" @click.stop="onToggle">
              <i :class="['fa-solid', device.data.state ? 'fa-lock-open' : 'fa-lock']"></i>
           </div>
           <div class="lock-info">
              <span class="device-name">{{ device.customName || device.name }}</span>
              <span class="lock-status">{{ device.data.state ? 'Unlocked' : 'Locked' }}</span>
           </div>
        </div>
      </template>

      <!-- Default/Half Size Layout -->
      <template v-else>
        <!-- Icon Section -->
        <div class="device-icon-wrapper" v-if="device.size !== 'half' || !device.data.state">
          <div v-if="isFontIcon(device.customIcon || device.type)" class="device-icon-font">
            <i :class="getIconClass(device.customIcon || device.type)"></i>
          </div>
          <el-avatar v-else :src="getIconUrl(device)" shape="circle" class="device-avatar">
            <img src="device_icons/default.png"/>
          </el-avatar>
          <div v-if="device.data.online" class="online-indicator"></div>
        </div>
        
        <!-- Info Section -->
        <div class="device-info">
          <div class="name-viewport" ref="nameContainer">
            <div class="name-scroll" :class="{ 'marquee': shouldMarquee }" :style="marqueeStyle">
              <span class="device-name">{{ device.customName || device.name }}</span>
              <span v-if="shouldMarquee" class="device-name marquee-spacer">{{ device.customName || device.name }}</span>
            </div>
          </div>
          <div class="status-row" v-if="device.size !== 'half'">
            <span class="device-status">{{ statusText }}</span>
          </div>
        </div>

        <!-- Controls Section -->
        <div class="device-control">
          <template v-if="device.type === 'scene'">
            <el-button circle size="small" class="trigger-btn" @click.stop="onTriggerScene">
              <i class="fa-solid fa-play"></i>
            </el-button>
          </template>
          <template v-else-if="isSensor">
            <div class="sensor-display">
              <span class="sensor-value">{{ temperature }}</span>
              <span class="sensor-unit">°C</span>
            </div>
          </template>
          <template v-else>
            <el-button circle size="small"
              :class="['toggle-btn', device.data.state ? 'state-on' : 'state-off']"
              :disabled="!device.data.online"
              @click.stop="onToggle"
            >
              <i :class="['fa-solid', device.data.online ? 'fa-power-off' : 'fa-cloud-slash']"></i>
            </el-button>
          </template>
        </div>
      </template>
    </div>
  </el-card>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  device: { type: Object, required: true },
  showHidden: { type: Boolean, default: false }
})

const emit = defineEmits([
  'edit', 'hide', 'delete', 'toggle', 'trigger-scene'
])

const busy = ref(false)
const nameContainer = ref(null)
const shouldMarquee = ref(false)
const marqueeDist = ref(0)

const marqueeStyle = computed(() => {
  return {
    '--dist': `-${marqueeDist.value}px`,
    '--dur': `${Math.max(3, marqueeDist.value / 30)}s`
  }
})

const checkMarquee = () => {
  if (nameContainer.value) {
    const el = nameContainer.value.querySelector('.device-name')
    if (el) {
      const parentWidth = nameContainer.value.clientWidth
      const textWidth = el.scrollWidth
      if (textWidth > parentWidth) {
        shouldMarquee.value = true
        marqueeDist.value = textWidth + 24 // 24px gap
      } else {
        shouldMarquee.value = false
      }
    }
  }
}

onMounted(() => {
  setTimeout(checkMarquee, 100) // Small delay for layout calculation
  window.addEventListener('resize', checkMarquee)
})

onUnmounted(() => {
  window.removeEventListener('resize', checkMarquee)
})

// Logic & Helpers
const isSensor = computed(() => {
  const d = props.device
  const type = d.type
  const sensorTypes = ['thermostat', 'sensor_temperature', 'humidity', 'sensor']
  if (sensorTypes.includes(type)) return true
  const hasTemp = temperature.value !== null
  const hasSwitch = typeof d.data.state === 'boolean'
  return hasTemp && !hasSwitch && type !== 'ac_unit'
})

const temperature = computed(() => {
  if (!props.device || !props.device.data) return null
  const d = props.device.data
  const temp = d.temperature || d.temp_current || d.cur_temp || d.current_temperature || d.temp
  if (temp === undefined || temp === null) return null
  if (temp > 100 && temp < 1000) return (temp / 10).toFixed(1)
  return typeof temp === 'number' ? temp.toFixed(1) : temp
})

const statusText = computed(() => {
  if (props.device.data.online === false) return 'Offline'
  if (isSensor.value) return 'Online'
  return props.device.data.state ? 'Active' : 'Standby'
})

const isFontIcon = (icon) => {
  if (!icon) return false
  return !icon.includes('.') && !icon.includes('/') && !icon.startsWith('http')
}

const getIconClass = (icon) => {
  if (!icon) return 'fa-solid fa-square-question'
  if (icon.includes('fa-')) return icon
  const mapping = {
    'virtual_card': 'fa-solid fa-layer-group',
    'switch': 'fa-solid fa-power-off',
    'scene': 'fa-solid fa-wand-magic-sparkles',
    'lightbulb': 'fa-solid fa-lightbulb',
    'outlet': 'fa-solid fa-plug',
    'fan': 'fa-solid fa-fan'
  }
  return mapping[icon] || `fa-solid fa-${icon}`
}

const getIconUrl = (device) => {
  const icon = device.customIcon || device.type
  return icon.startsWith('http') ? icon : `device_icons/${icon}.png`
}

// Event Handlers (with feedback)
const onToggle = async () => {
  busy.value = true
  try { await emit('toggle', props.device) } 
  finally { setTimeout(() => { busy.value = false }, 500) }
}

const onTriggerScene = async () => {
  busy.value = true
  try { await emit('trigger-scene', props.device) }
  finally { setTimeout(() => { busy.value = false }, 500) }
}

const adjustTemp = async (delta) => {
  if (busy.value) return
  busy.value = true
  try {
    const current = props.device.data.temperature || 24
    const next = current + delta
    await emit('toggle', { ...props.device, _tempAdjust: next }) // Pass temp update via event
    props.device.data.temperature = next
  } finally {
    setTimeout(() => { busy.value = false }, 300)
  }
}
</script>

<style scoped>
.device {
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: var(--card-bg);
  backdrop-filter: blur(12px);
  border-radius: 20px;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  overflow: hidden;
  height: 64px;
  position: relative;
  min-width: 180px; /* Ensure a stable width in grid */
}

.device.is-busy {
  opacity: 0.8;
}

.device:hover {
  transform: translateY(-2px) scale(1.01);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.04);
  background: rgba(255, 255, 255, 0.95);
}

.device.offline {
  filter: opacity(0.6) grayscale(0.5);
}

.device :deep(.el-card__body) {
  padding: 12px 14px;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  align-items: center;
}

/* Hover Tools Pill - Refined to not overlap content */
.card-tools {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(4px);
  opacity: 0;
  transition: opacity 0.2s ease;
  z-index: 20;
  border-radius: 20px; /* Match card radius */
  pointer-events: none;
}

.device:hover .card-tools {
  opacity: 1;
  pointer-events: auto;
}

.tool-pill {
  display: flex;
  gap: 16px;
  background: white;
  padding: 8px 16px;
  border-radius: 20px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  border: 1px solid rgba(0,0,0,0.05);
}

.edit-btn, .hide-btn, .delete-btn, .drag-handle {
  cursor: pointer;
  color: #64748b;
  font-size: 16px;
  transition: all 0.2s;
}

.edit-btn:hover { color: var(--primary-color); }
.hide-btn:hover { color: #f59e0b; }
.delete-btn:hover { color: #ef4444; }
.drag-handle { cursor: grab; }
.drag-handle:active { cursor: grabbing; }

/* Integrated Temp Tag */
.temp-tag {
  position: absolute;
  bottom: 4px;
  left: 54px; /* Align after icon */
  display: flex;
  align-items: center;
  gap: 2px;
  pointer-events: none;
  font-size: 9px;
  font-weight: 700;
  color: var(--primary-color);
  opacity: 0.8;
}

.temp-tag i {
  font-size: 8px;
}

.device-inner {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  width: 100%;
}

.device-icon-wrapper {
  position: relative;
  flex-shrink: 0;
}

.device-icon-font {
  width: 40px;
  height: 40px;
  background: #f1f5f9;
  border-radius: 12px;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s ease;
}

.device-icon-font i {
  font-size: 20px;
  color: #475569;
}

.device-avatar {
  width: 40px;
  height: 40px;
  background: #f1f5f9;
  padding: 6px;
  border-radius: 12px;
}

.online-indicator {
  position: absolute;
  bottom: -1px;
  right: -1px;
  width: 9px;
  height: 9px;
  background: #10b981;
  border: 2px solid white;
  border-radius: 50%;
}

.device-info {
  display: flex;
  flex-direction: column;
  gap: 0px;
  flex: 1;
  text-align: left;
  overflow: hidden;
  min-width: 0;
}

.name-viewport {
  overflow: hidden;
  white-space: nowrap;
  width: 100%;
}

.name-scroll {
  display: flex;
  width: max-content;
}

.device-name {
  font-weight: 700;
  font-size: 14px;
  color: var(--text-main);
  padding-right: 24px; /* Gap for marquee spacer */
}

.name-scroll:not(.marquee) .device-name {
  padding-right: 0;
  width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
}

.device:hover .name-scroll.marquee {
  animation: marquee-precise var(--dur) linear infinite;
}

@keyframes marquee-precise {
  0% { transform: translateX(0); }
  10% { transform: translateX(0); }
  90% { transform: translateX(var(--dist)); }
  100% { transform: translateX(var(--dist)); }
}

.device-status {
  font-size: 11px;
  color: #94a3b8;
  font-weight: 500;
}

/* Half Size Card */
.device.size-half {
  min-width: 120px;
  flex: 0 0 calc(25% - 12px);
}

.device.size-half :deep(.el-card__body) {
  padding: 8px 10px;
}

.device.size-half .device-inner {
  gap: 8px;
}

.device.size-half .device-name {
  font-size: 13px;
}

/* Door Lock Mode */
.device.mode-lock {
  height: 80px;
}

.lock-layout {
  display: flex;
  align-items: center;
  gap: 16px;
  width: 100%;
}

.lock-visual {
  width: 48px;
  height: 48px;
  background: #f1f5f9;
  border-radius: 16px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  transition: all 0.3s;
}

.lock-visual:hover {
  background: #e2e8f0;
  transform: scale(1.05);
}

.lock-visual i {
  font-size: 20px;
  color: #64748b;
}

.device.mode-lock.state-on .lock-visual {
  background: linear-gradient(135deg, #e0f2fe 0%, #bae6fd 100%);
  box-shadow: 0 4px 12px rgba(14, 165, 233, 0.2);
}

.device.mode-lock.state-on .lock-visual i {
  color: #0369a1;
  transform: scale(1.1);
}

.lock-info {
  display: flex;
  flex-direction: column;
}

.lock-status {
  font-size: 12px;
  font-weight: 600;
  color: #94a3b8;
}

.device.mode-lock.state-on .lock-status {
  color: #0ea5e9;
}

/* AC Mode */
.device.mode-ac {
  height: 100px;
}

.ac-layout {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.ac-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.ac-temp-display {
  display: flex;
  align-items: baseline;
  gap: 2px;
}

.ac-val {
  font-size: 28px;
  font-weight: 800;
  color: var(--text-main);
  line-height: 1;
}

.ac-unit {
  font-size: 14px;
  font-weight: 600;
  color: #94a3b8;
}

.ac-controls {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 8px;
}

.ac-btn {
  background: #f1f5f9 !important;
  color: #64748b !important;
  border: none !important;
}

.device.mode-ac.state-on .ac-btn {
  background: linear-gradient(135deg, #88c3a9 0%, #6ab091 100%) !important;
  color: white !important;
  box-shadow: 0 4px 12px rgba(136, 195, 169, 0.3);
}

.ac-adjust {
  display: flex;
  gap: 4px;
}

.ac-adjust .el-button {
  background: #f1f5f9 !important;
  border: none !important;
  color: #64748b !important;
}

.ac-adjust .el-button:hover {
  background: #e2e8f0 !important;
}

.device-control { 
  flex-shrink: 0;
}

.toggle-btn, .trigger-btn {
  width: 36px;
  height: 36px;
  border: none !important;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  transition: all 0.3s ease !important;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0 !important;
}

.toggle-btn.state-on {
  background: var(--primary-color) !important;
  color: white !important;
}

.toggle-btn.state-off {
  background: #f1f5f9 !important;
  color: #64748b !important;
}

.trigger-btn {
  background: #fde68a !important;
  color: #92400e !important;
}

.toggle-btn i, .trigger-btn i { font-size: 16px; }

.sensor-display {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  background: #f1f5f9;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.sensor-value {
  font-size: 13px;
  font-weight: 800;
  color: #334155;
  line-height: 1;
}

.sensor-unit {
  font-size: 8px;
  color: #64748b;
  font-weight: 600;
}
</style>
