<script setup lang="ts">
withDefaults(
  defineProps<{
    disabled?: boolean
    label: string
    type?: 'button' | 'submit' | 'reset'
  }>(),
  { disabled: false, type: 'button' }
)
</script>

<template>
  <button
    :type="type"
    class="cyber-icon-button"
    :title="label"
    :aria-label="label"
    :disabled="disabled"
  >
    <slot />
  </button>
</template>
