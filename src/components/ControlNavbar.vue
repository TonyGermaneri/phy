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
      icon="mdi-dice-6"
      variant="text"
      color="white"
      @click="randomizeParams"
      title="Randomize Parameters"
    />

    <v-btn
      :icon="lfoEnabled ? 'mdi-sine-wave' : 'mdi-sine-wave'"
      variant="text"
      :color="lfoEnabled ? 'green' : 'grey'"
      @click="toggleLFO"
      :title="lfoEnabled ? 'LFOs ON - Click to disable' : 'LFOs OFF - Click to enable'"
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
                @update:model-value="updateBaseValue('sensorDistance', $event)"
              />
              <!-- LFO Controls -->
              <LfoControl
                v-model:rate="lfoParams.sensorDistance.rate"
                v-model:amplitude="lfoParams.sensorDistance.amplitude"
                :max-amplitude="20"
                :step="0.1"
                :disabled="!lfoEnabled"
                color="green"
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
                @update:model-value="updateBaseValue('sensorAngle', $event)"
              />
              <!-- LFO Controls -->
              <LfoControl
                v-model:rate="lfoParams.sensorAngle.rate"
                v-model:amplitude="lfoParams.sensorAngle.amplitude"
                :max-amplitude="Math.PI"
                :step="0.02"
                :disabled="!lfoEnabled"
                color="orange"
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
                @update:model-value="updateBaseValue('rotationAngle', $event)"
              />
              <!-- LFO Controls -->
              <LfoControl
                v-model:rate="lfoParams.rotationAngle.rate"
                v-model:amplitude="lfoParams.rotationAngle.amplitude"
                :max-amplitude="Math.PI / 2"
                :step="0.01"
                :disabled="!lfoEnabled"
                color="purple"
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
                @update:model-value="updateBaseValue('moveDistance', $event)"
              />
              <!-- LFO Controls -->
              <LfoControl
                v-model:rate="lfoParams.moveDistance.rate"
                v-model:amplitude="lfoParams.moveDistance.amplitude"
                :max-amplitude="5"
                :step="0.1"
                :disabled="!lfoEnabled"
                color="red"
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
                @update:model-value="updateBaseValue('decayFactor', $event)"
              />
              <!-- LFO Controls -->
              <LfoControl
                v-model:rate="lfoParams.decayFactor.rate"
                v-model:amplitude="lfoParams.decayFactor.amplitude"
                :max-amplitude="1"
                :step="0.01"
                :disabled="!lfoEnabled"
                color="cyan"
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
                @update:model-value="updateBaseValue('depositAmount', $event)"
              />
              <!-- LFO Controls -->
              <LfoControl
                v-model:rate="lfoParams.depositAmount.rate"
                v-model:amplitude="lfoParams.depositAmount.amplitude"
                :max-amplitude="20"
                :step="0.2"
                :disabled="!lfoEnabled"
                color="yellow"
              />
            </div>
          </v-col>

          <!-- Particle Count -->
          <v-col cols="12" sm="6" md="4">
            <div class="control-group">
              <label class="control-label">Particles: {{ formatNumber(params.numParticles) }}</label>
              <v-slider
                v-model="params.numParticles"
                :min="5000"
                :max="1000000"
                :step="1"
                color="blue"
                track-color="grey-darken-1"
                thumb-color="blue"
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
import LfoControl from './LfoControl.vue'

const props = defineProps({
  isPlaying: Boolean
})

const emit = defineEmits(['update-params', 'toggle-play', 'reset'])

const showControls = ref(false)
const showNavbar = ref(true)
const lfoEnabled = ref(true)
let hideTimer = null
let isMouseOverControls = ref(false)

const params = reactive({
  numParticles: 1000000,
  sensorDistance: 9.0,
  sensorAngle: 0.5,
  rotationAngle: 0.3,
  moveDistance: 1.0,
  decayFactor: 0.95,
  depositAmount: 5.0,
  resolution: 1.0
})

// LFO parameters for each control (removed numParticles)
const lfoParams = reactive({
  sensorDistance: { rate: 0, amplitude: 0, baseValue: 9.0 },
  sensorAngle: { rate: 0, amplitude: 0, baseValue: 0.5 },
  rotationAngle: { rate: 0, amplitude: 0, baseValue: 0.3 },
  moveDistance: { rate: 0, amplitude: 0, baseValue: 1.0 },
  decayFactor: { rate: 0, amplitude: 0, baseValue: 0.95 },
  depositAmount: { rate: 0, amplitude: 0, baseValue: 5.0 }
})

let animationId = null
let startTime = Date.now()

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
      depositAmount: 5.0
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
      depositAmount: 8.0
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
      depositAmount: 3.0
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
      depositAmount: 6.0
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
      depositAmount: 10.0
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
      depositAmount: 2.0
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

