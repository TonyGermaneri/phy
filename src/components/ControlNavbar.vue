<template>
  <v-app-bar
    color="rgba(0,0,0,0.7)"
    class="control-navbar"
    density="compact"
    floating
    :style="{ 
      'backdrop-filter': 'blur(10px)', 
      position: 'fixed', 
      top: '16px', 
      left: '16px', 
      right: '16px', 
      width: 'auto', 
      'border-radius': '12px',
      transition: 'opacity 0.8s ease-in-out',
      opacity: showNavbar ? 1 : 0,
      'pointer-events': showNavbar ? 'auto' : 'none'
    }"
    @mouseenter="onMouseEnterControls"
    @mouseleave="onMouseLeaveControls"
  >
    <template #prepend>
      <v-btn
        icon="mdi-dna"
        variant="text"
        color="white"
      />
    </template>

    <v-app-bar-title class="text-white font-weight-bold">
      Physarum Simulation
    </v-app-bar-title>

    <v-spacer />

    <!-- Control buttons -->
    <v-btn
      :icon="isPlaying ? 'mdi-pause' : 'mdi-play'"
      variant="text"
      color="white"
      @click="togglePlay"
    />

    <v-btn
      icon="mdi-restart"
      variant="text"
      color="white"
      @click="resetSimulation"
    />

    <v-btn
      icon="mdi-cog"
      variant="text"
      color="white"
      @click="showControls = !showControls"
    />
  </v-app-bar>

  <!-- Expandable controls panel -->
  <v-expand-transition>
    <v-card
      v-if="showControls"
      class="controls-panel"
      color="rgba(0,0,0,0.8)"
      :style="{ 
        'backdrop-filter': 'blur(15px)', 
        position: 'fixed', 
        top: '80px', 
        left: '16px', 
        right: '16px', 
        'border-radius': '12px', 
        'z-index': '1001',
        transition: 'opacity 0.8s ease-in-out',
        opacity: showNavbar ? 1 : 0,
        'pointer-events': showNavbar ? 'auto' : 'none'
      }"
      @mouseenter="onMouseEnterControls"
      @mouseleave="onMouseLeaveControls"
    >
      <v-card-text>
        <v-row dense>
          <!-- Particle Count -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Particles: {{ formatNumber(params.numParticles) }}</label>
              <v-slider
                v-model="params.numParticles"
                :min="5000"
                :max="100000"
                :step="5000"
                color="blue"
                track-color="grey-darken-1"
                thumb-color="blue"
                hide-details
              />
            </div>
          </v-col>

          <!-- Sensor Distance -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Sensor Distance: {{ params.sensorDistance.toFixed(1) }}</label>
              <v-slider
                v-model="params.sensorDistance"
                :min="1"
                :max="20"
                :step="0.1"
                color="green"
                track-color="grey-darken-1"
                thumb-color="green"
                hide-details
              />
            </div>
          </v-col>

          <!-- Sensor Angle -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Sensor Angle: {{ (params.sensorAngle * 180 / Math.PI).toFixed(0) }}°</label>
              <v-slider
                v-model="params.sensorAngle"
                :min="0"
                :max="Math.PI"
                :step="0.01"
                color="orange"
                track-color="grey-darken-1"
                thumb-color="orange"
                hide-details
              />
            </div>
          </v-col>

          <!-- Rotation Angle -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Rotation Angle: {{ (params.rotationAngle * 180 / Math.PI).toFixed(0) }}°</label>
              <v-slider
                v-model="params.rotationAngle"
                :min="0"
                :max="Math.PI / 2"
                :step="0.01"
                color="purple"
                track-color="grey-darken-1"
                thumb-color="purple"
                hide-details
              />
            </div>
          </v-col>

          <!-- Move Distance -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Move Distance: {{ params.moveDistance.toFixed(1) }}</label>
              <v-slider
                v-model="params.moveDistance"
                :min="0.1"
                :max="5"
                :step="0.1"
                color="red"
                track-color="grey-darken-1"
                thumb-color="red"
                hide-details
              />
            </div>
          </v-col>

          <!-- Decay Factor -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Decay Factor: {{ (params.decayFactor * 100).toFixed(0) }}%</label>
              <v-slider
                v-model="params.decayFactor"
                :min="0.1"
                :max="1"
                :step="0.01"
                color="cyan"
                track-color="grey-darken-1"
                thumb-color="cyan"
                hide-details
              />
            </div>
          </v-col>

          <!-- Deposit Amount -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Deposit Amount: {{ params.depositAmount.toFixed(1) }}</label>
              <v-slider
                v-model="params.depositAmount"
                :min="0.1"
                :max="20"
                :step="0.1"
                color="yellow"
                track-color="grey-darken-1"
                thumb-color="yellow"
                hide-details
              />
            </div>
          </v-col>

          <!-- Resolution -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Resolution: {{ (params.resolution * 100).toFixed(0) }}%</label>
              <v-slider
                v-model="params.resolution"
                :min="0.1"
                :max="1.0"
                :step="0.05"
                color="pink"
                track-color="grey-darken-1"
                thumb-color="pink"
                hide-details
              />
            </div>
          </v-col>
        </v-row>

        <!-- Presets -->
        <v-divider class="my-4" />

        <div class="presets-section">
          <h4 class="text-white mb-3">Presets:</h4>
          <v-row dense>
            <v-col cols="6" sm="4" md="3" lg="2" v-for="preset in presets" :key="preset.name">
              <v-btn
                @click="loadPreset(preset)"
                :color="preset.color"
                variant="tonal"
                size="small"
                block
              >
                {{ preset.name }}
              </v-btn>
            </v-col>
          </v-row>
        </div>
      </v-card-text>
    </v-card>
  </v-expand-transition>
