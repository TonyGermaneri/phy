<template>
  <div class="lfo-controls" :class="{ disabled }">
    <div class="lfo-row">
      <span class="lfo-label">Rate: {{ rate.toFixed(2) }}Hz</span>
      <v-slider
        :model-value="rate"
        @update:model-value="$emit('update:rate', $event)"
        :min="0"
        :max="0.05"
        :step="0.001"
        :color="color + '-lighten-2'"
        track-color="grey-darken-2"
        thumb-size="12"
        :disabled="disabled"
        hide-details
        density="compact"
      />
    </div>
    <div class="lfo-row">
      <span class="lfo-label">Amp: {{ amplitude.toFixed(2) }}</span>
      <v-slider
        :model-value="amplitude"
        @update:model-value="$emit('update:amplitude', $event)"
        :min="0"
        :max="maxAmplitude"
        :step="step"
        :color="color + '-lighten-2'"
        track-color="grey-darken-2"
        thumb-size="12"
        :disabled="disabled"
        hide-details
        density="compact"
      />
    </div>
  </div>
</template>

<script setup>
defineProps({
  rate: { type: Number, required: true },
  amplitude: { type: Number, required: true },
  maxAmplitude: { type: Number, required: true },
  step: { type: Number, default: 0.1 },
  color: { type: String, default: 'blue' },
  disabled: { type: Boolean, default: false }
})

defineEmits(['update:rate', 'update:amplitude'])
</script>

<style scoped>
.lfo-controls {
  margin-top: 8px;
  padding: 8px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 6px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  transition: opacity 0.3s ease;
}

.lfo-controls.disabled {
  opacity: 0.4;
}

.lfo-row {
  display: flex;
  align-items: center;
  margin-bottom: 4px;
}

.lfo-row:last-child {
  margin-bottom: 0;
}

.lfo-label {
  color: rgba(255, 255, 255, 0.8);
  font-size: 0.75rem;
  width: 80px;
  flex-shrink: 0;
  margin-right: 8px;
}

.disabled .lfo-label {
  color: rgba(255, 255, 255, 0.4);
}
</style>
