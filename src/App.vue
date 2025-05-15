<script setup>
import TextUploader from './components/TextUploader.vue'
</script>

<template>
  <div class="app">
    <h1>English Dictation</h1>
    <TextUploader />
  </div>
</template>

<style scoped>
.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

h1 {
  text-align: center;
  color: #2c3e50;
  margin-bottom: 30px;
}
</style>
