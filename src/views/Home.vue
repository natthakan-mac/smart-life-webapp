<template>
  <div class="page-container">
    <div v-if="!loginState" class="login-wrapper">
      <el-card class="login-card">
        <div class="login-header">
          <h2>Welcome Home</h2>
          <p>Please sign in to control your devices.</p>
        </div>
        <el-form :model="loginForm" label-position="top" @submit.prevent="login()">
          <el-form-item label="Email address">
            <el-input v-model="loginForm.username" placeholder="your@email.com"></el-input>
          </el-form-item>
          <el-form-item label="Password">
            <el-input type="password" v-model="loginForm.password" placeholder="••••••••" show-password></el-input>
          </el-form-item>
          <div class="login-actions">
            <el-button type="primary" class="btn-login" @click="login()">Login</el-button>
          </div>
        </el-form>
      </el-card>
    </div>

    <div v-else class="dashboard-wrapper">
      <header class="dashboard-header">
        <div class="header-content">
          <div class="welcome-text">
            <h1>Your Devices</h1>
            <p>{{ pinnedDevices.length + generalDevices.length }} devices connected</p>
          </div>
          <div class="header-actions">
            <!-- Weather Card -->
            <div class="weather-card">
               <div class="weather-info">
                 <div class="date-time">
                   <div class="time">{{ currentTime }}</div>
                   <div class="date">{{ currentDate }}</div>
                 </div>
                 <div class="weather-detail" v-if="weatherData">
                   <i :class="getWeatherIcon(weatherData.weather_code)"></i>
                   <div class="weather-values">
                      <span>{{ weatherData.temperature_2m }}°C</span>
                      <span class="humidity"><i class="fa-solid fa-droplet"></i> {{ weatherData.relative_humidity_2m }}%</span>
                   </div>
                 </div>
               </div>
            </div>

            <el-button class="btn-secondary" @click="createVirtualDevice">
              <i class="fa-solid fa-plus"></i>
              <span>Add Custom Card</span>
            </el-button>

            <el-button class="btn-secondary" @click="showHiddenDevices = !showHiddenDevices">
              <i :class="['fa-solid', showHiddenDevices ? 'fa-eye-slash' : 'fa-eye']"></i>
              <span>{{ showHiddenDevices ? 'Hide Hidden' : 'Show Hidden' }}</span>
            </el-button>
            
            <el-button class="btn-secondary" @click="refreshDevices()">
              <i class="fa-solid fa-rotate"></i>
              <span>Refresh</span>
            </el-button>
            <el-button class="btn-logout" @click="logout()">
              <i class="fa-solid fa-right-from-bracket"></i>
              <span>Logout</span>
            </el-button>
          </div>
        </div>
      </header>
      
      <div class="section-container">
        <div class="section-header" @click="sectionConfig.custom.expanded = !sectionConfig.custom.expanded">
          <div class="header-left">
            <i class="fa-solid fa-chevron-right section-arrow" :class="{ 'expanded': sectionConfig.custom.expanded }"></i>
            <div v-if="sectionConfig.custom.editing" class="title-edit" @click.stop>
              <el-input 
                v-model="sectionConfig.custom.title" 
                size="small" 
                @blur="sectionConfig.custom.editing = false"
                @keyup.enter="sectionConfig.custom.editing = false"
                ref="customTitleInput"
              />
            </div>
            <h3 v-else class="section-title">{{ sectionConfig.custom.title }}</h3>
          </div>
          <div class="header-actions-sm">
             <el-button link size="small" @click.stop="startEditSection('custom')">
               <i class="fa-solid fa-pen"></i>
             </el-button>
          </div>
        </div>

        <draggable 
          v-if="sectionConfig.custom.expanded"
          v-model="pinnedDevices" 
          item-key="id" 
          handle=".drag-handle"
          ghost-class="ghost-card"
          group="devices"
          class="device-grid"
          @end="saveOrder"
        >
          <template #item="{ element: device }">
            <el-card 
              class="device" 
              :class="{ 'offline': device.data.online === false }"
              v-show="!device.customHidden || showHiddenDevices"
            >
              <div class="card-badges">
                <div v-if="getTemperature(device) && !isSensor(device)" class="temp-badge">
                  <i class="fa-solid fa-temperature-half"></i>
                  <span>{{ getTemperature(device) }}°C</span>
                </div>
                <div class="edit-btn" @click="openEditDialog(device)">
                  <i class="fa-solid fa-gear"></i>
                </div>
                <div class="hide-btn" @click.stop="quickHide(device)" title="Hide device">
                  <i class="fa-solid fa-eye-slash"></i>
                </div>
                <div class="drag-handle">
                  <i class="fa-solid fa-grip-vertical"></i>
                </div>
              </div>
              <div class="device-inner">
                <el-tooltip effect="light" :content="device.type" :offset="[-10, 0]" :visible-arrow="false">
                  <div class="device-icon-wrapper">
                    <div v-if="isFontIcon(device.customIcon || device.type)" class="device-icon-font">
                      <i :class="getIconClass(device.customIcon || device.type)"></i>
                    </div>
                    <el-avatar v-else :src="getIconUrl(device)" shape="circle" class="device-avatar">
                      <img src="device_icons/default.png"/>
                    </el-avatar>
                    <div v-if="device.data.online" class="online-indicator"></div>
                  </div>
                </el-tooltip>
                
                <div class="device-info">
                  <span class="device-name">{{ device.customName || device.name }}</span>
                  <span class="device-status">{{ device.data.online ? (device.data.state ? 'Active' : 'Standby') : 'Offline' }}</span>
                </div>

                <div class="device-control">
                  <template v-if="device.type === 'scene'">
                    <el-button circle size="large" class="trigger-btn" @click="triggerScene(device);">
                      <i class="fa-solid fa-play"></i>
                    </el-button>
                  </template>

                  <template v-else-if="device.customUi === 'door_lock' || device.customUi === 'ac_mode'">
                    <div class="custom-ui-grid" :class="device.customUi">
                      <el-button 
                        v-for="(btn, idx) in device.customButtons" 
                        :key="idx"
                        size="small" 
                        class="custom-btn"
                        @click="triggerCustomScene(btn.sceneId)"
                        :disabled="!btn.sceneId"
                      >
                        <div class="btn-content">
                          <i v-if="btn.icon" :class="btn.icon"></i>
                          <span>{{ btn.label || `Btn ${idx+1}` }}</span>
                        </div>
                      </el-button>
                    </div>
                  </template>

                  <template v-else-if="isSensor(device)">
                    <div class="sensor-display">
                      <span class="sensor-value">{{ getTemperature(device) }}</span>
                      <span class="sensor-unit">°C</span>
                    </div>
                  </template>
                  <template v-else>
                    <el-button circle size="large"
                      :class="['toggle-btn', device.data.state ? 'state-on' : 'state-off']"
                      :disabled="!device.data.online"
                      @click="toggleDevice(device);"
                    >
                      <i :class="['fa-solid', device.data.online ? 'fa-power-off' : 'fa-cloud-slash']"></i>
                    </el-button>
                  </template>
                </div class="device-control">
              </div>
            </el-card>
          </template>
        </draggable>
      </div>

      <div class="section-container">
        <div class="section-header" @click="sectionConfig.all.expanded = !sectionConfig.all.expanded">
          <div class="header-left">
             <i class="fa-solid fa-chevron-right section-arrow" :class="{ 'expanded': sectionConfig.all.expanded }"></i>
             <div v-if="sectionConfig.all.editing" class="title-edit" @click.stop>
               <el-input 
                 v-model="sectionConfig.all.title" 
                 size="small" 
                 @blur="sectionConfig.all.editing = false"
                 @keyup.enter="sectionConfig.all.editing = false"
                 ref="allTitleInput"
               />
             </div>
             <h3 v-else class="section-title">{{ sectionConfig.all.title }}</h3>
          </div>
           <div class="header-actions-sm">
             <el-button link size="small" @click.stop="startEditSection('all')">
               <i class="fa-solid fa-pen"></i>
             </el-button>
          </div>
        </div>

        <div v-show="sectionConfig.all.expanded">
          <div v-if="visibleGeneralDevices.length === 0" class="empty-state">
             <p v-if="visiblePinnedDevices.length > 0">All devices are in Custom section.</p>
             <p v-else>No devices found.</p>
          </div>
          <draggable 
            v-model="generalDevices" 
            item-key="id" 
            handle=".drag-handle"
            ghost-class="ghost-card"
            group="devices"
            class="device-grid"
            @end="saveOrder"
          >
          <template #item="{ element: device }">
            <el-card 
              class="device" 
              :class="{ 'offline': device.data.online === false }"
              v-show="!device.customHidden || showHiddenDevices"
            >
              <div class="card-badges">
                <div v-if="getTemperature(device) && !isSensor(device)" class="temp-badge">
                  <i class="fa-solid fa-temperature-half"></i>
                  <span>{{ getTemperature(device) }}°C</span>
                </div>
                <div class="edit-btn" @click="openEditDialog(device)">
                  <i class="fa-solid fa-gear"></i>
                </div>
                <div class="hide-btn" @click.stop="quickHide(device)" title="Hide device">
                  <i class="fa-solid fa-eye-slash"></i>
                </div>
                <div class="drag-handle">
                  <i class="fa-solid fa-grip-vertical"></i>
                </div>
              </div>
              <div class="device-inner">
                <el-tooltip effect="light" :content="device.type" :offset="[-10, 0]" :visible-arrow="false">
                  <div class="device-icon-wrapper">
                    <div v-if="isFontIcon(device.customIcon || device.type)" class="device-icon-font">
                      <i :class="getIconClass(device.customIcon || device.type)"></i>
                    </div>
                    <el-avatar v-else :src="getIconUrl(device)" shape="circle" class="device-avatar">
                      <img src="device_icons/default.png"/>
                    </el-avatar>
                    <div v-if="device.data.online" class="online-indicator"></div>
                  </div>
                </el-tooltip>
                
                <div class="device-info">
                  <span class="device-name">{{ device.customName || device.name }}</span>
                  <span class="device-status">{{ device.data.online ? (device.data.state ? 'Active' : 'Standby') : 'Offline' }}</span>
                </div>

                <div class="device-control">
                  <template v-if="device.type === 'scene'">
                    <el-button circle size="large" class="trigger-btn" @click="triggerScene(device);">
                      <i class="fa-solid fa-play"></i>
                    </el-button>
                  </template>

                  <template v-else-if="device.customUi === 'door_lock' || device.customUi === 'ac_mode'">
                    <div class="custom-ui-grid" :class="device.customUi">
                      <el-button 
                        v-for="(btn, idx) in device.customButtons" 
                        :key="idx"
                        size="small" 
                        class="custom-btn"
                        @click="triggerCustomScene(btn.sceneId)"
                        :disabled="!btn.sceneId"
                      >
                        <div class="btn-content">
                          <i v-if="btn.icon" :class="btn.icon"></i>
                          <span>{{ btn.label || `Btn ${idx+1}` }}</span>
                        </div>
                      </el-button>
                    </div>
                  </template>

                  <template v-else-if="isSensor(device)">
                    <div class="sensor-display">
                      <span class="sensor-value">{{ getTemperature(device) }}</span>
                      <span class="sensor-unit">°C</span>
                    </div>
                  </template>
                  <template v-else>
                    <el-button circle size="large"
                      :class="['toggle-btn', device.data.state ? 'state-on' : 'state-off']"
                      :disabled="!device.data.online"
                      @click="toggleDevice(device);"
                    >
                      <i :class="['fa-solid', device.data.online ? 'fa-power-off' : 'fa-cloud-slash']"></i>
                    </el-button>
                  </template>
                </div>
              </div>
            </el-card>
          </template>
        </draggable>
      </div>
      </div>

      <!-- Edit Dialog -->
      <el-dialog v-model="editDialogVisible" title="Customize Device" width="500px" border-radius="20px" custom-class="custom-dialog">
        <el-form label-position="top">
          <el-form-item label="Display Name">
            <el-input v-model="editForm.name" placeholder="e.g. Living Room Lamp"></el-input>
          </el-form-item>
          <el-form-item label="Icon">
            <el-input v-model="editForm.icon" placeholder="e.g. lightbulb, fa-brands fa-apple, or URL"></el-input>
            <div class="icon-help">
              <p>Tip: Use <code>fa-brands</code> for brand icons or <code>fa-regular</code> for outlined icons.</p>
              <a href="https://fontawesome.com/search?o=r&m=free" target="_blank" class="fa-link">
                Browse Font Awesome Free <i class="fa-solid fa-up-right-from-square"></i>
              </a>
            </div>
          </el-form-item>
          
          <el-form-item>
             <el-checkbox v-model="editForm.customSection" label="Show in Custom Section" border></el-checkbox>
             <el-checkbox v-model="editForm.hidden" label="Hide device" border class="mt-2"></el-checkbox>
          </el-form-item>

          <el-form-item v-if="editForm.isVirtual" label="Appearance Mode" class="mt-4">
             <el-select v-model="editForm.customUi" placeholder="Select Mode" style="width: 100%" @change="handleModeChange">
               <el-option label="Default" value="default" />
               <el-option label="Door Lock (3 Buttons)" value="door_lock" />
               <el-option label="Air Conditioner (5 Buttons)" value="ac_mode" />
             </el-select>
          </el-form-item>

          <div v-if="editForm.isVirtual && editForm.customUi && editForm.customUi !== 'default'" class="door-lock-config mt-4">
             <p class="config-title">Configure Buttons</p>
             <div v-for="(btn, i) in editForm.customButtons" :key="i" class="button-config">
               <span class="btn-label">Btn {{ i+1 }}</span>
               <el-input v-model="btn.label" placeholder="Label" size="small" style="width: 80px"></el-input>
               <el-input v-model="btn.icon" placeholder="Icon (fa-lock)" size="small" style="width: 100px"></el-input>
               <el-select v-model="btn.sceneId" placeholder="Select Scene" size="small" style="flex: 1">
                 <el-option v-for="s in availableScenes" :key="s.id" :label="s.name" :value="s.id" />
               </el-select>
             </div>
          </div>
        </el-form>
        <template #footer>
          <div class="dialog-footer" :class="{ 'has-delete': editForm.isVirtual }">
            <el-button v-if="editForm.isVirtual" type="danger" @click="deleteCustomCard" class="btn-delete">
               <i class="fa-solid fa-trash-can mr-1"></i> Delete
            </el-button>
            <div class="footer-actions">
              <el-button @click="editDialogVisible = false">Cancel</el-button>
              <el-button type="primary" @click="saveCustomization" class="btn-save">Save Changes</el-button>
            </div>
          </div>
        </template>
      </el-dialog>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Home'
}
</script>

