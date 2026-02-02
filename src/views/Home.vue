<template>
  <div class="page-container">
    <div v-if="!loginState" class="login-wrapper">
      <el-card class="login-card">
        <div class="login-header">
          <h2>Welcome Home</h2>
          <p>Please sign in to control your devices.</p>
        </div>
        <el-form :model="loginForm" label-position="top">
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
            <p>{{ devices.length }} devices connected</p>
          </div>
          <div class="header-actions">
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

      <draggable 
        v-model="devices" 
        item-key="id" 
        handle=".drag-handle"
        ghost-class="ghost-card"
        id="devices"
        @end="saveOrder"
      >
        <template #item="{ element: device }">
          <el-card class="device" :class="{ 'offline': device.data.online === false }">
            <div class="card-badges">
              <div v-if="getTemperature(device)" class="temp-badge">
                <i class="fa-solid fa-temperature-half"></i>
                <span>{{ getTemperature(device) }}°C</span>
              </div>
              <div class="edit-btn" @click="openEditDialog(device)">
                <i class="fa-solid fa-gear"></i>
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

      <!-- Edit Dialog -->
      <el-dialog v-model="editDialogVisible" title="Customize Device" width="400px" border-radius="20px" custom-class="custom-dialog">
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
        </el-form>
        <template #footer>
          <div class="dialog-footer">
            <el-button @click="editDialogVisible = false">Cancel</el-button>
            <el-button type="primary" @click="saveCustomization" class="btn-save">Save Changes</el-button>
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
const devices = ref([])

const loginForm = ref({ username: '', password: '' })

// Edit State
const editDialogVisible = ref(false)
const editingDevice = ref(null)
const editForm = reactive({
  name: '',
  icon: ''
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
  devices.value = savedDevices
})

const saveOrder = () => {
  localStorage.setItem('devices', JSON.stringify(devices.value))
}

const openEditDialog = (device) => {
  editingDevice.value = device
  editForm.name = device.customName || device.name
  editForm.icon = device.customIcon || device.type
  editDialogVisible.value = true
}

const saveCustomization = () => {
  if (editingDevice.value) {
    editingDevice.value.customName = editForm.name
    editingDevice.value.customIcon = editForm.icon
    saveOrder()
    editDialogVisible.value = false
    ElMessage.success('Changes saved!')
  }
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
  devices.value = []
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
      const freshD = newDevices.find(d => d.id === oldD.id)
      if (freshD) {
        // Merge fresh data with custom fields
        merged.push({
          ...freshD,
          customName: oldD.customName,
          customIcon: oldD.customIcon
        })
      }
    })
    
    // Add new items that weren't in the ordered list
    newDevices.forEach(freshD => {
      if (!savedDevices.some(oldD => oldD.id === freshD.id)) {
        merged.push(freshD)
      }
    })
    
    devices.value = merged.length > 0 ? merged : sortDevices(newDevices)
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
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 28px;
}

@media (max-width: 1200px) { #devices { grid-template-columns: repeat(3, 1fr); } }
@media (max-width: 900px) { #devices { grid-template-columns: repeat(2, 1fr); } }
@media (max-width: 600px) { #devices { grid-template-columns: repeat(1, 1fr); } }

.el-card.device {
  border: 1px solid rgba(255, 255, 255, 0.5);
  background: var(--card-bg);
  backdrop-filter: blur(12px);
  border-radius: 28px;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  overflow: visible;
}

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

.edit-btn, .drag-handle, .temp-badge {
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

.edit-btn:hover, .drag-handle:hover {
  color: var(--text-muted);
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

/* Edit Dialog Styles */
.custom-dialog :deep(.el-dialog) {
  border-radius: 24px;
  padding: 10px;
}


.icon-help {
  margin-top: 12px;
  text-align: left;
}

.icon-help p {
  font-size: 13px;
  color: var(--text-muted);
  margin: 0 0 4px 0;
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
</style>
