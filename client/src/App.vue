<script setup lang="ts">
import WebRTCCall from './components/WebRTCCall.vue'
</script>

<template>
  <div id="app">
    <h1>WebRTC 视频通话应用</h1>
    <WebRTCCall />
  </div>
</template>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}

h1 {
  color: #42b883;
  margin-bottom: 30px;
}
</style>
