<template>
  <div class="text-uploader">
    <CourseList @select-course="loadCourseContent" />
    <div class="content-wrapper">
      <div class="upload-section" :class="{ 'upload-section--centered': !fileContent }">
        <input
          type="file"
          accept=".txt"
          @change="handleFileUpload"
          class="file-input"
          id="file-input"
        />
        <label for="file-input" class="upload-button"> 选择文本文件 </label>
      </div>
      <div v-if="fileContent" class="content-section">
        <div class="content-header">
          <h3>文件内容：</h3>
          <div class="button-group">
            <button
              @click="toggleSpeaking"
              class="speak-button"
              :class="{ 'speak-button--stop': isSpeaking }"
            >
              {{ isSpeaking ? '停止朗读' : '朗读文本' }}
            </button>
            <button v-if="!isDictating" @click="startDictation" class="dictate-button">
              开始听写
            </button>
            <button
              v-else
              @click="nextSentence"
              class="dictate-button"
              :disabled="!isCurrentSentenceComplete"
            >
              下一句
            </button>
            <button
              v-if="isDictating"
              @click="speakCurrentSentence"
              class="repeat-button"
              :disabled="isSpeaking"
              title="重复播放当前句子 (Ctrl + ')"
            >
              <svg viewBox="0 0 24 24" class="repeat-icon">
                <path d="M7 7h10v3l4-4-4-4v3H5v6h2V7zm10 10H7v-3l-4 4 4 4v-3h12v-6h-2v4z" />
              </svg>
            </button>
            <button @click="showSettings = true" class="settings-button" title="语音设置">
              <svg viewBox="0 0 24 24" class="settings-icon">
                <path
                  d="M19.14 12.94c.04-.3.06-.61.06-.94 0-.32-.02-.64-.07-.94l2.03-1.58c.18-.14.23-.41.12-.61l-1.92-3.32c-.12-.22-.37-.29-.59-.22l-2.39.96c-.5-.38-1.03-.7-1.62-.94l-.36-2.54c-.04-.24-.24-.41-.48-.41h-3.84c-.24 0-.43.17-.47.41l-.36 2.54c-.59.24-1.13.57-1.62.94l-2.39-.96c-.22-.08-.47 0-.59.22L2.74 8.87c-.12.21-.08.47.12.61l2.03 1.58c-.05.3-.07.62-.07.94s.02.64.07.94l-2.03 1.58c-.18.14-.23.41-.12.61l1.92 3.32c.12.22.37.29.59.22l2.39-.96c.5.38 1.03.7 1.62.94l.36 2.54c.05.24.24.41.48.41h3.84c.24 0 .44-.17.47-.41l.36-2.54c.59-.24 1.13-.56 1.62-.94l2.39.96c.22.08.47 0 .59-.22l1.92-3.32c.12-.22.07-.47-.12-.61l-2.01-1.58zM12 15.6c-1.98 0-3.6-1.62-3.6-3.6s1.62-3.6 3.6-3.6 3.6 1.62 3.6 3.6-1.62 3.6-3.6 3.6z"
                />
              </svg>
            </button>
          </div>
        </div>
        <div v-if="isDictating" class="dictation-section">
          <div class="current-sentence">
            <div
              v-for="(word, index) in currentSentenceWords"
              :key="index"
              class="word-container"
              :class="{ 'current-word': index === currentWordIndex }"
            >
              <div class="input-text" :class="{ correct: isWordCorrect(index) }">
                {{ isPunctuation(word) ? word : userInput[index] || '' }}
              </div>
              <div class="underline" :style="{ width: word.length * 20 + 'px' }"></div>
            </div>
          </div>
          <div class="sentence-controls">
            <div class="sentence-progress">
              第 {{ currentSentenceIndex + 1 }} 句 / 共 {{ sentences.length }} 句
              <div class="hint-text" v-if="isCurrentSentenceComplete">(按回车键进入下一句)</div>
              <div class="hint-text">(按 Ctrl + ' 重复播放当前句子)</div>
              <div class="hint-text">(按 Ctrl + ; 查看当前句子内容)</div>
            </div>
          </div>
        </div>
        <div v-if="completedSentences.length > 0" class="completed-sentences">
          <h4>已完成句子：</h4>
          <div
            v-for="(sentence, index) in completedSentences"
            :key="index"
            class="completed-sentence"
          >
            <span class="sentence-text">{{ sentence }}</span>
            <button class="replay-button" @click="replaySentence(sentence)" title="重新朗读此句">
              <svg viewBox="0 0 24 24" class="replay-icon">
                <path
                  d="M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"
                />
              </svg>
            </button>
          </div>
        </div>
        <pre v-if="!isDictating" class="file-content">{{ fileContent }}</pre>
      </div>
    </div>
    <VoiceSettings
      v-model="showSettings"
      :current-settings="voiceSettings"
      @save="updateVoiceSettings"
    />
    <SentencePreview v-model="showSentencePreview" :sentence="currentSentence" />
    <CongratulationsModal v-model="showCongratulations" />
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'
import { ElMessage } from 'element-plus'
import VoiceSettings from './VoiceSettings.vue'
import SentencePreview from './SentencePreview.vue'
import CongratulationsModal from './CongratulationsModal.vue'
import CourseList from './CourseList.vue'

const fileContent = ref('')
const isSpeaking = ref(false)
const showSettings = ref(false)
const showCongratulations = ref(false)
const voiceSettings = ref({
  voice: null,
  rate: 1,
  pitch: 1,
  volume: 1,
})

// 听写相关状态
const isDictating = ref(false)
const sentences = ref([])
const currentSentenceIndex = ref(0)
const userInput = ref([])
const currentWordIndex = ref(0)
const completedSentences = ref([])

const currentSentenceWords = computed(() => {
  if (!sentences.value[currentSentenceIndex.value]) return []
  return sentences.value[currentSentenceIndex.value].match(/\b\w+(?:'\w+)?\b|[.,!?;:]/g) || []
})

// 检查是否是标点符号
const isPunctuation = (word) => {
  return /[.,!?;:]/.test(word)
}

// 检查当前句子是否完成
const isCurrentSentenceComplete = computed(() => {
  if (!currentSentenceWords.value.length) return false
  return currentSentenceWords.value.every((word, index) => {
    // 如果是标点符号，直接返回true
    if (isPunctuation(word)) return true
    // 对于单词，不区分大小写进行比较
    return userInput.value[index]?.toLowerCase() === word.toLowerCase()
  })
})

// 检查单词是否正确
const isWordCorrect = (index) => {
  const word = currentSentenceWords.value[index]
  // 如果是标点符号，直接返回true
  if (isPunctuation(word)) return true
  // 对于单词，不区分大小写进行比较
  const input = userInput.value[index] || ''
  return input.toLowerCase() === word.toLowerCase()
}

// 创建打字音效
const baseUrl = import.meta.env.BASE_URL
const typeSound = new Audio(`${baseUrl}sounds/type.mp3`)
typeSound.volume = 0.6 // 设置音量为30%

// 播放打字音效
const playTypeSound = () => {
  // 重置音频到开始位置
  typeSound.currentTime = 0
  // 播放音效
  typeSound.play().catch((error) => {
    console.log('Error playing type sound:', error)
  })
}

// 朗读当前句子
const speakCurrentSentence = () => {
  if (!sentences.value[currentSentenceIndex.value]) return

  const utterance = new SpeechSynthesisUtterance(sentences.value[currentSentenceIndex.value])

  if (voiceSettings.value.voice) {
    utterance.voice = voiceSettings.value.voice
  }
  utterance.rate = voiceSettings.value.rate
  utterance.pitch = voiceSettings.value.pitch
  utterance.volume = voiceSettings.value.volume

  // 添加朗读状态监听
  utterance.onstart = () => {
    isSpeaking.value = true
  }

  utterance.onend = () => {
    isSpeaking.value = false
  }

  utterance.onerror = () => {
    isSpeaking.value = false
  }

  window.speechSynthesis.speak(utterance)
}

// 句子预览相关
const showSentencePreview = ref(false)
const currentSentence = computed(() => {
  return sentences.value[currentSentenceIndex.value] || ''
})

// 处理键盘输入
const handleKeyPress = (event) => {
  if (!isDictating.value) return

  // 如果当前句子已完成，按回车键进入下一句
  if (event.key === 'Enter' && isCurrentSentenceComplete.value) {
    nextSentence()
    return
  }

  // 按 Ctrl + ' 重复播放当前句子
  if (event.key === "'" && event.ctrlKey && !isSpeaking.value) {
    speakCurrentSentence()
    return
  }

  // 按 Ctrl + ; 显示当前句子
  if (event.key === ';' && event.ctrlKey) {
    showSentencePreview.value = true
    return
  }

  if (currentWordIndex.value >= currentSentenceWords.value.length) return

  const currentWord = currentSentenceWords.value[currentWordIndex.value]
  const currentInput = userInput.value[currentWordIndex.value] || ''

  if (event.key === 'Backspace') {
    if (currentInput.length > 0) {
      // 删除当前单词的最后一个字母
      userInput.value[currentWordIndex.value] = currentInput.slice(0, -1)
      playTypeSound() // 播放删除音效
    } else if (currentWordIndex.value > 0) {
      // 如果当前单词为空，则回到上一个单词
      currentWordIndex.value--
      // 如果上一个单词是标点符号，继续往前找
      while (
        currentWordIndex.value > 0 &&
        isPunctuation(currentSentenceWords.value[currentWordIndex.value])
      ) {
        currentWordIndex.value--
      }
      userInput.value.pop()
      playTypeSound() // 播放删除音效
    }
  } else if (event.key === ' ') {
    // 阻止空格键的默认行为（页面滚动）
    event.preventDefault()
    // 只有在当前单词有输入内容时才移动到下一个单词
    if (currentInput.length > 0 && currentWordIndex.value < currentSentenceWords.value.length - 1) {
      currentWordIndex.value++
      // 如果下一个是标点符号，继续往后找
      while (
        currentWordIndex.value < currentSentenceWords.value.length - 1 &&
        isPunctuation(currentSentenceWords.value[currentWordIndex.value])
      ) {
        currentWordIndex.value++
      }
      // 确保不会跳过最后一个单词
      if (currentWordIndex.value >= currentSentenceWords.value.length) {
        currentWordIndex.value = currentSentenceWords.value.length - 1
      }
      if (!userInput.value[currentWordIndex.value]) {
        userInput.value[currentWordIndex.value] = ''
      }
      playTypeSound() // 播放空格音效
    }
  } else if (event.key.length === 1 && !event.ctrlKey && !event.altKey && !event.metaKey) {
    // 将新字母添加到当前单词
    if (!userInput.value[currentWordIndex.value]) {
      userInput.value[currentWordIndex.value] = event.key
    } else {
      userInput.value[currentWordIndex.value] = currentInput + event.key
    }
    playTypeSound() // 播放打字音效
  }
}

// 开始听写
const startDictation = () => {
  // 使用正则表达式匹配句子，保留标点符号
  sentences.value =
    fileContent.value.match(/[^.!?。！？]+[.!?。！？]/g) ||
    [].map((s) => s.trim()).filter((s) => s.length > 0)

  currentSentenceIndex.value = 0
  currentWordIndex.value = 0
  userInput.value = []
  completedSentences.value = []
  isDictating.value = true

  // 朗读第一句
  speakCurrentSentence()
}

// 下一句
const nextSentence = () => {
  if (currentSentenceIndex.value < sentences.value.length - 1) {
    // 将当前完成的句子添加到已完成列表
    completedSentences.value.push(sentences.value[currentSentenceIndex.value])
    currentSentenceIndex.value++
    currentWordIndex.value = 0
    userInput.value = []
    speakCurrentSentence()
  } else {
    // 添加最后一句
    completedSentences.value.push(sentences.value[currentSentenceIndex.value])
    isDictating.value = false
    showCongratulations.value = true
  }
}

// 处理文件上传
const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  // 检查文件类型
  if (file.type !== 'text/plain' && !file.name.endsWith('.txt')) {
    ElMessage({
      message: '请选择文本文件(.txt)',
      type: 'warning',
      duration: 3000,
    })
    // 清空文件输入框
    event.target.value = ''
    return
  }

  const reader = new FileReader()
  reader.onload = (e) => {
    fileContent.value = e.target.result
  }
  reader.readAsText(file)
}

// 更新语音设置
const updateVoiceSettings = (settings) => {
  voiceSettings.value = settings
}

// 停止朗读
const stopSpeaking = () => {
  window.speechSynthesis.cancel()
  isSpeaking.value = false
}

// 开始朗读
const startSpeaking = () => {
  if (!fileContent.value) return

  const utterance = new SpeechSynthesisUtterance(fileContent.value)

  if (voiceSettings.value.voice) {
    utterance.voice = voiceSettings.value.voice
  }
  utterance.rate = voiceSettings.value.rate
  utterance.pitch = voiceSettings.value.pitch
  utterance.volume = voiceSettings.value.volume

  utterance.onstart = () => {
    isSpeaking.value = true
  }

  utterance.onend = () => {
    isSpeaking.value = false
  }

  utterance.onerror = () => {
    isSpeaking.value = false
  }

  window.speechSynthesis.speak(utterance)
}

// 切换朗读状态
const toggleSpeaking = () => {
  if (isSpeaking.value) {
    stopSpeaking()
  } else {
    startSpeaking()
  }
}

// 加载课程内容
const loadCourseContent = (course) => {
  fileContent.value = course.content
}

// 重新朗读已完成的句子
const replaySentence = (sentence) => {
  const utterance = new SpeechSynthesisUtterance(sentence)

  if (voiceSettings.value.voice) {
    utterance.voice = voiceSettings.value.voice
  }
  utterance.rate = voiceSettings.value.rate
  utterance.pitch = voiceSettings.value.pitch
  utterance.volume = voiceSettings.value.volume

  window.speechSynthesis.speak(utterance)
}

// 页面加载时
onMounted(() => {
  stopSpeaking()
  window.addEventListener('keydown', handleKeyPress)

  // 初始化默认语音
  const initDefaultVoice = () => {
    const voices = window.speechSynthesis.getVoices()
    const preferredVoices = [
      'Microsoft David Desktop',
      'Google US English',
      'Microsoft Zira Desktop',
      'Microsoft Mark Desktop',
    ]

    // 首先尝试找到首选语音
    let defaultVoice = voices.find(
      (voice) => preferredVoices.includes(voice.name) && voice.lang === 'en-US',
    )

    // 如果没有找到首选语音，则选择第一个美式英语语音
    if (!defaultVoice) {
      defaultVoice = voices.find((voice) => voice.lang === 'en-US') || voices[0]
    }

    if (defaultVoice) {
      voiceSettings.value.voice = defaultVoice
    }
  }

  // 立即尝试初始化
  initDefaultVoice()

  // 某些浏览器需要等待语音列表加载完成
  if (window.speechSynthesis.onvoiceschanged !== undefined) {
    window.speechSynthesis.onvoiceschanged = initDefaultVoice
  }
})

// 页面卸载前
onBeforeUnmount(() => {
  stopSpeaking()
  window.removeEventListener('keydown', handleKeyPress)
})

// 监听页面可见性变化
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    stopSpeaking()
  }
})

