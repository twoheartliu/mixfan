<script setup>
import { onMounted, ref, onBeforeUnmount } from 'vue'
import AppHeader from '@/components/layout/AppHeader.vue'
import AppFooter from '@/components/layout/AppFooter.vue'
import ComposeBox from '@/components/compose/ComposeBox.vue'
import TimelineFilter from '@/components/timeline/TimelineFilter.vue'
import UserProfile from '@/components/user/UserProfile.vue'
import PostCard from '@/components/timeline/PostCard.vue'
import LoadMore from '@/components/timeline/LoadMore.vue'
import LoadingSpinner from '@/components/common/LoadingSpinner.vue'
import PullToRefresh from '@/components/common/PullToRefresh.vue'

// 导入组合式函数
import useAuth from '@/composables/useAuth.js'
import useTimeline from '@/composables/useTimeline.js'
import useCompose from '@/composables/useCompose.js'

// 导入样式
import '@/assets/styles/base.css'

// 使用组合式函数
const { ffStore, mastoStore, instanceURL, logoutff, logoutNofan } = useAuth()
const {
  timelineLoading,
  isLoading,      // 新增：获取统一的加载状态
  mergedTimeline,
  currentFilter,
  hasMorePosts,
  loadTimelines,
  loadMore,
  refreshTimelines,
  setFilter
} = useTimeline()
const { sendMessage } = useCompose()

// 控制发布框弹窗显示状态
const showComposeModal = ref(false)
// 控制悬浮按钮显示状态
const showFloatingButton = ref(false)
// 控制刷新按钮显示状态
const showRefreshButton = ref(true)
// 用于追踪滚动位置
const lastScrollPosition = ref(0)
// 发布框区域的引用
const composeBoxRef = ref(null)

// 初始化
onMounted(async () => {
  ffStore.getUserInfo()
  mastoStore.getUserInfo(instanceURL.value)

  // 加载 Mastodon 自定义表情
  mastoStore.fetchCustomEmojis(instanceURL.value)

  // 加载时间线
  loadTimelines()

  // 添加滚动监听
  window.addEventListener('scroll', handleScroll)
})

onBeforeUnmount(() => {
  // 移除滚动监听
  window.removeEventListener('scroll', handleScroll)
})

// 处理滚动事件
function handleScroll() {
  if (composeBoxRef.value) {
    const composeBoxBottom = composeBoxRef.value.getBoundingClientRect().bottom
    // 当发布框底部不在可视区域内时显示悬浮按钮
    showFloatingButton.value = composeBoxBottom < 0
  }

  lastScrollPosition.value = window.scrollY
}

// 处理筛选器变更
function handleFilterChange(filter) {
  setFilter(filter)
}

// 处理发送消息后刷新时间线
async function handleSendMessage() {
  const success = await sendMessage()
  if (success) {
    loadTimelines()
    // 发送成功后关闭弹窗
    showComposeModal.value = false
  }
}

// 打开发布框弹窗
function openComposeModal() {
  showComposeModal.value = true
}

// 关闭发布框弹窗
function closeComposeModal() {
  showComposeModal.value = false
}

// 滚动到顶部
function scrollToTop() {
  window.scrollTo({
    top: 0,
    behavior: 'smooth'
  })
}


// 节流函数
function throttle(fn, delay) {
  let lastCall = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastCall >= delay) {
      lastCall = now;
      return fn.apply(this, args);
    }
  };
}

// 刷新时间线并回到顶部
const refreshAndScrollToTop = throttle(async () => {
  // 如果已经在刷新中，不执行
  if (isLoading.value) return;

  try {
    scrollToTop();
    await refreshTimelines();
  } catch (error) {
    console.error('刷新失败:', error);
  }
}, 3000); // 3秒内只能执行一次

</script>

