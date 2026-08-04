<script setup lang="ts">
withDefaults(
  defineProps<{
    disabled?: boolean
    loading?: boolean
    loadingLabel?: string
    surface?: 'console' | 'gateway'
    type?: 'button' | 'submit' | 'reset'
    variant?: 'primary' | 'secondary' | 'ghost' | 'danger'
  }>(),
  {
    disabled: false,
    loading: false,
    loadingLabel: '',
    surface: 'console',
    type: 'button',
    variant: 'primary',
  }
)
</script>

<template>
  <button
    :type="type"
    class="cyber-button"
    :class="[`cyber-button--${variant}`, `cyber-button--${surface}`]"
    :disabled="disabled || loading"
    :aria-busy="loading || undefined"
  >
    <span v-if="loading" class="cyber-spinner" aria-hidden="true" />
    <slot v-else name="icon" />
    <span>{{ loading ? loadingLabel : undefined }}<slot v-if="!loading" /></span>
  </button>
</template>
