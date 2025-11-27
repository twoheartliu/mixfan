<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'
import LoadingSpinner from '@/components/common/LoadingSpinner.vue'

const props = defineProps({
  onRefresh: {
    type: Function,
    required: true
  },
  threshold: {
    type: Number,
    default: 60
  },
  // 新增：外部传入的加载状态
  isLoading: {
    type: Boolean,
    default: false
  }
})

const containerRef = ref(null)
const pullDistance = ref(0)
const isRefreshing = ref(false)
const isPulling = ref(false)
let startY = 0
let isDragging = false

// 监听外部loading状态变化
watch(() => props.isLoading, (newValue) => {
  isRefreshing.value = newValue
  if (!newValue) {
    // 当外部loading结束时，重置pullDistance
    setTimeout(() => {
      pullDistance.value = 0
    }, 500)
  }
})

// Check if an element or its parents are scrollable and not at top
const isScrollable = (el) => {
  let current = el;
  while (current && current !== containerRef.value) {
    // Prevent accessing null style or non-element nodes
    if (!current || current.nodeType !== 1) {
      current = current ? current.parentElement : null;
      continue;
    }

    const style = window.getComputedStyle(current);
    // Check for scrollable overflow
    if (['auto', 'scroll'].includes(style.overflowY)) {
      // If it's scrolled down, it's scrollable
      if (current.scrollTop > 0) return true;
    }
    current = current.parentElement;
  }
  return false;
}

// 统一处理开始事件
const handleStart = (y, target) => {
  // Only activate if we are at the very top of the page
  if (window.scrollY <= 5 && !isRefreshing.value) {
    // If touching a scrollable inner element that isn't at top, ignore
    if (isScrollable(target)) return;

    startY = y
    isDragging = true
  }
}

// 统一处理移动事件
const handleMove = (y, e) => {
  if (!isDragging) return
  if (isRefreshing.value) return

  const diff = y - startY

  // Only activate if pulling down and we are roughly at top
  if (diff > 0 && window.scrollY <= 5) {
    if (e.cancelable) {
      e.preventDefault() // Prevent native scroll behavior
    }
    isPulling.value = true
    // Resistance: pull harder, move less
    pullDistance.value = diff * 0.4
  } else {
    isPulling.value = false
    pullDistance.value = 0
    // If we scroll up or down past start, reset drag to avoid sticking
    if (diff < 0) isDragging = false
  }
}

// 统一处理结束事件
const handleEnd = async () => {
  isDragging = false
  if (!isPulling.value) return

  if (pullDistance.value >= props.threshold) {
    // Trigger Refresh
    isPulling.value = false
    isRefreshing.value = true
    pullDistance.value = props.threshold // Snap to loading position

    try {
      await props.onRefresh()
    } finally {
      setTimeout(() => {
        isRefreshing.value = false
        pullDistance.value = 0
      }, 500)
    }
  } else {
    // Cancel pull
    isPulling.value = false
    pullDistance.value = 0
  }
}

// 执行刷新动作
const executeRefresh = async () => {
  isPulling.value = false
  isRefreshing.value = true
  pullDistance.value = props.threshold

  try {
    await props.onRefresh()
  } finally {
    // 不在这里重置isRefreshing，而是依赖外部传入的isLoading状态
    if (!props.isLoading) {
      setTimeout(() => {
        isRefreshing.value = false
        pullDistance.value = 0
      }, 500)
    }
  }
}

// Touch Events
const handleTouchStart = (e) => handleStart(e.touches[0].clientY, e.target)
const handleTouchMove = (e) => handleMove(e.touches[0].clientY, e)
const handleTouchEnd = () => handleEnd()

// Mouse Events
const handleMouseDown = (e) => handleStart(e.clientY, e.target)
const handleMouseMove = (e) => {
  if (isDragging) {
    // For mouse, we need to prevent default to stop text selection etc.
    // e.preventDefault() 
    handleMove(e.clientY, e)
  }
}
const handleMouseUp = () => handleEnd()
const handleMouseLeave = () => handleEnd() // Also cancel if mouse leaves the area

// Trackpad / Wheel Events
let wheelTimeout = null
let lastWheelTime = 0
let accumulatedDelta = 0
let lastWheelEventTime = 0
let inactivityTimer = null

