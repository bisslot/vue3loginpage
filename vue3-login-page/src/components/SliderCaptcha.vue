<template>
  <div class="slider-captcha" :class="{ 'is-verified': verified }">
    <div class="captcha-track" ref="trackRef">
      <div class="captcha-bg">
        <span v-if="!verified">{{ message }}</span>
      </div>
      <div 
        class="captcha-slider" 
        :style="{ left: `${sliderLeft}px` }"
        @mousedown="startDrag"
        @touchstart="startDrag"
        :class="{ 'is-active': isDragging, 'is-verified': verified }"
      >
        <div class="slider-icon">
          <ArrowRightIcon v-if="!verified && !isDragging" />
          <ChevronRightIcon v-if="isDragging" />
          <CheckCircleIcon v-if="verified" />
        </div>
      </div>
      <div 
        class="captcha-progress" 
        :style="{ width: `${sliderLeft + sliderWidth/2}px` }"
      ></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { ArrowRightIcon, ChevronRightIcon, CheckCircleIcon } from 'tdesign-icons-vue-next'

const props = defineProps({
  successThreshold: {
    type: Number,
    default: 0.8 // 滑动到80%位置即视为成功
  }
})

const emit = defineEmits(['update:verified', 'success', 'reset'])

// 状态变量
const verified = ref(false)
const isDragging = ref(false)
const sliderLeft = ref(0)
const sliderWidth = ref(40)
const trackWidth = ref(0)
const trackRef = ref<HTMLElement | null>(null)
const startX = ref(0)

// 计算属性
const message = computed(() => {
  if (verified.value) {
    return '验证成功'
  }
  return '请向右拖动滑块完成验证'
})

// 开始拖动
const startDrag = (e: MouseEvent | TouchEvent) => {
  if (verified.value) return
  
  isDragging.value = true
  document.body.classList.add('no-select') // 防止拖动时选中文本
  
  // 记录起始位置
  if ('touches' in e) {
    startX.value = e.touches[0].clientX
  } else {
    startX.value = e.clientX
  }
  
  // 添加事件监听
  document.addEventListener('mousemove', onDrag)
  document.addEventListener('touchmove', onDrag)
  document.addEventListener('mouseup', stopDrag)
  document.addEventListener('touchend', stopDrag)
}

// 拖动中
const onDrag = (e: MouseEvent | TouchEvent) => {
  if (!isDragging.value) return
  
  let currentX: number
  if ('touches' in e) {
    currentX = e.touches[0].clientX
  } else {
    currentX = e.clientX
  }
  
  // 计算移动距离
  let moveX = currentX - startX.value
  
  // 限制在轨道范围内
  const maxLeft = trackWidth.value - sliderWidth.value
  sliderLeft.value = Math.max(0, Math.min(maxLeft, moveX))
}

// 停止拖动
const stopDrag = () => {
  if (!isDragging.value) return
  
  isDragging.value = false
  document.body.classList.remove('no-select')
  
  // 移除事件监听
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('touchmove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
  document.removeEventListener('touchend', stopDrag)
  
  // 检查是否验证成功
  const maxLeft = trackWidth.value - sliderWidth.value
  const successPosition = maxLeft * props.successThreshold
  
  if (sliderLeft.value >= successPosition) {
    // 验证成功
    verified.value = true
    sliderLeft.value = maxLeft // 自动完成剩余距离
    emit('update:verified', true)
    emit('success')
  } else {
    // 验证失败，重置
    resetSlider()
  }
}

// 重置滑块
const resetSlider = () => {
  // 添加动画效果
  const slider = document.querySelector('.captcha-slider') as HTMLElement
  if (slider) {
    slider.style.transition = 'left 0.3s ease-out'
    sliderLeft.value = 0
    
    setTimeout(() => {
      if (slider) {
        slider.style.transition = ''
      }
    }, 300)
  }
  
  verified.value = false
  emit('update:verified', false)
  emit('reset')
}

// 初始化
onMounted(() => {
  if (trackRef.value) {
    trackWidth.value = trackRef.value.clientWidth
    
    // 监听窗口大小变化，更新轨道宽度
    window.addEventListener('resize', updateTrackWidth)
  }
})

// 更新轨道宽度
const updateTrackWidth = () => {
  if (trackRef.value) {
    trackWidth.value = trackRef.value.clientWidth
    
    // 如果当前滑块位置超出新的轨道范围，重置位置
    const maxLeft = trackWidth.value - sliderWidth.value
    if (sliderLeft.value > maxLeft) {
      sliderLeft.value = verified.value ? maxLeft : 0
    }
  }
}

// 组件卸载时清理
onUnmounted(() => {
  document.removeEventListener('mousemove', onDrag)
  document.removeEventListener('touchmove', onDrag)
  document.removeEventListener('mouseup', stopDrag)
  document.removeEventListener('touchend', stopDrag)
  window.removeEventListener('resize', updateTrackWidth)
  document.body.classList.remove('no-select')
})

// 暴露方法
defineExpose({
  reset: resetSlider
})
</script>

<style scoped>
.slider-captcha {
  width: 100%;
  height: 44px;
  position: relative;
  border-radius: 6px;
  overflow: hidden;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.slider-captcha:hover {
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

.captcha-track {
  width: 100%;
  height: 100%;
  background-color: #f5f5f5;
  position: relative;
  border-radius: 6px;
  border: 1px solid var(--td-border-level-1-color);
  overflow: hidden;
}

.captcha-bg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--td-text-color-secondary);
  font-size: 14px;
  z-index: 1;
}

.captcha-progress {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background-color: rgba(0, 82, 217, 0.1);
  z-index: 2;
  transition: width 0.1s ease;
}

.captcha-slider {
  position: absolute;
  top: 0;
  left: 0;
  width: 44px;
  height: 100%;
  background-color: var(--td-brand-color);
  border-radius: 6px;
  z-index: 3;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  box-shadow: 0 2px 6px rgba(0, 82, 217, 0.4);
  transition: background-color 0.3s ease;
}

.captcha-slider.is-active {
  background-color: var(--td-brand-color-hover);
  box-shadow: 0 2px 10px rgba(0, 82, 217, 0.6);
}

.captcha-slider.is-verified {
  background-color: var(--td-success-color);
  box-shadow: 0 2px 10px rgba(0, 168, 112, 0.4);
}

.slider-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  animation: pulse 1.5s infinite ease-in-out;
}

.is-active .slider-icon {
  animation: none;
}

.is-verified .slider-icon {
  animation: success-pulse 0.5s ease-in-out;
}

.slider-captcha.is-verified .captcha-progress {
  background-color: rgba(0, 168, 112, 0.1);
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.1);
  }
  100% {
    transform: scale(1);
  }
}

@keyframes success-pulse {
  0% {
    transform: scale(0.8);
  }
  50% {
    transform: scale(1.2);
  }
  100% {
    transform: scale(1);
  }
}

/* 防止文本选中 */
:global(.no-select) {
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}
</style>