</template>

<script setup>
import { ref, reactive, watch, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  isPlaying: Boolean
})

const emit = defineEmits(['update-params', 'toggle-play', 'reset'])

const showControls = ref(false)
const showNavbar = ref(true)
let hideTimer = null
let isMouseOverControls = ref(false)

const params = reactive({
  numParticles: 50000,
  sensorDistance: 9.0,
  sensorAngle: 0.5,
  rotationAngle: 0.3,
  moveDistance: 1.0,
  decayFactor: 0.95,
  depositAmount: 5.0,
  resolution: 0.3
})

const presets = [
  {
    name: 'Default',
    color: 'blue',
    params: {
      numParticles: 50000,
      sensorDistance: 9.0,
      sensorAngle: 0.5,
      rotationAngle: 0.3,
      moveDistance: 1.0,
      decayFactor: 0.95,
      depositAmount: 5.0,
      resolution: 0.3
    }
  },
  {
    name: 'Dense',
    color: 'green',
    params: {
      numParticles: 75000,
      sensorDistance: 12.0,
      sensorAngle: 0.8,
      rotationAngle: 0.4,
      moveDistance: 0.8,
      decayFactor: 0.92,
      depositAmount: 8.0,
      resolution: 0.25
    }
  },
  {
    name: 'Wispy',
    color: 'purple',
    params: {
      numParticles: 30000,
      sensorDistance: 15.0,
      sensorAngle: 0.3,
      rotationAngle: 0.2,
      moveDistance: 1.5,
      decayFactor: 0.98,
      depositAmount: 3.0,
      resolution: 0.4
    }
  },
  {
    name: 'Chaos',
    color: 'red',
    params: {
      numParticles: 40000,
      sensorDistance: 6.0,
      sensorAngle: 1.2,
      rotationAngle: 0.8,
      moveDistance: 2.0,
      decayFactor: 0.85,
      depositAmount: 6.0,
      resolution: 0.35
    }
  },
  {
    name: 'Organic',
    color: 'orange',
    params: {
      numParticles: 60000,
      sensorDistance: 18.0,
      sensorAngle: 0.6,
      rotationAngle: 0.25,
      moveDistance: 1.2,
      decayFactor: 0.96,
      depositAmount: 10.0,
      resolution: 0.3
    }
  },
  {
    name: 'Fast',
    color: 'teal',
    params: {
      numParticles: 25000,
      sensorDistance: 8.0,
      sensorAngle: 0.4,
      rotationAngle: 0.5,
      moveDistance: 3.0,
      decayFactor: 0.9,
      depositAmount: 2.0,
      resolution: 0.5
    }
  }
]

function formatNumber(num) {
  if (num >= 1000000) {
    return (num / 1000000).toFixed(1) + 'M'
  } else if (num >= 1000) {
    return (num / 1000).toFixed(0) + 'K'
  }
  return num.toString()
}

function togglePlay() {
  emit('toggle-play')
}

function resetSimulation() {
  emit('reset')
}

function loadPreset(preset) {
  Object.assign(params, preset.params)
}

// Mouse tracking functions
function onMouseMove() {
  showNavbar.value = true
  clearTimeout(hideTimer)
  
  // Don't hide if controls are open or mouse is over controls
  if (!showControls.value && !isMouseOverControls.value) {
    hideTimer = setTimeout(() => {
      showNavbar.value = false
    }, 3000) // Hide after 3 seconds of no movement
  }
}

function onMouseEnterControls() {
  isMouseOverControls.value = true
  showNavbar.value = true
  clearTimeout(hideTimer)
}

function onMouseLeaveControls() {
  isMouseOverControls.value = false
  if (!showControls.value) {
    hideTimer = setTimeout(() => {
      showNavbar.value = false
    }, 3000)
  }
}

// Watch for controls panel state changes
watch(showControls, (isOpen) => {
  if (isOpen) {
    // Keep navbar visible when controls are open
    showNavbar.value = true
    clearTimeout(hideTimer)
  } else {
    // Start hide timer when controls are closed
    if (!isMouseOverControls.value) {
      hideTimer = setTimeout(() => {
        showNavbar.value = false
      }, 3000)
    }
  }
})

// Lifecycle hooks
onMounted(() => {
  document.addEventListener('mousemove', onMouseMove)
  // Start the initial hide timer
  hideTimer = setTimeout(() => {
    showNavbar.value = false
  }, 5000) // Show for 5 seconds initially
})

onUnmounted(() => {
  document.removeEventListener('mousemove', onMouseMove)
  clearTimeout(hideTimer)
})

// Watch for parameter changes and emit them
watch(params, (newParams) => {
  emit('update-params', { ...newParams })
}, { deep: true })

// Emit initial parameters
emit('update-params', { ...params })
</script>

<style scoped>
.control-navbar {
  z-index: 1000;
}

.controls-panel {
  max-height: 80vh;
  overflow-y: auto;
}

.control-group {
  margin-bottom: 8px;
}

.control-label {
  color: white;
  font-size: 0.875rem;
  font-weight: 500;
  display: block;
  margin-bottom: 4px;
}

.presets-section h4 {
  color: white;
}

/* Custom scrollbar for the controls panel */
.controls-panel::-webkit-scrollbar {
  width: 6px;
}

.controls-panel::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 3px;
}

.controls-panel::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 3px;
}

.controls-panel::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}
</style>