<script setup="" >
/* eslint-disable no-unused-vars */
import { ref, reactive, computed, onMounted, watch } from 'vue'
import { ElMessage } from "element-plus"
import draggable from 'vuedraggable'

import tuya from '@/libs/tuya'

const homeAssistantClient = new tuya.HomeAssistantClient(
  JSON.parse(localStorage.getItem('session'))
)

const loginState = ref(false)
const pinnedDevices = ref([])
const generalDevices = ref([])

const loginForm = ref({ username: '', password: '' })

// Edit State
const editDialogVisible = ref(false)
const editingDevice = ref(null)
const editForm = reactive({
  name: '',
  icon: '',
  hidden: false,
  customSection: false,
  isVirtual: false,
  customUi: 'default',
  customButtons: []
})

const sectionConfig = reactive({
  custom: { title: 'Custom Cards', expanded: true, editing: false },
  all: { title: 'All Devices', expanded: false, editing: false }
})

watch(sectionConfig, (newVal) => {
  localStorage.setItem('dashboard_section_config', JSON.stringify(newVal))
}, { deep: true })

const showHiddenDevices = ref(false)
const currentTime = ref('')
const currentDate = ref('')
const weatherData = ref(null)

const visiblePinnedDevices = computed(() => {
  if (showHiddenDevices.value) return pinnedDevices.value
  return pinnedDevices.value.filter(d => !d.customHidden)
})

