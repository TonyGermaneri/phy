<template>
  <v-app>
    <!-- Physarum simulation takes up full screen -->
    <PhysarumTexture
      ref="simulation"
      :numParticles="simulationParams.numParticles"
      :sensorDistance="simulationParams.sensorDistance"
      :sensorAngle="simulationParams.sensorAngle"
      :rotationAngle="simulationParams.rotationAngle"
      :moveDistance="simulationParams.moveDistance"
      :decayFactor="simulationParams.decayFactor"
      :depositAmount="simulationParams.depositAmount"
      :diffusionStrength="simulationParams.diffusionStrength"
      :blurRadius="simulationParams.blurRadius"
      :trailFade="simulationParams.trailFade"
      :edgeEnhancement="simulationParams.edgeEnhancement"
      :resolution="simulationParams.resolution"
      :colorRemap="simulationParams.colorRemap"
      :hueOffset="simulationParams.hueOffset"
      :hueSpeed="simulationParams.hueSpeed"
      :saturation="simulationParams.saturation"
      :brightness="simulationParams.brightness"
      :contrast="simulationParams.contrast"
      :spawnRate="simulationParams.spawnRate"
      :spawnRadius="simulationParams.spawnRadius"
      :isPlaying="isPlaying"
      @ready="onSimulationReady"
      @double-tap="onDoubleTap"
    />

    <!-- Control navbar floating over the simulation -->
    <ControlNavbar
      ref="controlNavbar"
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
import PhysarumTexture from './components/PhysarumTexture.vue'
import ControlNavbar from './components/ControlNavbar.vue'

const simulation = ref(null)
const controlNavbar = ref(null)
const isReady = ref(false)
const isPlaying = ref(true)

const simulationParams = reactive({
  numParticles: 50000,
  sensorDistance: 9.0,
  sensorAngle: 0.5,
  rotationAngle: 0.3,
  moveDistance: 1.0,
  decayFactor: 0.95,
  depositAmount: 5.0,
  diffusionStrength: 0.3,
  blurRadius: 1.5,
  trailFade: 0.98,
  edgeEnhancement: 0.1,
  resolution: 0.3,
  colorRemap: 0,
  hueOffset: 0.0,
  hueSpeed: 0.01,
  saturation: 0.8,
  brightness: 1.0,
  contrast: 1.0,
  spawnRate: 5.0,
  spawnRadius: 30.0
})

function onSimulationReady() {
  isReady.value = true
}

function updateParams(params) {
  const oldNumParticles = simulationParams.numParticles
  const oldResolution = simulationParams.resolution

  Object.assign(simulationParams, params)

  // Check if parameters that require restart have changed
  const needsRestart = (
    params.numParticles !== oldNumParticles ||
    params.resolution !== oldResolution
  )

  if (needsRestart && simulation.value) {
    // Small delay to ensure parameter updates are applied first
    setTimeout(() => {
      simulation.value.resetSimulation()
    }, 50)
  }
}

function togglePlay() {
  isPlaying.value = !isPlaying.value
}

function resetSimulation() {
  if (simulation.value) {
    simulation.value.resetSimulation()
  }
}

function onDoubleTap() {
  if (controlNavbar.value) {
    controlNavbar.value.showControlsPanel()
  }
}
</script>

<style scoped>
.v-app {
  background: #000 !important;
}
</style>
