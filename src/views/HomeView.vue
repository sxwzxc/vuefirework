<script setup lang="ts">
import ParticleCanvas from '../components/ParticleCanvas.vue'
</script>

<template>
  <main>
    <ParticleCanvas />
    <div class="hint">
      <p class="title">✨ 炫酷粒子宇宙</p>
      <p class="desc">移动鼠标 · 点击爆裂 · 按住螺旋 · 右上角打开控制面板</p>
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
  font-size: 1.2rem;
  color: rgba(255, 255, 255, 0.55);
  letter-spacing: 3px;
  margin-bottom: 0.5rem;
  text-shadow: 0 0 25px rgba(147, 51, 234, 0.4), 0 0 10px rgba(59, 130, 246, 0.3);
  font-weight: 500;
}

.desc {
  font-size: 0.85rem;
  color: rgba(255, 255, 255, 0.35);
  letter-spacing: 1px;
  line-height: 1.6;
}

@keyframes fadeInUp {
  0% {
    opacity: 0;
    transform: translateX(-50%) translateY(30px);
  }
  100% {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}
</style>
