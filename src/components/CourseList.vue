<template>
  <div class="course-list">
    <h3>听力素材</h3>
    <div v-for="(courses, category) in courseList" :key="category" class="category">
      <div class="category-name">{{ category }}</div>
      <div class="course-items">
        <div
          v-for="course in courses"
          :key="course.path"
          class="course-item"
          :class="{ active: selectedCourse === course.path }"
          @click="selectCourse(course)"
        >
          {{ course.name }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const courseList = ref({})
const selectedCourse = ref('')

const emit = defineEmits(['select-course'])

const selectCourse = (course) => {
  selectedCourse.value = course.path
  emit('select-course', course)
}

// 使用 Vite 的 import.meta.glob 导入所有课程文件
const courseModules = import.meta.glob('../course/**/*.txt', {
  query: '?raw',
  import: 'default',
})

const fetchCourseList = async () => {
  try {
    const courses = {}
    console.log('Available course modules:', courseModules)

    // 处理所有课程文件
    for (const path in courseModules) {
      console.log('Processing path:', path)
      const parts = path.split('/')
      const category = parts[2] // ../course/初级/lesson1.txt -> 初级
      const fileName = parts[3] // ../course/初级/lesson1.txt -> lesson1.txt

      console.log('Category:', category, 'FileName:', fileName)

      if (!courses[category]) {
        courses[category] = []
      }

      courses[category].push({
        name: fileName.replace('.txt', ''),
        path: path,
        content: await courseModules[path](),
      })
    }

    // 对每个分类的课程按名称排序
    for (const category in courses) {
      courses[category].sort((a, b) => a.name.localeCompare(b.name))
    }

    console.log('Final course list:', courses)
    courseList.value = courses
  } catch (error) {
    console.error('Error loading course list:', error)
  }
}

onMounted(() => {
  fetchCourseList()
})
</script>

<style scoped>
.course-list {
  width: 280px;
  padding: 24px 16px;
  background-color: #ffffff;
  border-right: 1px solid #e0e0e0;
  height: 100vh;
  overflow-y: auto;
  box-shadow: 2px 0 8px rgba(0, 0, 0, 0.05);
}

.course-list h3 {
  margin: 0 0 24px 0;
  color: #1a1a1a;
  font-size: 20px;
  font-weight: 600;
  padding-bottom: 12px;
  border-bottom: 2px solid #2196f3;
}

.category {
  margin-bottom: 24px;
  background-color: #f8f9fa;
  border-radius: 8px;
  padding: 12px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.02);
}

.category-name {
  font-weight: 600;
  color: #2196f3;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid #e0e0e0;
  font-size: 16px;
  display: flex;
  align-items: center;
}

.category-name::before {
  content: '';
  display: inline-block;
  width: 4px;
  height: 16px;
  background-color: #2196f3;
  margin-right: 8px;
  border-radius: 2px;
}

.course-items {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.course-item {
  padding: 10px 14px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  color: #2c3e50;
  font-size: 14px;
  background-color: #ffffff;
  border: 1px solid #e0e0e0;
  display: flex;
  align-items: center;
}

.course-item::before {
  content: '📝';
  margin-right: 8px;
  font-size: 16px;
}

.course-item:hover {
  background-color: #f0f7ff;
  border-color: #2196f3;
  transform: translateX(4px);
}

.course-item.active {
  background-color: #2196f3;
  color: white;
  border-color: #2196f3;
  box-shadow: 0 2px 8px rgba(33, 150, 243, 0.3);
}

.course-item.active::before {
  content: '📖';
}

/* 自定义滚动条样式 */
.course-list::-webkit-scrollbar {
  width: 6px;
}

.course-list::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.course-list::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.course-list::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}
</style>