const visibleGeneralDevices = computed(() => {
  if (showHiddenDevices.value) return generalDevices.value
  return generalDevices.value.filter(d => !d.customHidden)
})

const availableScenes = computed(() => {
  // Combine all known devices to find scenes
  const all = [...pinnedDevices.value, ...generalDevices.value]
  return all.filter(d => d.type === 'scene')
})


// Initial sort by online status, but only if no manual order exists
const sortDevices = (list) => {
  const order = { true: 0, undefined: 1, false: 2 }
  return list.slice().sort((d1, d2) =>
    order[d1.data.online] > order[d2.data.online] ? 1 : -1
  )
}

onMounted(async () => {
  // TODO handle expired session
  loginState.value = !!homeAssistantClient.getSession()
  if (!loginState.value) {
    localStorage.clear()
  }
  const savedDevices = JSON.parse(localStorage.getItem('devices')) || []
  
  const savedSectionConfig = JSON.parse(localStorage.getItem('dashboard_section_config'))
  if (savedSectionConfig) {
    Object.assign(sectionConfig, savedSectionConfig)
    // Reset editing state just in case
    sectionConfig.custom.editing = false
    sectionConfig.all.editing = false
  }

  const pinned = savedDevices.filter(d => d.customSection)
  const general = savedDevices.filter(d => !d.customSection)
  pinnedDevices.value = pinned
  generalDevices.value = general
  updateTime()
  setInterval(updateTime, 60000)
  fetchWeather()
})

