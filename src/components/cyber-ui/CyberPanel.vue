<script setup lang="ts">
withDefaults(defineProps<{ variant?: 'default' | 'metric' | 'danger' }>(), {
  variant: 'default',
})
</script>

<template>
  <section class="cyber-panel" :class="`cyber-panel--${variant}`"><slot /></section>
</template>
