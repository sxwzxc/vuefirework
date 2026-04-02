<script setup lang="ts">
import ParticleCanvas from '../components/ParticleCanvas.vue'
</script>

<template>
  <main>
    <ParticleCanvas />
    <div class="hint">
      <p class="title">✨ 炫酷粒子</p>
      <p class="desc">移动鼠标感受粒子跟随 · 点击或按住鼠标触发螺旋漩涡</p>
    </div>
  </main>
</template>

<style scoped>
main {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.hint {
  position: fixed;
  bottom: 2.5rem;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  text-align: center;
  pointer-events: none;
  animation: fadeInUp 2s ease-out;
}

.title {
  font-size: 1.1rem;
  color: rgba(255, 255, 255, 0.5);
  letter-spacing: 2px;
  margin-bottom: 0.4rem;
  text-shadow: 0 0 20px rgba(147, 51, 234, 0.3);
}

.desc {
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.3);
  letter-spacing: 0.5px;
}

@keyframes fadeInUp {
  0% {
    opacity: 0;
    transform: translateX(-50%) translateY(20px);
  }
  100% {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}
</style>