const updateTime = () => {
  const now = new Date()
  currentTime.value = now.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
  currentDate.value = now.toLocaleDateString([], { weekday: 'short', month: 'short', day: 'numeric' })
}

const fetchWeather = async () => {
  try {
    // 12150 Pathum Thani: Lat 13.933, Lon 100.75
    const res = await fetch('https://api.open-meteo.com/v1/forecast?latitude=13.933&longitude=100.75&current=temperature_2m,relative_humidity_2m,weather_code&timezone=auto')
    const data = await res.json()
    weatherData.value = data.current
  } catch (e) {
    console.warn('Weather fetch failed', e)
  }
}

const quickHide = (device) => {
  device.customHidden = true
  saveOrder()
  ElMessage.success('Device hidden')
}

const getWeatherIcon = (code) => {
  // Simple mapping for OpenMeteo WMO codes
  if (code === 0) return 'fa-solid fa-sun'
  if (code < 3) return 'fa-solid fa-cloud-sun'
  if (code < 45) return 'fa-solid fa-cloud'
  if (code < 51) return 'fa-solid fa-smog'
  if (code < 61) return 'fa-solid fa-cloud-rain'
  if (code < 80) return 'fa-solid fa-snowflake' // rough approximation
  return 'fa-solid fa-cloud'
}

const saveOrder = () => {
  const all = [...pinnedDevices.value, ...generalDevices.value]
  localStorage.setItem('devices', JSON.stringify(all))
}

const openEditDialog = (device) => {
  editingDevice.value = device
  editForm.name = device.customName || device.name
  editForm.icon = device.customIcon || device.type
  editForm.hidden = device.customHidden || false
  editForm.customSection = device.customSection || false
  editForm.hidden = device.customHidden || false
  editForm.customSection = device.customSection || false
  editForm.isVirtual = device.isVirtual || false
  
  editForm.customUi = device.customUi || 'default'
  if (['door_lock', 'ac_mode'].includes(editForm.customUi)) {
     editForm.customButtons = JSON.parse(JSON.stringify(device.customButtons || []))
     // Ensure correct length just in case
     const requiredLen = editForm.customUi === 'door_lock' ? 3 : 5
     while (editForm.customButtons.length < requiredLen) editForm.customButtons.push({ label: '', sceneId: '', icon: '' })
     editForm.customButtons = editForm.customButtons.slice(0, requiredLen)
  } else {
     editForm.customButtons = []
  }

  editDialogVisible.value = true
}

