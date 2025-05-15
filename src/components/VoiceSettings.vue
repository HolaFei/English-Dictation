<template>
  <div v-if="modelValue" class="modal-overlay" @click="closeModal">
    <div class="modal-content" @click.stop>
      <div class="modal-header">
        <h3>语音设置</h3>
        <button class="close-button" @click="closeModal">&times;</button>
      </div>
      <div class="modal-body">
        <div class="setting-item">
          <label>语音类型：</label>
          <div class="voice-filter">
            <button
              v-for="type in voiceTypes"
              :key="type.value"
              @click="selectedType = type.value"
              :class="['type-button', { active: selectedType === type.value }]"
            >
              {{ type.label }}
            </button>
          </div>
          <select v-model="selectedVoice" class="voice-select" @change="previewVoice">
            <option v-for="voice in filteredVoices" :key="voice.name" :value="voice">
              {{ voice.name }}
            </option>
          </select>
          <button class="preview-button" @click="previewVoice" :disabled="isPreviewing">
            {{ isPreviewing ? '正在播放...' : '试听语音' }}
          </button>
        </div>
        <div class="setting-item">
          <label>语速：</label>
          <input type="range" v-model="rate" min="0.5" max="2" step="0.1" class="slider" />
          <span class="value">{{ rate }}</span>
        </div>
        <div class="setting-item">
          <label>音调：</label>
          <input type="range" v-model="pitch" min="0.5" max="2" step="0.1" class="slider" />
          <span class="value">{{ pitch }}</span>
        </div>
        <div class="setting-item">
          <label>音量：</label>
          <input type="range" v-model="volume" min="0" max="1" step="0.1" class="slider" />
          <span class="value">{{ volume }}</span>
        </div>
      </div>
      <div class="modal-footer">
        <button class="save-button" @click="saveSettings">保存设置</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue'

const props = defineProps({
  modelValue: Boolean,
  currentSettings: {
    type: Object,
    default: () => ({
      voice: null,
      rate: 1,
      pitch: 1,
      volume: 1,
    }),
  },
})

const emit = defineEmits(['update:modelValue', 'save'])

const voices = ref([])
const selectedVoice = ref(null)
const rate = ref(props.currentSettings.rate)
const pitch = ref(props.currentSettings.pitch)
const volume = ref(props.currentSettings.volume)
const selectedType = ref('en-US')
const isPreviewing = ref(false)

// 定义英语语音类型
const voiceTypes = [
  { label: '美式英语', value: 'en-US' },
  { label: '英式英语', value: 'en-GB' },
]

// 根据选择的类型筛选语音
const filteredVoices = computed(() => {
  return voices.value.filter((voice) => voice.lang === selectedType.value)
})

// 预览文本
const previewText = 'Hello, this is a preview of the selected voice. How does it sound?'

// 预览语音
const previewVoice = () => {
  if (!selectedVoice.value) return

  // 停止当前正在播放的语音
  window.speechSynthesis.cancel()

  const utterance = new SpeechSynthesisUtterance(previewText)
  utterance.voice = selectedVoice.value
  utterance.rate = rate.value
  utterance.pitch = pitch.value
  utterance.volume = volume.value

  utterance.onstart = () => {
    isPreviewing.value = true
  }

  utterance.onend = () => {
    isPreviewing.value = false
  }

  utterance.onerror = () => {
    isPreviewing.value = false
  }

  window.speechSynthesis.speak(utterance)
}

// 获取可用的语音列表
const loadVoices = () => {
  const availableVoices = window.speechSynthesis.getVoices()
  if (availableVoices.length > 0) {
    voices.value = availableVoices

    // 尝试找到最佳的默认语音
    // 优先选择 Microsoft David Desktop (en-US) 或 Google US English
    const preferredVoices = [
      'Microsoft David Desktop',
      'Google US English',
      'Microsoft Zira Desktop',
      'Microsoft Mark Desktop',
    ]

    // 首先尝试找到首选语音
    let defaultVoice = voices.value.find(
      (voice) => preferredVoices.includes(voice.name) && voice.lang === 'en-US',
    )

    // 如果没有找到首选语音，则选择第一个美式英语语音
    if (!defaultVoice) {
      defaultVoice = voices.value.find((voice) => voice.lang === 'en-US') || voices.value[0]
    }

    selectedVoice.value = defaultVoice
    if (selectedVoice.value) {
      selectedType.value = selectedVoice.value.lang
    }
  }
}

onMounted(() => {
  // 立即尝试加载语音
  loadVoices()

  // 某些浏览器需要等待语音列表加载完成
  if (window.speechSynthesis.onvoiceschanged !== undefined) {
    window.speechSynthesis.onvoiceschanged = loadVoices
  }
})

// 监听模态框显示状态
watch(
  () => props.modelValue,
  (newValue) => {
    if (newValue) {
      // 当模态框显示时，重新加载语音列表
      loadVoices()
    }
  },
)

const closeModal = () => {
  // 停止当前正在播放的语音
  window.speechSynthesis.cancel()
  emit('update:modelValue', false)
}

const saveSettings = () => {
  // 停止当前正在播放的语音
  window.speechSynthesis.cancel()
  emit('save', {
    voice: selectedVoice.value,
    rate: rate.value,
    pitch: pitch.value,
    volume: volume.value,
  })
  closeModal()
}
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
  background: white;
  padding: 20px;
  border-radius: 8px;
  width: 90%;
  max-width: 500px;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.modal-header h3 {
  margin: 0;
  color: #2c3e50;
}

.close-button {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #666;
}

.setting-item {
  margin-bottom: 20px;
}

.setting-item label {
  display: block;
  margin-bottom: 8px;
  color: #2c3e50;
}

.voice-filter {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

.type-button {
  padding: 8px 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  transition: all 0.3s;
  flex: 1;
}

.type-button:hover {
  background-color: #f5f5f5;
}

.type-button.active {
  background-color: #2196f3;
  color: white;
  border-color: #2196f3;
}

.voice-select {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-bottom: 10px;
}

.preview-button {
  width: 100%;
  padding: 8px 16px;
  background-color: #ff9800;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-bottom: 10px;
}

.preview-button:hover:not(:disabled) {
  background-color: #f57c00;
}

.preview-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.slider {
  width: 100%;
  margin: 10px 0;
}

.value {
  display: inline-block;
  margin-left: 10px;
  color: #666;
}

.save-button {
  background-color: #4caf50;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  cursor: pointer;
  width: 100%;
  font-size: 16px;
}

.save-button:hover {
  background-color: #45a049;
}
</style>
