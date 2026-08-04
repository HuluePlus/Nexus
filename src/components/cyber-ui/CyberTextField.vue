<script setup lang="ts">
import CyberFieldLabel from './CyberFieldLabel.vue'

defineOptions({ inheritAttrs: false })

withDefaults(
  defineProps<{
    id: string
    label: string
    modelValue: string
    surface?: 'console' | 'gateway'
  }>(),
  { surface: 'console' }
)

defineEmits<{ 'update:modelValue': [value: string] }>()
</script>

<template>
  <div class="cyber-field-group">
    <CyberFieldLabel :for-id="id">{{ label }}</CyberFieldLabel>
    <div class="cyber-field" :class="`cyber-field--${surface}`">
      <span v-if="$slots.icon" class="cyber-field__icon"><slot name="icon" /></span>
      <input
        :id="id"
        v-bind="$attrs"
        :value="modelValue"
        @input="$emit('update:modelValue', ($event.target as HTMLInputElement).value)"
      >
      <span v-if="$slots.suffix" class="cyber-field__suffix"><slot name="suffix" /></span>
    </div>
  </div>
</template>