const handleModeChange = (val) => {
  if (val === 'default') {
    editForm.customButtons = []
  } else if (val === 'door_lock') {
    updateButtonCount(3)
  } else if (val === 'ac_mode') {
    updateButtonCount(5)
  }
}

const updateButtonCount = (count) => {
  const current = editForm.customButtons
  if (current.length < count) {
    for (let i = current.length; i < count; i++) {
        current.push({ label: '', sceneId: '', icon: '' })
    }
  } else if (current.length > count) {
    editForm.customButtons = current.slice(0, count)
  }
}

const saveCustomization = () => {
  if (editingDevice.value) {
    editingDevice.value.customName = editForm.name
    editingDevice.value.customIcon = editForm.icon
    editingDevice.value.customHidden = editForm.hidden
    
    editingDevice.value.customHidden = editForm.hidden
    
    // Save UI config
    if (editForm.customUi !== 'default') {
      editingDevice.value.customUi = editForm.customUi
      editingDevice.value.customButtons = JSON.parse(JSON.stringify(editForm.customButtons))
    } else {
      editingDevice.value.customUi = undefined
      editingDevice.value.customButtons = undefined
    }

    // Check if section changed
    const oldSection = editingDevice.value.customSection || false
    const newSection = editForm.customSection
    
    editingDevice.value.customSection = newSection
    editingDevice.value.isVirtual = editForm.isVirtual
    
    if (oldSection !== newSection) {
       // Move device
       if (newSection) {
         generalDevices.value = generalDevices.value.filter(d => d.id !== editingDevice.value.id)
         pinnedDevices.value.push(editingDevice.value)
       } else {
         pinnedDevices.value = pinnedDevices.value.filter(d => d.id !== editingDevice.value.id)
         generalDevices.value.push(editingDevice.value)
       }
    }

    saveOrder()
    editDialogVisible.value = false
    ElMessage.success('Changes saved!')
    editDialogVisible.value = false
    ElMessage.success('Changes saved!')
  }
}

const deleteCustomCard = () => {
  if (!confirm('Are you sure you want to delete this custom card?')) return
  
  // Remove from pinnedDevices
  pinnedDevices.value = pinnedDevices.value.filter(d => d.id !== editingDevice.value.id)
  // Just in case it somehow got to generalDevices
  generalDevices.value = generalDevices.value.filter(d => d.id !== editingDevice.value.id)
  
  saveOrder()
  editDialogVisible.value = false
  ElMessage.success('Card deleted')
}

const isFontIcon = (icon) => {
  if (!icon) return false
  return !icon.includes('.') && !icon.includes('/') && !icon.startsWith('http')
}

const getIconClass = (icon) => {
  if (!icon) return 'fa-solid fa-square-question'
  
  // If it's a full FA class string (e.g. "fa-brands fa-bluetooth"), use it as is
  if (icon.includes('fa-')) return icon
  
  // Smart mappings for standard Tuya/SmartLife types and names
  const mapping = {
    'switch': 'fa-solid fa-power-off',
    'scene': 'fa-solid fa-wand-magic-sparkles',
    'lightbulb': 'fa-solid fa-lightbulb',
    'outlet': 'fa-solid fa-plug',
    'router': 'fa-solid fa-wifi',
    'tv': 'fa-solid fa-tv',
    'air': 'fa-solid fa-wind',
    'ac_unit': 'fa-solid fa-snowflake',
    'thermostat': 'fa-solid fa-temperature-half',
    'door_front': 'fa-solid fa-door-open',
    'videocam': 'fa-solid fa-video',
    'sensor_window': 'fa-solid fa-window-restore',
    'settings_remote': 'fa-solid fa-remote-control',
    'wash': 'fa-solid fa-soap',
    'coffee_maker': 'fa-solid fa-coffee-pot',
    'curtains': 'fa-solid fa-scroll'
  }
  
  return mapping[icon] || `fa-solid fa-${icon}`
}

const getIconUrl = (device) => {
  const icon = device.customIcon || device.type
  if (icon.startsWith('http')) return icon
  return `device_icons/${icon}.png`
}

const getTemperature = (device) => {
  if (!device || !device.data) return null
  // Common Tuya/HomeAssistant fields for temperature
  const temp = device.data.temperature || 
               device.data.temp_current || 
               device.data.cur_temp || 
               device.data.current_temperature || 
               device.data.temp
  
  if (temp === undefined || temp === null) return null
  
  // Tuya often sends temperature as an integer (e.g., 225 for 22.5) or float
  // If it's very large, it might be multiplied by 10
  if (temp > 100 && temp < 1000) {
    return (temp / 10).toFixed(1)
  }
  
  return typeof temp === 'number' ? temp.toFixed(1) : temp
}

const isSensor = (device) => {
  if (!device) return false
  const type = device.type
  const sensorTypes = ['thermostat', 'sensor_temperature', 'humidity', 'sensor']
  if (sensorTypes.includes(type)) return true

  // Also treat as sensor if it has temperature data but NO boolean state (switch)
  // This prevents hiding the power button for smart plugs that report temperature.
  const hasTemp = getTemperature(device) !== null
  const hasSwitch = typeof device.data.state === 'boolean'
  
  return hasTemp && !hasSwitch && type !== 'ac_unit' 
}