function randomizeParams() {
  // Randomize all parameters within reasonable ranges
  Object.assign(params, {
    numParticles: Math.floor(Math.random() * 750000) + 150000, // 150K - 900K
    sensorDistance: Math.random() * 15 + 5, // 5 - 20
    sensorAngle: Math.random() * Math.PI * 0.8 + 0.1, // 0.1 - 2.6 radians
    rotationAngle: Math.random() * Math.PI * 0.4 + 0.1, // 0.1 - 1.3 radians
    moveDistance: Math.random() * 3 + 0.5, // 0.5 - 3.5
    decayFactor: Math.random() * 0.25 + 0.75, // 0.75 - 1.0
    depositAmount: Math.random() * 15 + 2, // 2 - 17
  })

  // Randomize LFO parameters
  Object.keys(lfoParams).forEach(paramName => {
    const lfo = lfoParams[paramName]

    // 50% chance to enable LFO for each parameter
    if (Math.random() < 0.5) {
      lfo.rate = Math.random() * 0.03 + 0.005 // 0.005 - 0.035 Hz (much slower)

      // Parameter-specific amplitude ranges
      switch (paramName) {
        case 'sensorDistance':
          lfo.amplitude = Math.random() * 3 + 0.5 // 0.5 - 3.5
          break
        case 'sensorAngle':
          lfo.amplitude = Math.random() * 0.4 + 0.1 // 0.1 - 0.5
          break
        case 'rotationAngle':
          lfo.amplitude = Math.random() * 0.2 + 0.05 // 0.05 - 0.25
          break
        case 'moveDistance':
          lfo.amplitude = Math.random() * 1.0 + 0.2 // 0.2 - 1.2
          break
        case 'decayFactor':
          lfo.amplitude = Math.random() * 0.08 + 0.02 // 0.02 - 0.1
          break
        case 'depositAmount':
          lfo.amplitude = Math.random() * 4 + 1 // 1 - 5
          break
      }
    } else {
      // Disable LFO for this parameter
      lfo.rate = 0
      lfo.amplitude = 0
    }
  })

  // Update base values for LFOs
  updateAllBaseValues()
}

function toggleLFO() {
  lfoEnabled.value = !lfoEnabled.value

  if (!lfoEnabled.value) {
    // When disabling LFOs, stop all oscillations by setting rates to 0
    Object.keys(lfoParams).forEach(paramName => {
      lfoParams[paramName].rate = 0
    })
  }
}

function updateBaseValue(paramName, value) {
  if (lfoParams[paramName]) {
    lfoParams[paramName].baseValue = value
  }
}

function updateAllBaseValues() {
  Object.keys(params).forEach(key => {
    if (lfoParams[key]) {
      lfoParams[key].baseValue = params[key]
    }
  })
}

// LFO Animation function
function animateLFOs() {
  const currentTime = (Date.now() - startTime) / 1000

  // Only animate LFOs if globally enabled
  if (lfoEnabled.value) {
    Object.keys(lfoParams).forEach(paramName => {
      const lfo = lfoParams[paramName]
      if (lfo.rate > 0 && lfo.amplitude > 0) {
        // Calculate oscillation
        const oscillation = Math.sin(currentTime * lfo.rate * Math.PI * 2) * lfo.amplitude

        // Apply to parameter with clamping to valid ranges
        let newValue = lfo.baseValue + oscillation

        // Clamp to parameter-specific ranges
        switch (paramName) {
          case 'sensorDistance':
            newValue = Math.max(1, Math.min(20, newValue))
            params[paramName] = Math.round(newValue * 10) / 10 // One decimal place
            break
          case 'sensorAngle':
            newValue = Math.max(0, Math.min(Math.PI, newValue))
            params[paramName] = Math.round(newValue * 100) / 100 // Two decimal places
            break
          case 'rotationAngle':
            newValue = Math.max(0, Math.min(Math.PI / 2, newValue))
            params[paramName] = Math.round(newValue * 100) / 100
            break
          case 'moveDistance':
            newValue = Math.max(0.1, Math.min(5, newValue))
            params[paramName] = Math.round(newValue * 10) / 10
            break
          case 'decayFactor':
            newValue = Math.max(0.1, Math.min(1, newValue))
            params[paramName] = Math.round(newValue * 100) / 100
            break
          case 'depositAmount':
            newValue = Math.max(0.1, Math.min(20, newValue))
            params[paramName] = Math.round(newValue * 10) / 10
            break
        }
      }
    })
  }

  animationId = requestAnimationFrame(animateLFOs)
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
  updateAllBaseValues() // Initialize base values
  animateLFOs() // Start LFO animation
  // Start the initial hide timer
  hideTimer = setTimeout(() => {
    showNavbar.value = false
  }, 5000) // Show for 5 seconds initially
  randomizeParams();
})

onUnmounted(() => {
  document.removeEventListener('mousemove', onMouseMove)
  clearTimeout(hideTimer)
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
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
