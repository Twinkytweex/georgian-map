<script setup>
import MapComponent from './components/MapComponent.vue'
</script>

<template>
  <MapComponent />
</template>

<style>
/* Global reset to ensure MapComponent takes full space */
html, body, #app {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}
</style>