const login = async () => {
  try {
    await homeAssistantClient.login(
      loginForm.value.username,
      loginForm.value.password
    )
    localStorage.setItem('session', JSON.stringify(homeAssistantClient.getSession()))
    loginState.value = true
    loginForm.value = { username: '', password: '' }
    refreshDevices()
  } catch (err) {
    ElMessage.error(`Oops, login error. (${err})`)
  }
}

const logout = () => {
  homeAssistantClient.dropSession()
  localStorage.clear()
  loginState.value = false
  loginForm.value = { username: '', password: '' }
  pinnedDevices.value = []
  devices.value = []
}

const createVirtualDevice = () => {
  const newDevice = {
    id: `virtual_${Date.now()}`,
    name: 'New Custom Card',
    type: 'virtual_card',
    isVirtual: true,
    data: { online: true, state: false },
    customSection: true, // Default to custom section
    customUi: 'default',
    customButtons: []
  }
  pinnedDevices.value.push(newDevice)
  saveOrder()
  openEditDialog(newDevice)
}

// Refs require nextTick to focus
const customTitleInput = ref(null)
const allTitleInput = ref(null)

import { nextTick } from 'vue'

const startEditSection = (section) => {
  sectionConfig[section].editing = true
  nextTick(() => {
    // Determine which ref to focus
    // Note: in v-for or setup refs might be arrays or objects
    // Since we used distinct refs:
    if (section === 'custom' && customTitleInput.value) {
       // El-input exposes input via ref? Check element-plus docs or try focus method
       customTitleInput.value.focus()
    } else if (section === 'all' && allTitleInput.value) {
       allTitleInput.value.focus()
    }
  })
}

const refreshDevices = async () => {
  // TODO handle expired session
  try {
    const discoveryResponse = await homeAssistantClient.deviceDiscovery()
    const newDevices = discoveryResponse.payload.devices || []
    
    // Merge with existing customizations & order
    const savedDevices = JSON.parse(localStorage.getItem('devices')) || []
    const merged = []
    
    // Keep existing ordered items that are still present, merging new data
    savedDevices.forEach(oldD => {
      if (oldD.isVirtual) {
        merged.push(oldD)
      } else {
        const freshD = newDevices.find(d => d.id === oldD.id)
        if (freshD) {
          // Merge fresh data with custom fields
          merged.push({
            ...freshD,
            customName: oldD.customName,
            customIcon: oldD.customIcon,
            customHidden: oldD.customHidden,
            customSection: oldD.customSection,
            customUi: oldD.customUi,
            customButtons: oldD.customButtons
          })
        }
      }
    })
    
    // Add new items that weren't in the ordered list
    newDevices.forEach(freshD => {
      if (!savedDevices.some(oldD => oldD.id === freshD.id)) {
        merged.push(freshD)
      }
    })
    
    const all = merged.length > 0 ? merged : sortDevices(newDevices)
    pinnedDevices.value = all.filter(d => d.customSection)
    generalDevices.value = all.filter(d => !d.customSection)
    saveOrder()
  } catch (err) {
    ElMessage.error(`Oops, device discovery error. (${err})`)
  }
}

const toggleDevice = async (device) => {
  // TODO handle expired session
  // TODO change icon to el-icon-loading
  try {
    const newState = !device.data.state
    await homeAssistantClient.deviceControl(device.id, 'turnOnOff', newState)
    device.data.state = newState
  } catch (err) {
    ElMessage.error(`Oops, device control error. (${err})`)
  }
}

const triggerScene = async (device) => {
  // TODO handle expired session
  // TODO change icon to el-icon-loading
  try {
    await homeAssistantClient.deviceControl(device.id, 'turnOnOff', true)
  } catch (err) {
    ElMessage.error(`Oops, device control error. (${err})`)
  }
}

const triggerCustomScene = async (sceneId) => {
  if (!sceneId) return
  try {
    await homeAssistantClient.deviceControl(sceneId, 'turnOnOff', true)
    ElMessage.success('Scene triggered')
  } catch (err) {
    ElMessage.error(`Oops, scene error. (${err})`)
  }
}
</script>

<style scoped>
.page-container {
  padding: 40px 20px;
  max-width: 1400px;
  margin: 0 auto;
}

/* Login Card Styles */
.login-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 70vh;
}

.login-card {
  width: 100%;
  max-width: 420px;
  border-radius: 32px;
  border: 1px solid rgba(255, 255, 255, 0.4);
  background: var(--card-bg);
  backdrop-filter: blur(20px);
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.05);
  padding: 20px;
}

.login-header {
  margin-bottom: 32px;
}

.login-header h2 {
  font-size: 28px;
  font-weight: 700;
  margin-bottom: 8px;
  color: var(--text-main);
}

.login-header p {
  color: var(--text-muted);
  font-size: 15px;
}

.login-actions {
  margin-top: 24px;
}

.btn-login {
  width: 100%;
  height: 50px;
  font-size: 16px;
  font-weight: 600;
  background-color: var(--primary-color) !important;
  border: none;
  border-radius: 16px;
  transition: all 0.3s ease;
}

