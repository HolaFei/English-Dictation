<template>
  <div v-if="modelValue" class="modal-overlay">
    <div class="modal-content">
      <h2>🎉 恭喜完成！</h2>
      <p>你已经完成了所有句子的听写练习！</p>
      <button @click="$emit('update:modelValue', false)" class="close-button">关闭</button>
    </div>
    <canvas ref="confettiCanvas" class="confetti-canvas"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue'

const props = defineProps({
  modelValue: Boolean,
})

const emit = defineEmits(['update:modelValue'])
const confettiCanvas = ref(null)
let confetti = []

// 创建彩条
const createConfetti = () => {
  const canvas = confettiCanvas.value
  if (!canvas) return

  const ctx = canvas.getContext('2d')
  canvas.width = window.innerWidth
  canvas.height = window.innerHeight

  // 清空之前的彩条
  confetti = []

  // 彩条颜色
  const colors = [
    '#FF5252',
    '#FF4081',
    '#E040FB',
    '#7C4DFF',
    '#536DFE',
    '#448AFF',
    '#40C4FF',
    '#18FFFF',
    '#64FFDA',
    '#69F0AE',
    '#B2FF59',
    '#EEFF41',
    '#FFFF00',
    '#FFD740',
    '#FFAB40',
    '#FF6E40',
  ]

  // 创建新的彩条
  for (let i = 0; i < 150; i++) {
    confetti.push({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height - canvas.height,
      width: Math.random() * 10 + 5,
      height: Math.random() * 10 + 5,
      color: colors[Math.floor(Math.random() * colors.length)],
      speed: Math.random() * 3 + 2,
      rotation: Math.random() * 360,
      rotationSpeed: Math.random() * 2 - 1,
    })
  }

  // 动画循环
  const animate = () => {
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    confetti.forEach((confetto) => {
      ctx.save()
      ctx.translate(confetto.x, confetto.y)
      ctx.rotate((confetto.rotation * Math.PI) / 180)
      ctx.fillStyle = confetto.color
      ctx.fillRect(-confetto.width / 2, -confetto.height / 2, confetto.width, confetto.height)
      ctx.restore()

      confetto.y += confetto.speed
      confetto.rotation += confetto.rotationSpeed

      if (confetto.y > canvas.height) {
        confetto.y = -confetto.height
        confetto.x = Math.random() * canvas.width
      }
    })

    requestAnimationFrame(animate)
  }

  animate()
}

// 监听窗口大小变化
const handleResize = () => {
  if (confettiCanvas.value) {
    confettiCanvas.value.width = window.innerWidth
    confettiCanvas.value.height = window.innerHeight
  }
}

onMounted(() => {
  // 确保在下一个 tick 时创建彩条，此时 canvas 已经渲染
  nextTick(() => {
    createConfetti()
  })
  window.addEventListener('resize', handleResize)
})

watch(
  () => props.modelValue,
  (newValue) => {
    if (newValue) {
      // 确保在下一个 tick 时创建彩条
      nextTick(() => {
        createConfetti()
      })
    }
  },
)
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  padding: 30px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  position: relative;
  z-index: 1001;
}

.modal-content h2 {
  margin: 0 0 15px 0;
  color: #2196f3;
  font-size: 24px;
}

.modal-content p {
  margin: 0 0 20px 0;
  color: #666;
}

.close-button {
  padding: 8px 20px;
  background-color: #2196f3;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.close-button:hover {
  background-color: #1976d2;
}

.confetti-canvas {
  position: fixed;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 1000;
}
</style>
