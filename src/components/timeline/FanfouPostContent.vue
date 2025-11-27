<script setup>
import { ref, computed } from 'vue'
import { safeGet } from '@/utils/helpers'
import { formatTimeAgo } from '@/utils/formatters'
import MediaAttachment from './MediaAttachment.vue'
import { ff } from '@/utils/fanfou'

const props = defineProps({
  post: {
    type: Object,
    required: true
  }
})

// 定义事件，用于向父组件传递预览图片的请求
const emit = defineEmits(['preview-image', 'toggle-comments', 'repost-success'])

// 获取媒体附件
function getMediaAttachments(post) {
  return post.photo ? [post.photo] : []
}

// 处理图片预览
const handlePreviewImage = (imageUrl) => {
  emit('preview-image', imageUrl)
}

// 转发相关状态
const showRepostModal = ref(false)
const repostComment = ref('') // 用户的评论
const isReposting = ref(false)

// 获取原始消息 ID（如果是转发消息，获取原始消息 ID）
function getOriginalPostId() {
  // 如果已经是转发，获取原始消息 ID
  if (props.post.repost_status_id) {
    return props.post.repost_status_id
  }
  // 否则使用当前消息 ID（去掉 ff_ 前缀）
  return props.post.id.replace(/^ff_/, '')
}

// 获取原始消息作者（如果是转发，获取原始作者）
function getOriginalAuthor() {
  if (props.post.repost_screen_name) {
    return props.post.repost_screen_name
  }
  return safeGet(props.post, 'user.name', '')
}

// 获取原始消息内容
function getOriginalText() {
  // 如果是转发消息，原文可能在 repost_status 中
  // 否则使用当前消息的 text
  return props.post.text || ''
}

// 计算最终的转发内容
function buildRepostContent() {
  const comment = repostComment.value.trim()
  const author = getOriginalAuthor()
  const originalText = getOriginalText()

  // 格式：评论 转@用户 原文
  if (comment) {
    return `${comment} 转@${author} ${originalText}`
  }
  return `转@${author} ${originalText}`
}

// 计算当前内容长度
const currentLength = computed(() => {
  return buildRepostContent().length
})

// 打开转发弹窗
function openRepostModal() {
  repostComment.value = ''
  showRepostModal.value = true
}

// 关闭转发弹窗
function closeRepostModal() {
  showRepostModal.value = false
  repostComment.value = ''
}

// 执行转发
async function handleRepost() {
  if (isReposting.value) return

  const finalContent = buildRepostContent()
  if (finalContent.length > 140) {
    alert('转发内容超过140字限制')
    return
  }

  isReposting.value = true
  try {
    const originalId = getOriginalPostId()

    await ff.post('/statuses/update', {
      status: finalContent,
      repost_status_id: originalId
    })

    closeRepostModal()
    emit('repost-success')
  } catch (error) {
    console.error('转发失败:', error)
    alert('转发失败，请重试')
  } finally {
    isReposting.value = false
  }
}
</script>