.btn-login:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 15px rgba(136, 195, 169, 0.3);
}

/* Dashboard Header */
.dashboard-header {
  margin-bottom: 60px;
  text-align: left;
}

.header-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 20px;
}

.welcome-text h1 {
  font-size: 36px;
  font-weight: 800;
  margin: 0 0 4px 0;
  background: linear-gradient(135deg, var(--text-main), #88929b);
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.welcome-text p {
  color: var(--text-muted);
  margin: 0;
  font-size: 16px;
}

.header-actions {
  display: flex;
  gap: 12px;
}

.btn-secondary, .btn-logout {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  height: 44px;
  border-radius: 14px;
  font-weight: 500;
  border: 1px solid rgba(0,0,0,0.05);
  background: white !important;
  color: var(--text-main) !important;
  transition: all 0.2s ease;
}

.btn-secondary:hover, .btn-logout:hover {
  background: #f8f9fa !important;
  transform: translateY(-1px);
}

.btn-logout:hover {
  color: #ef4444 !important;
}

/* Device Grid */
#devices {
  /* Legacy ID, removed from usage */
}

.el-card.device {
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: var(--card-bg);
  backdrop-filter: blur(12px);
  border-radius: 28px;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  overflow: visible;
}

.device-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 28px;
}

.section-container {
  margin-bottom: 40px;
}

.section-title {
  text-align: left;
  font-size: 20px;
  font-weight: 700;
  color: var(--text-main);
  margin-bottom: 20px;
  padding-left: 8px;
  border-left: 4px solid var(--primary-color);
}

@media (max-width: 1200px) { .device-grid { grid-template-columns: repeat(3, 1fr); } }
@media (max-width: 900px) { .device-grid { grid-template-columns: repeat(2, 1fr); } }
@media (max-width: 600px) { .device-grid { grid-template-columns: repeat(1, 1fr); } }

.el-card.device:hover {
  transform: translateY(-8px) scale(1.02);
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.04);
  background: rgba(255, 255, 255, 0.9);
}

.el-card.device.offline {
  filter: opacity(0.6) grayscale(0.5);
}

.el-card.device :deep(.el-card__body) {
  padding: 28px;
  position: relative;
}

.card-badges {
  position: absolute;
  top: 12px;
  left: 12px;
  right: 12px;
  display: flex;
  justify-content: space-between;
  z-index: 10;
  opacity: 0;
  transition: opacity 0.2s;
}

.device:hover .card-badges {
  opacity: 1;
}

.edit-btn, .hide-btn, .drag-handle, .temp-badge {
  cursor: pointer;
  color: #cad4e0;
  transition: color 0.2s;
}


.temp-badge {
  cursor: default;
  display: flex;
  align-items: center;
  gap: 4px;
  background: rgba(255, 255, 255, 0.5);
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 13px;
  font-weight: 700;
  color: var(--text-main);
  backdrop-filter: blur(4px);
  border: 1px solid rgba(255, 255, 255, 0.4);
}

.temp-badge i {
  font-size: 12px;
  color: var(--accent-color);
}

.edit-btn:hover, .hide-btn:hover, .drag-handle:hover {
  color: var(--text-muted);
}

.hide-btn:hover {
  color: #ef4444;
}

.drag-handle {
  cursor: grab;
}

.drag-handle:active {
  cursor: grabbing;
}

.ghost-card {
  opacity: 0.3;
  background: var(--primary-color) !important;
  border: 2px dashed var(--primary-color) !important;
}

.device-inner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
}

.device-icon-wrapper {
  position: relative;
  margin-bottom: 4px;
}

.device-icon-font {
  width: 72px;
  height: 72px;
  background: #f0f2f5;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s ease;
}

.device-icon-font i {
  font-size: 36px;
  color: var(--text-main);
}

.device:hover .device-icon-font {
  background: white;
  transform: rotate(5deg);
}

.device-avatar {
  width: 72px;
  height: 72px;
  background: #f0f2f5;
  padding: 12px;
  transition: all 0.3s ease;
}

.device:hover .device-avatar {
  background: white;
  transform: rotate(5deg);
}

.online-indicator {
  position: absolute;
  bottom: 2px;
  right: 2px;
  width: 14px;
  height: 14px;
  background: var(--primary-color);
  border: 3px solid white;
  border-radius: 50%;
  box-shadow: 0 0 10px rgba(136, 195, 169, 0.5);
}

