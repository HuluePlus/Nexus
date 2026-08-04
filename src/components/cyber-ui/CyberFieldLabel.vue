<script setup lang="ts">
defineProps<{ forId?: string }>()
</script>

<template>
  <label class="cyber-field-label" :for="forId"><slot /></label>
</template>