<template>
  <div>
    <!-- 用户信息 -->
    <div class="flex items-center space-x-2 flex-wrap">
      <span class="font-bold truncate">
        {{ safeGet(post, 'user.name', '未知用户') }}
      </span>
      <span class="text-sm text-gray-500">
        @{{ safeGet(post, 'user.screen_name', '') }}
      </span>
      <span class="text-sm text-gray-500">·</span>
      <span class="text-sm text-gray-500">
        {{ formatTimeAgo(post.created_at) }}
      </span>

      <!-- 受保护账号标记 -->
      <span v-if="safeGet(post, 'user.protected', false)" class="text-sm text-gray-500" title="受保护的账号">
        <i class="fas fa-lock text-xs"></i>
      </span>

      <Icon name="fanfou" class="text-blue-400 ml-1" />

      <!-- 转发标记 -->
      <span v-if="post.repost_status_id" class="ml-2 text-sm text-gray-500">
        <i class="fas fa-retweet mr-1"></i> 转发
      </span>
    </div>

    <!-- 发布应用信息 -->
    <div class="text-xs text-gray-500 mt-1">
      通过 <span v-html="post.application || '网页'"></span>
    </div>

    <!-- 转发者信息 -->
    <div v-if="post.repost_status_id && post.repost_screen_name" class="mt-1 text-sm text-gray-500">
      转自 <span class="font-medium">@{{ post.repost_screen_name }}</span>
    </div>

    <!-- 消息内容 -->
    <p class="mt-2 text-sm md:text-base">
      {{ post.text }}
    </p>

    <!-- 媒体附件 -->
    <MediaAttachment :media="getMediaAttachments(post)" @preview-image="handlePreviewImage" />

    <!-- 交互按钮 -->
    <div class="flex flex-wrap gap-4 mt-3 text-sm">
      <button @click="emit('toggle-comments')" class="flex items-center space-x-2 hover:text-blue-400 transition-colors">
        <i class="far fa-comment"></i>
      </button>
      <button @click="openRepostModal" class="flex items-center space-x-2 hover:text-green-400 transition-colors">
        <i class="fas fa-retweet"></i>
      </button>
      <button class="flex items-center space-x-2 hover:text-red-400 transition-colors">
        <i class="far fa-heart"></i>
      </button>
    </div>

    <!-- 转发弹窗 -->
    <Teleport to="body">
      <div v-if="showRepostModal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
        <div class="bg-white dark:bg-gray-800 rounded-lg shadow-xl w-full max-w-lg">
          <!-- 弹窗头部 -->
          <div class="flex justify-between items-center p-4 border-b border-gray-200 dark:border-gray-700">
            <h2 class="text-lg font-bold dark:text-white">转发到饭否</h2>
            <button @click="closeRepostModal" class="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200">
              <i class="fas fa-times"></i>
            </button>
          </div>

          <!-- 弹窗内容 -->
          <div class="p-4">
            <!-- 评论输入框 -->
            <textarea
              v-model="repostComment"
              class="w-full bg-gray-100 dark:bg-gray-700 rounded-lg p-3 text-sm resize-none focus:outline-none focus:ring-2 focus:ring-blue-500 dark:text-white"
              placeholder="添加评论（可选）..."
              rows="2"
              :disabled="isReposting"
            ></textarea>

            <!-- 原消息预览 -->
            <div class="mt-3 p-3 bg-gray-100 dark:bg-gray-700 rounded-lg border-l-2 border-blue-500">
              <div class="text-xs text-gray-500 mb-1">原消息</div>
              <div class="flex items-start space-x-2">
                <img
                  v-if="safeGet(post, 'user.profile_image_url')"
                  :src="safeGet(post, 'user.profile_image_url')"
                  class="w-6 h-6 rounded-full flex-shrink-0"
                />
                <div class="flex-1 min-w-0">
                  <div class="font-medium text-sm truncate">@{{ getOriginalAuthor() }}</div>
                  <div class="text-sm text-gray-600 dark:text-gray-300 line-clamp-3">{{ getOriginalText() }}</div>
                </div>
              </div>
            </div>

            <!-- 最终转发内容预览 -->
            <div class="mt-3 p-3 bg-gray-50 dark:bg-gray-900 rounded-lg">
              <div class="text-xs text-gray-500 mb-1">转发内容预览</div>
              <div class="text-sm text-gray-700 dark:text-gray-300 break-all">{{ buildRepostContent() }}</div>
            </div>

            <!-- 字数统计 -->
            <div class="text-right text-xs mt-2" :class="currentLength > 140 ? 'text-red-500' : 'text-gray-500'">
              {{ currentLength }}/140
            </div>
          </div>

          <!-- 弹窗底部 -->
          <div class="flex justify-end gap-3 p-4 border-t border-gray-200 dark:border-gray-700">
            <button
              @click="closeRepostModal"
              class="px-4 py-2 text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg transition-colors"
              :disabled="isReposting"
            >
              取消
            </button>
            <button
              @click="handleRepost"
              :disabled="isReposting || currentLength > 140"
              class="px-4 py-2 bg-blue-500 hover:bg-blue-600 disabled:bg-gray-400 disabled:cursor-not-allowed text-white rounded-lg transition-colors"
            >
              <span v-if="isReposting"><i class="fas fa-spinner fa-spin mr-1"></i>转发中...</span>
              <span v-else>转发</span>
            </button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>