.device-info {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.device-name {
  font-weight: 700;
  font-size: 17px;
  color: var(--text-main);
  word-break: break-word;
}

.device-status {
  font-size: 13px;
  color: var(--text-muted);
  font-weight: 500;
}

.device-control {
  margin-top: 4px;
}

.weather-card {
  background: white;
  border-radius: 14px;
  padding: 0 16px;
  display: flex;
  align-items: center;
  border: 1px solid rgba(0,0,0,0.05);
  margin-right: 8px;
  height: 44px;
}

.weather-info {
  display: flex;
  align-items: center;
  gap: 16px;
}

.date-time {
  text-align: right;
  line-height: 1.2;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.time {
  font-weight: 700;
  font-size: 15px;
  color: var(--text-main);
}

.date {
  font-size: 10px;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.weather-detail {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 16px;
  font-weight: 600;
  color: var(--primary-color);
  padding-left: 12px;
  border-left: 1px solid #f0f0f0;
  height: 24px;
}

.weather-values {
  display: flex;
  flex-direction: column;
  line-height: 1;
  align-items: flex-start;
  gap: 2px;
}

.humidity {
  font-size: 11px;
  color: var(--text-muted);
  font-weight: 500;
}

.humidity {
  font-size: 11px;
  color: var(--text-muted);
  font-weight: 500;
}

.door-lock-grid {
  display: flex;
  flex-direction: column;
  gap: 6px;
  width: 100%;
}

.lock-btn {
  margin: 0 !important;
  width: 100%;
  border-radius: 8px;
}

.door-lock-config {
  background: #f8f9fa;
  padding: 12px;
  border-radius: 12px;
}

.config-title {
  margin: 0 0 8px 0;
  font-size: 12px;
  font-weight: 700;
  color: var(--text-muted);
  text-transform: uppercase;
}

.button-config {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.btn-label {
  font-size: 12px;
  width: 40px;
  color: var(--text-muted);
  flex-shrink: 0;
}

.button-config .el-input {
  flex-shrink: 0;
}


/* Edit Dialog Styles */
.custom-dialog :deep(.el-dialog) {
  border-radius: 24px;
  padding: 10px;
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.dialog-footer.has-delete {
  justify-content: space-between;
}

.footer-actions {
  display: flex;
  gap: 8px;
}


.icon-help {
  margin-top: 12px;
  text-align: left;
  font-size: 12px;
  color: var(--text-muted);
}

.icon-help p {
  margin: 0 0 4px 0;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  padding: 8px 4px;
  border-radius: 8px;
  margin-bottom: 8px;
  user-select: none;
}
.section-header:hover {
  background-color: rgba(0,0,0,0.03);
}
.header-left {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}
.section-arrow {
  transition: transform 0.2s;
  font-size: 12px;
  color: var(--text-muted);
}
.section-arrow.expanded {
  transform: rotate(90deg);
}
.header-actions-sm {
  opacity: 0;
  transition: opacity 0.2s;
}
.section-header:hover .header-actions-sm {
  opacity: 1;
}
.title-edit {
  flex: 1;
  max-width: 200px;
}

.fa-link {
  font-size: 13px;
  color: var(--primary-color) !important;
  text-decoration: none;
  font-weight: 600;
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.fa-link:hover {
  text-decoration: underline;
}

.mt-12 { margin-top: 12px; }

.btn-save {
  border-radius: 12px;
  padding: 10px 24px;
}

/* Buttons */
.toggle-btn, .trigger-btn {
  width: 56px;
  height: 56px;
  border: none !important;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1) !important;
}

.toggle-btn.state-on {
  background: var(--primary-color) !important;
  color: white !important;
  box-shadow: 0 8px 20px rgba(136, 195, 169, 0.4) !important;
}

.toggle-btn.state-off {
  background: #f0f2f5 !important;
  color: #94a3b8 !important;
}

.trigger-btn {
  background: var(--accent-color) !important;
  color: white !important;
  box-shadow: 0 8px 20px rgba(243, 182, 100, 0.4) !important;
}

.toggle-btn:hover:not(:disabled), .trigger-btn:hover {
  transform: scale(1.1);
}

.toggle-btn i, .trigger-btn i {
  font-size: 28px;
}

.sensor-display {
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: white;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
  border: 1px solid rgba(0,0,0,0.05);
}

.sensor-value {
  font-size: 18px;
  font-weight: 800;
  color: var(--text-main);
  line-height: 1;
}

.sensor-unit {
  font-size: 10px;
  color: var(--text-muted);
  font-weight: 600;
}
.custom-ui-grid {
  display: grid;
  gap: 6px;
  width: 100%;
}

.custom-ui-grid.door_lock {
  grid-template-columns: repeat(3, 1fr);
}

.custom-ui-grid.ac_mode {
  grid-template-columns: 1fr 1fr;
}
/* Make the last item in AC mode span full width if odd number */
.custom-ui-grid.ac_mode .custom-btn:last-child:nth-child(odd) {
  grid-column: span 2;
}

.custom-btn {
  margin: 0 !important;
  width: 100%;
  border-radius: 8px;
  height: auto !important;
  padding: 8px !important;
}

.custom-btn .btn-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  line-height: 1.2;
  width: 100%;
}

.custom-btn i {
  font-size: 20px;
  margin-bottom: 2px;
}


.door-lock-config {
  background: #f8f9fa;
  padding: 12px;
  border-radius: 12px;
}

.config-title {
  margin: 0 0 8px 0;
  display: block;
  font-size: 12px;
  font-weight: 700;
  color: var(--text-muted);
  text-transform: uppercase;
}

.button-config {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.btn-label {
  font-size: 12px;
  width: 40px;
  color: var(--text-muted);
}
</style>