<template>
  <!-- 顶部导航栏 -->
  <AppHeader />

  <!-- 主内容区 -->
  <main class="container mx-auto px-4 py-4 min-h-[calc(100vh-170px)]">
    <!-- 传入刷新状态给PullToRefresh组件 -->
    <PullToRefresh :on-refresh="refreshTimelines" :is-loading="isLoading">
      <!-- 普通发布框 -->
      <div ref="composeBoxRef" class="mb-4">
        <ComposeBox @send-message="handleSendMessage" />

        <!-- 筛选器 -->
        <TimelineFilter :current-filter="currentFilter" @update:filter="handleFilterChange" class="mt-4" />
      </div>

      <!-- 提示信息 -->
      <div v-if="!ffStore.userInfo.id || !mastoStore.userInfo.id" class="mt-8 p-4 bg-opacity-10 bg-blue-500 rounded-lg">
        <h2 class="text-lg font-bold mb-2">欢迎使用拌饭 MixFan</h2>
        <ul class="list-disc list-inside space-y-1 text-sm">
          <li>请点击连接 NOFAN 实例账号和饭否账号</li>
          <li>消息将同步发送到长毛象 NOFAN 实例和饭否</li>
          <li>饭否先审后发，人工审核后才会在 TL 显示</li>
        </ul>
      </div>

      <!-- 账号信息展示 -->
      <div class="mt-4 grid grid-cols-1 md:grid-cols-2 gap-4">
        <UserProfile v-if="mastoStore.userInfo.id" :user-info="mastoStore.userInfo" type="mastodon"
          :on-logout="logoutNofan" />

        <UserProfile v-if="ffStore.userInfo.id" :user-info="ffStore.userInfo" type="fanfou" :on-logout="logoutff" />
      </div>

      <!-- 统一时间轴 -->
      <div class="mt-6 space-y-4">
        <!-- 加载中提示 - 只在没有数据时显示，避免与下拉刷新重复 -->
        <div v-if="timelineLoading && isLoading && mergedTimeline.length === 0" class="text-center py-8">
          <LoadingSpinner size="md" text="加载中..." center />
        </div>

        <!-- 没有数据提示 -->
        <div v-else-if="mergedTimeline.length === 0" class="text-center py-8 text-gray-500">
          <i class="fas fa-inbox text-4xl mb-2"></i>
          <p>暂无数据，请先登录您的账号</p>
        </div>

        <!-- 帖子列表 -->
        <PostCard v-for="post in mergedTimeline" :key="post.id" :post="post" @repost-success="refreshTimelines" />

        <!-- 加载更多 - 使用timelineLoading而非isLoading，避免与下拉刷新冲突 -->
        <LoadMore v-if="mergedTimeline.length > 0" :loading="timelineLoading" :has-more="hasMorePosts"
          @load-more="loadMore" />
      </div>
    </PullToRefresh>
  </main>

  <!-- 悬浮按钮组 -->
  <div class="fixed bottom-4 right-3 md:bottom-6 md:right-6 flex flex-col space-y-3 z-40">
    <!-- 回到顶部按钮 - 仅在滚动时显示 -->
    <transition name="fade">
      <button v-if="showFloatingButton" @click="scrollToTop"
        class="bg-gray-200 hover:bg-gray-300 dark:bg-gray-700 dark:hover:bg-gray-600 w-12 h-12 md:w-14 md:h-14 rounded-full flex items-center justify-center shadow-lg transition-colors"
        title="回到顶部">
        <i class="fas fa-arrow-up text-gray-600 dark:text-gray-300 text-base md:text-lg"></i>
      </button>
    </transition>

    <!-- 刷新按钮 -->
    <transition name="fade">
      <button @click="refreshAndScrollToTop"
        class="bg-gray-200 hover:bg-gray-300 dark:bg-gray-700 dark:hover:bg-gray-600 w-12 h-12 md:w-14 md:h-14 rounded-full flex items-center justify-center shadow-lg transition-colors"
        title="刷新时间线" :disabled="isLoading">
        <!-- 当刷新中时显示 LoadingSpinner，否则显示刷新图标 -->
        <LoadingSpinner v-if="isLoading" size="sm" />
        <i v-else class="fas fa-rotate-right text-gray-600 dark:text-gray-300 text-base md:text-lg"></i>
      </button>
    </transition>

    <!-- 发布按钮 - 仅在滚动时显示 -->
    <transition name="fade">
      <button v-if="showFloatingButton" @click="openComposeModal"
        class="bg-blue-500 hover:bg-blue-600 text-white w-12 h-12 md:w-14 md:h-14 rounded-full flex items-center justify-center shadow-lg transition-colors"
        title="发布新消息">
        <i class="fas fa-pen text-base md:text-lg"></i>
      </button>
    </transition>
  </div>

  <!-- 发布框弹窗 -->
  <div v-if="showComposeModal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
    <div class="bg-white dark:bg-gray-800 rounded-lg shadow-xl w-full max-w-xl p-4">
      <div class="flex justify-between items-center mb-4">
        <h2 class="text-lg font-bold dark:text-white">发布新消息</h2>
        <button @click="closeComposeModal"
          class="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200">
          <i class="fas fa-times"></i>
        </button>
      </div>
      <ComposeBox @send-message="handleSendMessage" />
    </div>
  </div>

  <!-- 页脚 -->
  <AppFooter />
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* 禁用按钮样式 */
button:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}
</style>