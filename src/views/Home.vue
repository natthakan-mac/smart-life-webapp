<template>
  <div class="page-container">
    <div v-if="!loginState" class="login-wrapper">
      <el-card class="login-card">
        <div class="login-header">
          <img src="https://play-lh.googleusercontent.com/M29pkEabzdIihXxY6d9N1i-hX1ZO8Trt2UTni65CG9NcOZaCTwEusFO3PEBWM4cWdcs=w240-h480-rw" alt="Smart Life Logo" class="login-logo">
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
            <p>{{ sections.reduce((acc, s) => acc + s.devices.length, 0) }} devices connected</p>
          </div>
          <div class="header-actions">
            <!-- Weather Card -->
            <div class="weather-pill">
               <div class="weather-brief">
                 <div class="vibrant-time">{{ currentTime }}</div>
                 <div class="vibrant-date">{{ currentDate }}</div>
               </div>
               <div class="vibrant-weather" v-if="weatherData">
                 <div class="weather-main">
                   <i :class="getWeatherIcon(weatherData.weather_code)"></i>
                   <span class="vibrant-temp">{{ Math.round(weatherData.temperature_2m) }}°</span>
                 </div>
                 <div class="weather-meta">
                    <span class="vibrant-humidity"><i class="fa-solid fa-droplet"></i> {{ weatherData.relative_humidity_2m }}%</span>
                 </div>
               </div>
            </div>

            <el-button class="btn-primary-sm" @click="addSection">
              <i class="fa-solid fa-folder-plus"></i>
              <span>Add Section</span>
            </el-button>

            <el-button class="btn-secondary-sm" @click="createVirtualDevice">
              <i class="fa-solid fa-plus"></i>
              <span>Add Card</span>
            </el-button>

            <el-button class="btn-logout-sm" @click="logout()">
              <i class="fa-solid fa-right-from-bracket"></i>
            </el-button>
          </div>
        </div>
      </header>
      
      <el-tabs v-model="activeTab" class="dashboard-tabs">
        <el-tab-pane label="Dashboard" name="dashboard">
          <div class="tab-content">
            <draggable 
              v-model="sections" 
              item-key="id" 
              handle=".section-drag-handle"
              ghost-class="ghost-section"
              class="sections-wrapper"
              @end="saveOrder"
            >
              <template #item="{ element: section }">
                <div class="section-container">
                  <div class="section-header" @mouseenter="section.hover = true" @mouseleave="section.hover = false">
                    <div class="header-left" @click="section.expanded = !section.expanded">
                      <i class="fa-solid fa-chevron-right section-arrow" :class="{ 'expanded': section.expanded }"></i>
                      <div v-if="section.editing" class="title-edit" @click.stop>
                        <el-input 
                          v-model="section.title" 
                          size="small" 
                          @blur="section.editing = false"
                          @keyup.enter="section.editing = false"
                          :ref="el => { if (el) sectionRefs[section.id] = el }"
                        />
                      </div>
                      <h3 v-else class="section-title">{{ section.title }}</h3>
                    </div>
                    
                    <div class="header-actions-sm" v-show="section.hover || section.editing">
                      <el-button link size="small" @click.stop="editSection(section)">
                        <i class="fa-solid fa-pen"></i>
                      </el-button>
                      <el-button v-if="section.id !== 'default'" link size="small" @click.stop="deleteSection(section)">
                        <i class="fa-solid fa-trash"></i>
                      </el-button>
                      <div class="section-drag-handle">
                        <i class="fa-solid fa-grip-lines"></i>
                      </div>
                    </div>
                  </div>

                  <div v-show="section.expanded">
                    <div v-if="section.devices.length === 0" class="empty-state">
                       <i class="fa-solid fa-box-open mb-2" style="font-size: 24px; opacity: 0.2"></i>
                       <p>Drag devices here.</p>
                    </div>
                    <draggable 
                      v-model="section.devices" 
                      item-key="id" 
                      handle=".drag-handle"
                      ghost-class="ghost-card"
                      group="devices"
                      class="device-grid"
                      @end="saveOrder"
                    >
                      <template #item="{ element: device }">
                        <DeviceCard 
                          v-if="!device.customHidden"
                          :device="device"
                          @edit="openEditDialog"
                          @hide="quickHide"
                          @delete="handleDelete"
                          @toggle="toggleDevice"
                          @trigger-scene="triggerScene"
                        />
                      </template>
                    </draggable>
                  </div>
                </div>
              </template>
            </draggable>
        </div>
      </el-tab-pane>

      <el-tab-pane label="Hidden" name="hidden">
        <div class="tab-content">
          <div class="empty-state" v-if="hiddenDevices.length === 0">
            <i class="fa-solid fa-eye-slash mb-2" style="font-size: 32px; opacity: 0.3"></i>
            <p>No hidden devices.</p>
          </div>
          <div class="device-grid" v-else>
            <DeviceCard 
              v-for="device in hiddenDevices"
              :key="device.id"
              :device="device"
              :show-hidden="true"
              @edit="openEditDialog"
              @hide="toggleHide(device)"
              @delete="handleDelete"
              @toggle="toggleDevice"
              @trigger-scene="triggerScene"
            />
          </div>
        </div>
      </el-tab-pane>
    </el-tabs>

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
          
          <el-form-item label="Card Size">
            <el-radio-group v-model="editForm.size" size="small">
              <el-radio-button label="full">Full Size</el-radio-button>
              <el-radio-button label="half">Half Size</el-radio-button>
            </el-radio-group>
          </el-form-item>

          <el-form-item label="UI Mode">
            <el-radio-group v-model="editForm.uiMode" size="small">
              <el-radio-button label="default">Default</el-radio-button>
              <el-radio-button label="door_lock">Door Lock</el-radio-button>
              <el-radio-button label="ac_unit">AC Unit</el-radio-button>
            </el-radio-group>
          </el-form-item>
          
          <el-form-item>
             <el-checkbox v-model="editForm.hidden" label="Hide device" border class="mt-2"></el-checkbox>
          </el-form-item>
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



