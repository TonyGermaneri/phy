<template>
  <v-app>
    <!-- Physarum simulation takes up full screen -->
    <PhysarumCanvas
      ref="simulation"
      :numParticles="simulationParams.numParticles"
      :sensorDistance="simulationParams.sensorDistance"
      :sensorAngle="simulationParams.sensorAngle"
      :rotationAngle="simulationParams.rotationAngle"
      :moveDistance="simulationParams.moveDistance"
      :decayFactor="simulationParams.decayFactor"
      :depositAmount="simulationParams.depositAmount"
      :isPlaying="isPlaying"
      @ready="onSimulationReady"
    />

    <!-- Control navbar floating over the simulation -->
    <ControlNavbar
      :isPlaying="isPlaying"
      @update-params="updateParams"
      @toggle-play="togglePlay"
      @reset="resetSimulation"
    />

    <!-- Loading overlay -->
    <v-overlay
      v-if="!isReady"
      class="d-flex align-center justify-center"
      persistent
    >
      <v-progress-circular
        color="primary"
        indeterminate
        size="64"
      />
      <div class="ml-4 text-h6">Loading Physarum Simulation...</div>
    </v-overlay>
  </v-app>
</template>

<script setup>
import { ref, reactive } from 'vue'
import PhysarumCanvas from './components/PhysarumCanvas.vue'
import ControlNavbar from './components/ControlNavbar.vue'

const simulation = ref(null)
const isReady = ref(false)
const isPlaying = ref(true)

const simulationParams = reactive({
  numParticles: 100000,
  sensorDistance: 9.0,
  sensorAngle: 0.5,
  rotationAngle: 0.3,
  moveDistance: 1.0,
  decayFactor: 0.9,
  depositAmount: 5.0
})

function onSimulationReady() {
  isReady.value = true
}

function updateParams(params) {
  Object.assign(simulationParams, params)
}

function togglePlay() {
  isPlaying.value = !isPlaying.value
}

function resetSimulation() {
  if (simulation.value) {
    simulation.value.resetSimulation()
  }
}
</script>

<style scoped>
.v-app {
  background: #000 !important;
}
</style>