// 监听页面刷新/关闭
window.addEventListener('beforeunload', () => {
  stopSpeaking()
})
</script>

<style scoped>
.text-uploader {
  display: flex;
  max-width: none;
  margin: 0;
  padding: 0;
  height: 100vh;
}

.content-wrapper {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
}

.upload-section {
  margin-bottom: 20px;
  display: flex;
  justify-content: flex-start;
}

.upload-section--centered {
  justify-content: center;
  margin: 40px 0;
}

.file-input {
  display: none;
}

.upload-button {
  display: inline-block;
  padding: 12px 24px;
  background-color: #4caf50;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s;
  font-size: 16px;
  font-weight: 500;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.upload-button:hover {
  background-color: #45a049;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.content-section {
  margin-top: 20px;
  padding: 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.content-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
}

.button-group {
  display: flex;
  gap: 10px;
}

.speak-button,
.settings-button,
.dictate-button {
  padding: 8px 16px;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s;
}

.speak-button {
  background-color: #2196f3;
}

.speak-button:hover {
  background-color: #1976d2;
}

.speak-button--stop {
  background-color: #f44336;
}

.speak-button--stop:hover {
  background-color: #d32f2f;
}

.settings-button {
  padding: 8px;
  background-color: transparent;
  color: #666;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.settings-button:hover {
  background-color: #f0f0f0;
  color: #2196f3;
}

.settings-icon {
  width: 20px;
  height: 20px;
  fill: currentColor;
}

.dictate-button {
  background-color: #9c27b0;
}

.dictate-button:hover:not(:disabled) {
  background-color: #7b1fa2;
}

.dictate-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.dictation-section {
  margin: 20px 0;
  padding: 20px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.current-sentence {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 20px;
  min-height: 50px;
  padding: 10px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.word-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 0 6px;
  padding: 4px 0;
  position: relative;
}

.input-text {
  min-height: 24px;
  height: 24px;
  text-align: center;
  min-width: 24px;
  font-size: 24px;
  font-weight: 500;
  line-height: 1.2;
  font-family:
    'Segoe UI',
    -apple-system,
    BlinkMacSystemFont,
    'Helvetica Neue',
    Arial,
    sans-serif;
  letter-spacing: 0.5px;
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
}

.input-text.correct {
  color: #4caf50;
  font-weight: 600;
  letter-spacing: 0.3px;
}

.underline {
  height: 2px;
  background-color: #666;
  min-width: 24px;
  margin-top: 28px;
  position: relative;
}

.current-word .underline {
  background-color: #2196f3;
  height: 3px;
}

.sentence-controls {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  margin-top: 15px;
}

.repeat-button {
  padding: 8px;
  background-color: transparent;
  color: #666;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.repeat-button:hover:not(:disabled) {
  background-color: #f0f0f0;
  color: #2196f3;
}

.repeat-button:disabled {
  color: #ccc;
  cursor: not-allowed;
}

.repeat-icon {
  width: 20px;
  height: 20px;
  fill: currentColor;
}

.sentence-progress {
  text-align: center;
  color: #666;
  font-size: 14px;
  margin-top: 10px;
}

.hint-text {
  color: #2196f3;
  font-size: 12px;
  margin-top: 5px;
}

.file-content {
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family:
    'Segoe UI',
    -apple-system,
    BlinkMacSystemFont,
    'Helvetica Neue',
    Arial,
    sans-serif;
  background-color: #f8f9fa;
  padding: 20px;
  border-radius: 8px;
  max-height: 500px;
  overflow-y: auto;
  line-height: 1.6;
  font-size: 16px;
  color: #2c3e50;
  border: 1px solid #e9ecef;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.file-content::-webkit-scrollbar {
  width: 8px;
}

.file-content::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.file-content::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 4px;
}

.file-content::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}

.completed-sentences {
  margin: 20px 0;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.completed-sentences h4 {
  margin: 0 0 10px 0;
  color: #666;
}

.completed-sentence {
  padding: 8px;
  margin: 5px 0;
  background-color: white;
  border-radius: 4px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
}

.sentence-text {
  flex: 1;
  line-height: 1.4;
}

.replay-button {
  padding: 4px;
  background-color: transparent;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  color: #666;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.replay-button:hover {
  background-color: #f0f0f0;
  color: #2196f3;
}

.replay-icon {
  width: 20px;
  height: 20px;
  fill: currentColor;
}
</style>
