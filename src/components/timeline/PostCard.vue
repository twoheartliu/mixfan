<script setup>
import MastodonPostContent from './MastodonPostContent.vue'
import FanfouPostContent from './FanfouPostContent.vue'
import CommentSection from './CommentSection.vue'
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { safeGet } from '@/utils/helpers'

const props = defineProps({
  post: {
    type: Object,
    required: true
  }
})

// 图片预览相关状态
const showImagePreview = ref(false)
const previewImageUrl = ref('')
const scrollPosition = ref(0)

// 评论展开状态
const showComments = ref(false)

// 切换评论展开
const toggleComments = () => {
  showComments.value = !showComments.value
}

// 获取原帖作者信息（用于回复）
const replyToUserId = computed(() => {
  if (props.post.source === 'fanfou') {
    return safeGet(props.post, 'user.id', '')
  }
  return ''
})

const replyToUsername = computed(() => {
  if (props.post.source === 'mastodon') {
    return safeGet(props.post, 'reblog.account.acct', safeGet(props.post, 'account.acct', ''))
  } else {
    return safeGet(props.post, 'user.screen_name', '')
  }
})

// 打开图片预览
const openImagePreview = (imageUrl) => {
  previewImageUrl.value = imageUrl
  showImagePreview.value = true
  // 保存当前滚动位置
  scrollPosition.value = window.pageYOffset || document.documentElement.scrollTop
  // 禁止背景滚动
  document.body.style.overflow = 'hidden'
  document.body.style.position = 'fixed'
  document.body.style.top = `-${scrollPosition.value}px`
  document.body.style.width = '100%'
}

// 关闭图片预览
const closeImagePreview = () => {
  showImagePreview.value = false
  // 恢复背景滚动
  document.body.style.overflow = ''
  document.body.style.position = ''
  document.body.style.width = ''
  document.body.style.top = ''
  // 恢复滚动位置
  window.scrollTo(0, scrollPosition.value)
}

// 处理键盘事件
const handleKeyDown = (event) => {
  if (event.key === 'Escape' && showImagePreview.value) {
    closeImagePreview()
  }
}

// 添加和移除键盘事件监听器
onMounted(() => {
  document.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeyDown)
})
</script>

<template>
  <article :class="[
    'post-card relative rounded-lg border border-gray-700 p-4',
    post.source === 'mastodon' ? 'source-mastodon' : 'source-fanfou'
  ]">
    <div class="source-indicator"></div>
    <div class="flex items-start space-x-3">
      <!-- 头像 -->
      <img v-if="post.source === 'mastodon'" :src="post.reblog?.account?.avatar || post.account?.avatar || ''"
        class="w-10 h-10 rounded-full cursor-pointer" alt="avatar"
        @click="openImagePreview(post.reblog?.account?.avatar || post.account?.avatar || '')">
      <img v-else :src="post.user?.profile_image_url || ''" class="w-10 h-10 rounded-full cursor-pointer" alt="avatar"
        @click="openImagePreview(post.user?.profile_image_url || '')">

      <div class="flex-1 min-w-0">
        <!-- 根据来源显示不同的内容组件 -->
        <MastodonPostContent v-if="post.source === 'mastodon'" :post="post" @preview-image="openImagePreview"
          @toggle-comments="toggleComments" @reblog-success="$emit('reblog-success')" />
        <FanfouPostContent v-else :post="post" @preview-image="openImagePreview" @toggle-comments="toggleComments"
          @repost-success="$emit('repost-success')" />

        <!-- 评论区 -->
        <CommentSection v-if="showComments" :post-id="post.id" :source="post.source" :reply-to-user-id="replyToUserId"
          :reply-to-username="replyToUsername" />
      </div>
    </div>
  </article>

  <!-- 图片预览模态框 - 不使用 Teleport，直接放在 body 下 -->
  <div v-if="showImagePreview" class="modal-overlay" @click="closeImagePreview">
    <div class="modal-content" @click.stop>
      <img :src="previewImageUrl" alt="Preview" class="modal-image">
      <button class="modal-close" @click="closeImagePreview">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none"
          stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <line x1="18" y1="6" x2="6" y2="18"></line>
          <line x1="6" y1="6" x2="18" y2="18"></line>
        </svg>
      </button>
    </div>
  </div>
</template>

<style>
/* 全局样式，确保模态框覆盖所有内容 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.modal-content {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 10px;
  box-sizing: border-box;
}

.modal-image {
  max-width: 100%;
  max-height: 90vh;
  object-fit: contain;
}

.modal-close {
  position: absolute;
  top: 10px;
  right: 10px;
  background-color: rgba(50, 50, 50, 0.7);
  color: white;
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
}

.cursor-pointer {
  cursor: pointer;
}
</style>