<script setup>
/* eslint-disable no-unused-vars */
import { ref, reactive, computed, onMounted, watch, nextTick } from 'vue'
import { ElMessage, ElMessageBox } from "element-plus"
import draggable from 'vuedraggable'
import tuya from '@/libs/tuya'
import DeviceCard from '@/components/DeviceCard.vue'

const homeAssistantClient = new tuya.HomeAssistantClient(
  JSON.parse(localStorage.getItem('session'))
)

const loginState = ref(false)
const sections = ref([
  { id: 'default', title: 'All Devices', expanded: true, editing: false, devices: [], hover: false }
])

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
  size: 'full',
  uiMode: 'default'
})

const sectionRefs = reactive({})
const activeTab = ref('dashboard')
const currentTime = ref('')
const currentDate = ref('')
const weatherData = ref(null)

const allAvailableDevices = computed(() => {
  const all = sections.value.flatMap(s => s.devices)
  return all.filter(d => !d.isVirtual)
})

const hiddenDevices = computed(() => {
  const all = sections.value.flatMap(s => s.devices)
  return all.filter(d => d.customHidden)
})


// Initial sort by online status, but only if no manual order exists
const sortDevices = (list) => {
  const order = { true: 0, undefined: 1, false: 2 }
  return list.slice().sort((d1, d2) =>
    order[d1.data.online] > order[d2.data.online] ? 1 : -1
  )
}