const handleWheel = (e) => {
  if (isRefreshing.value) return

  // 如果不在顶部，直接退出
  if (window.scrollY > 0) {
    pullDistance.value = 0
    isPulling.value = false
    return
  }

  // 如果内部可滚动，不处理
  if (isScrollable(e.target)) return;

  // 更新最后一次wheel事件的时间
  lastWheelEventTime = Date.now()

  // 清除之前的不活动定时器
  if (inactivityTimer) {
    clearTimeout(inactivityTimer)
  }

  // 设置新的不活动检测定时器 - 更短的时间
  inactivityTimer = setTimeout(() => {
    // 如果已经超过阈值，立即触发刷新
    if (pullDistance.value >= props.threshold) {
      executeRefresh()
    } else {
      // 否则重置
      isPulling.value = false
      pullDistance.value = 0
    }
    accumulatedDelta = 0
  }, 50) // 减少到50ms，更快地检测手势结束

  // 当前时间
  const now = Date.now()

  // 如果是向下滚动 (deltaY < 0)
  if (e.deltaY < 0) {
    // 节流处理：限制事件处理频率，减少性能开销
    if (now - lastWheelTime < 16) { // 约60fps
      // 只累积值，不更新UI
      accumulatedDelta += (-e.deltaY * 0.3) // 降低系数，减少灵敏度
      return
    }

    // 更新最后处理时间
    lastWheelTime = now

    // 只在必要时阻止默认行为
    if (pullDistance.value > 0 && e.cancelable) {
      e.preventDefault()
    }

    isPulling.value = true

    // 累积距离，但使用更低的系数减少灵敏度
    const newDistance = pullDistance.value + accumulatedDelta + (-e.deltaY * 0.3)
    accumulatedDelta = 0 // 重置累积值

    // 限制最大拉动距离
    pullDistance.value = Math.min(newDistance, props.threshold * 2)

    // 如果已经超过阈值并且拉动幅度很大，立即触发刷新
    // 这可以处理快速大幅度拉动的情况
    if (pullDistance.value >= props.threshold && -e.deltaY > 50) {
      clearTimeout(inactivityTimer)
      executeRefresh()
    }
  } else {
    // 向上滚动 (deltaY > 0)，重置
    if (pullDistance.value > 0) {
      pullDistance.value = Math.max(0, pullDistance.value - (e.deltaY * 0.5))
      if (pullDistance.value === 0) {
        isPulling.value = false
      }

      // 只在拉动状态时阻止默认行为
      if (e.cancelable && isPulling.value) {
        e.preventDefault()
      }
    }
  }
}

onMounted(() => {
  const el = containerRef.value
  if (el) {
    // Touch
    el.addEventListener('touchstart', handleTouchStart, { passive: true })
    el.addEventListener('touchmove', handleTouchMove, { passive: false })
    el.addEventListener('touchend', handleTouchEnd)

    // Mouse
    el.addEventListener('mousedown', handleMouseDown)
    el.addEventListener('mousemove', handleMouseMove)
    el.addEventListener('mouseup', handleMouseUp)
    el.addEventListener('mouseleave', handleMouseLeave)

    // Wheel (Trackpad) - 使用passive为true，只在必要时在函数内部preventDefault
    el.addEventListener('wheel', handleWheel, { passive: false })
  }
})

onUnmounted(() => {
  const el = containerRef.value
  if (el) {
    el.removeEventListener('touchstart', handleTouchStart)
    el.removeEventListener('touchmove', handleTouchMove)
    el.removeEventListener('touchend', handleTouchEnd)

    el.removeEventListener('mousedown', handleMouseDown)
    el.removeEventListener('mousemove', handleMouseMove)
    el.removeEventListener('mouseup', handleMouseUp)
    el.removeEventListener('mouseleave', handleMouseLeave)

    el.removeEventListener('wheel', handleWheel)
  }

  // 清理定时器
  if (wheelTimeout) {
    clearTimeout(wheelTimeout)
  }
  if (inactivityTimer) {
    clearTimeout(inactivityTimer)
  }
})
</script>

<template>
  <div ref="containerRef" class="pull-to-refresh-container relative select-none">
    <!-- Indicator -->
    <div class="refresh-indicator absolute left-0 w-full flex justify-center items-center pointer-events-none" :style="{
      top: `-${props.threshold}px`,
      height: `${props.threshold}px`,
      transform: `translateY(${pullDistance}px)`,
      opacity: pullDistance > 0 ? 1 : 0,
      zIndex: 0
    }">
      <div class="flex items-center space-x-2 text-gray-500 dark:text-gray-400">
        <LoadingSpinner v-if="isRefreshing" size="sm" />
        <span v-else-if="pullDistance > props.threshold" class="transform rotate-180 transition-transform duration-200">
          <i class="fas fa-arrow-up"></i>
        </span>
        <span v-else class="transition-transform duration-200">
          <i class="fas fa-arrow-down"></i>
        </span>

        <span class="text-sm font-medium">
          {{ isRefreshing ? '刷新中...' : (pullDistance > props.threshold ? '释放刷新' : '下拉刷新') }}
        </span>
      </div>
    </div>

    <!-- Content -->
    <div class="content-wrapper transition-transform relative bg-transparent"
      :class="{ 'duration-300 ease-out': !isPulling && !isDragging }" :style="{
        transform: `translateY(${pullDistance}px)`,
        zIndex: 10
      }">
      <slot></slot>
    </div>
  </div>
</template>