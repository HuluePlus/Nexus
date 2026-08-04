<script setup lang="ts">
withDefaults(defineProps<{ tone?: 'accent' | 'success' | 'danger' }>(), { tone: 'accent' })
</script>

<template>
  <span class="cyber-status-dot" :class="`cyber-status-dot--${tone}`" aria-hidden="true" />
</template>
