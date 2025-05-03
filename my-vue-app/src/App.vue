<script setup lang="ts">
import Particles from './components/pricticles.vue'
import Circle from './components/circle.vue'
import { ref } from 'vue'
const loadingProgress = ref(0)
let progressInterval = 0 // 直接使用 number 类型

function start() {
  if (progressInterval) {
    clearInterval(progressInterval)
  }
  loadingProgress.value = 0
  progressInterval = window.setInterval(() => {
    if (loadingProgress.value >= 100) {
      window.clearInterval(progressInterval)
      progressInterval = 0
    } else {
      loadingProgress.value += 1
    }
  }, 50)
}
</script>

<template>
  <div id="container">
    <button
      @click="start"
      style="position: absolute; top: 0; left: 0; color: black; z-index: 2"
    >
      增加进度
    </button>
    <Circle :progress="loadingProgress" style="z-index: 1" />
    <Particles />
  </div>
</template>

<style scoped></style>