onMounted(async () => {
  loginState.value = !!homeAssistantClient.getSession()
  if (!loginState.value) {
    localStorage.clear()
  }
  
  const savedSections = JSON.parse(localStorage.getItem('dashboard_sections'))
  if (savedSections) {
    sections.value = savedSections.map(s => ({ ...s, editing: false, hover: false }))
  } else {
    // Legacy migration or fresh start
    const savedDevices = JSON.parse(localStorage.getItem('devices')) || []
    if (savedDevices.length > 0) {
      const pinned = savedDevices.filter(d => d.customSection)
      const general = savedDevices.filter(d => !d.customSection)
      sections.value = [
        { id: 'custom', title: 'Custom Cards', expanded: true, editing: false, devices: pinned, hover: false },
        { id: 'default', title: 'All Devices', expanded: true, editing: false, devices: general, hover: false }
      ]
    }
  }

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
  ElMessage.success('Device moved to Hidden tab')
}

const toggleHide = (device) => {
  device.customHidden = !device.customHidden
  saveOrder()
}

const handleDelete = (device) => {
  if (device.isVirtual) {
    editingDevice.value = device
    deleteCustomCard()
  } else {
    if (confirm(`Remove ${device.customName || device.name} from dashboard?`)) {
      sections.value.forEach(s => {
        s.devices = s.devices.filter(d => d.id !== device.id)
      })
      saveOrder()
      ElMessage.success('Removed from dashboard')
    }
  }
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
  localStorage.setItem('dashboard_sections', JSON.stringify(sections.value))
}

const openEditDialog = (device) => {
  editingDevice.value = device
  editForm.name = device.customName || device.name
  editForm.icon = device.customIcon || device.type
  editForm.hidden = device.customHidden || false
  editForm.customSection = device.customSection || false
  editForm.isVirtual = device.isVirtual || false
  editForm.size = device.size || 'full'
  editForm.uiMode = device.uiMode || 'default'
  
  editDialogVisible.value = true
}



const saveCustomization = () => {
  if (editingDevice.value) {
    editingDevice.value.customName = editForm.name
    editingDevice.value.customIcon = editForm.icon
    editingDevice.value.customHidden = editForm.hidden
    editingDevice.value.isVirtual = editForm.isVirtual
    editingDevice.value.size = editForm.size
    editingDevice.value.uiMode = editForm.uiMode
    
    saveOrder()
    editDialogVisible.value = false
    ElMessage.success('Changes saved!')
  }
}

const deleteCustomCard = () => {
  if (!confirm('Are you sure you want to delete this custom card?')) return
  
  sections.value.forEach(s => {
    s.devices = s.devices.filter(d => d.id !== editingDevice.value.id)
  })
  
  saveOrder()
  editDialogVisible.value = false
  ElMessage.success('Card deleted')
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
  ElMessageBox.confirm(
    'Are you sure you want to log out?',
    'Logout Confirmation',
    {
      confirmButtonText: 'Logout',
      cancelButtonText: 'Cancel',
      type: 'warning',
      roundButton: true,
      customClass: 'premium-confirm'
    }
  ).then(() => {
    homeAssistantClient.dropSession()
    localStorage.clear()
    loginState.value = false
    loginForm.value = { username: '', password: '' }
    sections.value = [
      { id: 'default', title: 'All Devices', expanded: true, editing: false, devices: [], hover: false }
    ]
    ElMessage.success('Logged out successfully')
  }).catch(() => {
    // User cancelled
  })
}

const createVirtualDevice = () => {
  const newDevice = {
    id: `virtual_${Date.now()}`,
    name: 'New Custom Card',
    type: 'virtual_card',
    isVirtual: true,
    size: 'full',
    uiMode: 'default',
    data: { online: true, state: false, temperature: 24 },
    customSection: true,
    customUi: 'default',
    customButtons: []
  }
  // Add to the first section or default
  const targetSection = sections.value[0] || { devices: [] }
  targetSection.devices.push(newDevice)
  saveOrder()
  openEditDialog(newDevice)
}

const addSection = () => {
  const id = `section_${Date.now()}`
  sections.value.push({
    id,
    title: 'New Section',
    expanded: true,
    editing: true,
    devices: [],
    hover: false
  })
  nextTick(() => {
    if (sectionRefs[id]) sectionRefs[id].focus()
  })
}

const editSection = (section) => {
  section.editing = true
  nextTick(() => {
    if (sectionRefs[section.id]) sectionRefs[section.id].focus()
  })
}

const deleteSection = (section) => {
  if (section.id === 'default') return
  if (confirm(`Are you sure you want to delete "${section.title}"? Devices will be moved to default section.`)) {
    const defaultSection = sections.value.find(s => s.id === 'default')
    if (defaultSection) {
      defaultSection.devices.push(...section.devices)
    }
    sections.value = sections.value.filter(s => s.id !== section.id)
    saveOrder()
  }
}

// Section management methods already implemented above: addSection, editSection, deleteSection

const refreshDevices = async () => {
  try {
    const discoveryResponse = await homeAssistantClient.deviceDiscovery()
    const newDevices = discoveryResponse.payload.devices || []
    
    // Flatten current devices to find existing ones
    const currentDevices = sections.value.flatMap(s => s.devices)
    
    // Add new devices to default section if they don't exist anywhere
    newDevices.forEach(freshD => {
      const exists = currentDevices.some(d => d.id === freshD.id)
      if (!exists) {
        const defaultSection = sections.value.find(s => s.id === 'default') || sections.value[0]
        defaultSection.devices.push({
          ...freshD,
          customName: '',
          customIcon: freshD.type,
          customHidden: false,
          customSection: false,
          customUi: 'default',
          customButtons: []
        })
      } else {
        // Sync fresh data for existing devices
        sections.value.forEach(s => {
          s.devices = s.devices.map(d => {
            if (d.id === freshD.id) {
              return { ...freshD, ...d, data: freshD.data } // Keep custom fields, update data/online
            }
            return d
          })
        })
      }
    })
    
    saveOrder()
  } catch (err) {
    ElMessage.error(`Oops, device discovery error. (${err})`)
  }
}

const toggleDevice = async (device) => {
  // TODO handle expired session
  // TODO change icon to el-icon-loading
  try {
    if (device._tempAdjust !== undefined) {
      await homeAssistantClient.deviceControl(device.id, 'setTemperature', device._tempAdjust)
      return
    }
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

const findDeviceById = (id) => {
  return sections.value.flatMap(s => s.devices).find(d => d.id === id)
}

const toggleGroupDevice = async (id) => {
  const d = findDeviceById(id)
  if (d) {
    await toggleDevice(d)
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
  text-align: center;
}

.login-logo {
  width: 80px;
  height: 80px;
  object-fit: contain;
  border-radius: 18px;
  margin-bottom: 20px;
  box-shadow: 0 8px 16px rgba(0,0,0,0.05);
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
  align-items: center; /* Ensure vertical alignment */
  flex-wrap: wrap;
  gap: 24px;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 16px;
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

.btn-primary-sm, .btn-secondary-sm, .btn-logout-sm {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  height: 40px;
  padding: 0 16px;
  border-radius: 12px;
  font-weight: 600;
  transition: all 0.2s ease;
  border: none;
}

.btn-primary-sm {
  background: var(--primary-color) !important;
  color: white !important;
  box-shadow: 0 4px 12px rgba(136, 195, 169, 0.3);
}

.btn-secondary-sm {
  background: rgba(255, 255, 255, 0.6) !important;
  backdrop-filter: blur(8px);
  color: var(--text-main) !important;
  border: 1px solid rgba(0,0,0,0.05) !important;
}

.btn-primary-sm:hover, .btn-secondary-sm:hover {
  transform: translateY(-1px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.05);
}

.btn-logout-sm {
  background: #fee2e2 !important;
  color: #ef4444 !important;
  width: 40px;
  padding: 0;
}

.btn-logout-sm:hover {
  background: #fecaca !important;
  transform: scale(1.05);
}

/* Premium Weather Pill */
.weather-pill {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.6);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.5);
  padding: 6px 20px;
  border-radius: 24px;
  gap: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.03);
  margin-right: 8px;
}

.weather-brief {
  border-right: 1px solid rgba(0, 0, 0, 0.05);
  padding-right: 16px;
  text-align: right;
}

.vibrant-time {
  font-weight: 800;
  font-size: 16px;
  color: var(--text-main);
  line-height: 1.1;
}

.vibrant-date {
  font-size: 10px;
  font-weight: 600;
  color: #94a3b8;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.vibrant-weather {
  display: flex;
  align-items: center;
  gap: 12px;
}

.weather-main {
  display: flex;
  align-items: center;
  gap: 6px;
}

.weather-main i {
  font-size: 20px;
  color: #f59e0b; /* Sunny yellow */
}

.vibrant-temp {
  font-size: 18px;
  font-weight: 800;
  color: var(--text-main);
}

.weather-meta {
  display: flex;
  flex-direction: column;
  line-height: 1;
}

.vibrant-humidity {
  font-size: 11px;
  color: #64748b;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 3px;
}

.vibrant-humidity i {
  color: #3b82f6;
}

/* Tabs Styling */
.dashboard-tabs :deep(.el-tabs__header) {
  margin-bottom: 32px;
  border: none;
}

.dashboard-tabs :deep(.el-tabs__nav-wrap::after) {
  display: none;
}

.dashboard-tabs :deep(.el-tabs__item) {
  font-size: 16px;
  font-weight: 700;
  color: #94a3b8;
  padding: 0 24px;
  height: 48px;
  line-height: 48px;
  transition: all 0.3s;
}

.dashboard-tabs :deep(.el-tabs__item.is-active) {
  color: var(--primary-color);
}

.dashboard-tabs :deep(.el-tabs__active-bar) {
  background-color: var(--primary-color);
  height: 3px;
  border-radius: 3px;
}

.tab-content {
  animation: fadeIn 0.4s ease-out;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.el-card.device {
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: var(--card-bg);
  backdrop-filter: blur(12px);
  border-radius: 20px;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  overflow: visible;
}

.device-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
  grid-auto-rows: min-content;
  gap: 16px;
  align-items: start;
}

/* Base Device Layout overriden by specific sizes */
:deep(.device) {
  grid-column: span 2;
}

:deep(.device.size-half) {
  grid-column: span 1;
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

@media (max-width: 900px) {
  .device-grid {
    grid-template-columns: repeat(auto-fill, minmax(130px, 1fr));
  }
}

@media (max-width: 600px) { 
  .device-grid { 
    grid-template-columns: 1fr; 
  }
  :deep(.device), :deep(.device.size-half) {
    grid-column: span 1;
  }
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
  padding: 12px 16px;
  border-radius: 12px;
  margin-bottom: 8px;
  user-select: none;
  background: rgba(0,0,0,0.02);
  transition: all 0.2s;
}

.section-header:hover {
  background: rgba(0,0,0,0.04);
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
  display: flex;
  align-items: center;
  gap: 8px;
  opacity: 0;
  transition: opacity 0.2s;
}

.section-header:hover .header-actions-sm {
  opacity: 1;
}

.section-drag-handle {
  cursor: grab;
  color: #94a3b8;
  padding: 4px 8px;
  border-radius: 6px;
  transition: background 0.2s;
}

.section-drag-handle:hover {
  background: rgba(0,0,0,0.05);
  color: var(--text-main);
}

.section-drag-handle:active {
  cursor: grabbing;
}

.ghost-section {
  opacity: 0.5;
  background: #f1f5f9;
  border-radius: 20px;
}

.sections-wrapper {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.title-edit {
  flex: 1;
  max-width: 250px;
}

.empty-state {
  padding: 40px;
  text-align: center;
  color: #94a3b8;
  background: rgba(0,0,0,0.01);
  border-radius: 20px;
  border: 2px dashed rgba(0,0,0,0.05);
  margin: 12px 0;
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

</style>
