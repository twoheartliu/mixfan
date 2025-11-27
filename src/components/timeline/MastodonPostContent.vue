<script setup>
import { ref, onMounted, computed, onBeforeUnmount, nextTick } from 'vue'
import { safeGet, getMediaAttachments } from '@/utils/helpers'
import { formatTimeAgo } from '@/utils/formatters'
import { replaceEmojisFromStore } from '@/utils/emoji'
import MediaAttachment from './MediaAttachment.vue'
import { apis } from '@/utils/mastodon'
import useMastodonStore from '@/store/mastodon'

const mastoStore = useMastodonStore()

const props = defineProps({
  post: {
    type: Object,
    required: true
  }
})

// 定义事件，用于向父组件传递预览图片的请求
const emit = defineEmits(['preview-image', 'toggle-comments', 'reblog-success'])

// 控制内容警告的展开/收起
const showSpoilerContent = ref(false)

// 转发相关状态
const showReblogMenu = ref(false)
const showQuoteModal = ref(false)
const quoteContent = ref('')
const isReblogging = ref(false)

// 本地状态跟踪，确保响应式更新
const localReblogged = ref(null)
const localReblogCount = ref(null)

// 父级消息（如果是回复）
const parentPost = ref(null)
const loadingParent = ref(false)

// 检查当前帖子是否是回复
const isReply = computed(() => {
  const inReplyToId = safeGet(props.post, 'reblog.inReplyToId', safeGet(props.post, 'inReplyToId'))
  return !!inReplyToId
})

// 获取回复的消息ID
const replyToId = computed(() => {
  return safeGet(props.post, 'reblog.inReplyToId', safeGet(props.post, 'inReplyToId', ''))
})

// 检查是否有引用消息
const hasQuote = computed(() => {
  const quotedStatus = safeGet(props.post, 'reblog.quote.quotedStatus', safeGet(props.post, 'quote.quotedStatus'))
  return !!quotedStatus
})

// 获取引用的消息
const quotedPost = computed(() => {
  return safeGet(props.post, 'reblog.quote.quotedStatus', safeGet(props.post, 'quote.quotedStatus', null))
})

// 获取当前帖子的内容（移除引用的内联标记，替换 emoji）
const currentPostContent = computed(() => {
  let content = safeGet(props.post, 'reblog.content', safeGet(props.post, 'content', ''))

  // 如果有引用，移除 <p class="quote-inline">...</p> 部分
  if (hasQuote.value) {
    // 使用正则表达式移除 quote-inline 段落
    content = content.replace(/<p class="quote-inline">.*?<\/p>/i, '').trim()
  }

  // 替换自定义 emoji
  const postEmojis = safeGet(props.post, 'reblog.emojis', safeGet(props.post, 'emojis', []))
  content = replaceEmojisFromStore(content, mastoStore.customEmojis, postEmojis)

  return content
})

// 判断是否可以转发（private 和 direct 不允许转发）
const canReblog = computed(() => {
  const visibility = safeGet(props.post, 'reblog.visibility', safeGet(props.post, 'visibility', 'public'))
  return visibility === 'public' || visibility === 'unlisted'
})

// 判断是否已转发
const isReblogged = computed(() => {
  // 使用本地状态如果已设置
  if (localReblogged.value !== null) {
    return localReblogged.value
  }

  if (props.post.reblog) {
    return !!props.post.reblog.reblogged
  }
  return !!props.post.reblogged
})

// 计算转发总数（包括转发和引用）
const totalReblogsCount = computed(() => {
  // 使用本地计数如果已设置
  if (localReblogCount.value !== null) {
    return localReblogCount.value
  }

  if (props.post.reblog) {
    const reblogs = props.post.reblog.reblogsCount || 0
    const quotes = props.post.reblog.quotesCount || 0
    return reblogs + quotes
  }
  const reblogs = props.post.reblogsCount || 0
  const quotes = props.post.quotesCount || 0
  return reblogs + quotes
})

// 获取帖子的可见性
const postVisibility = computed(() => {
  return safeGet(props.post, 'reblog.visibility', safeGet(props.post, 'visibility', 'public'))
})

// 获取用户显示名称（替换 emoji）
const userDisplayName = computed(() => {
  const displayName = safeGet(props.post, 'reblog.account.displayName',
    safeGet(props.post, 'reblog.account.username',
      safeGet(props.post, 'account.displayName',
        safeGet(props.post, 'account.username', '未知用户'))))

  // 获取用户的 emoji
  const accountEmojis = safeGet(props.post, 'reblog.account.emojis', safeGet(props.post, 'account.emojis', []))

  return replaceEmojisFromStore(displayName, mastoStore.customEmojis, accountEmojis)
})

// 获取引用消息的内容（替换 emoji）
const quotedPostContent = computed(() => {
  if (!quotedPost.value) return ''

  const content = quotedPost.value.content || ''
  const quotedEmojis = quotedPost.value.emojis || []

  return replaceEmojisFromStore(content, mastoStore.customEmojis, quotedEmojis)
})

// 获取引用消息作者显示名称（替换 emoji）
const quotedPostAuthorName = computed(() => {
  if (!quotedPost.value) return ''

  const displayName = quotedPost.value.account?.displayName || quotedPost.value.account?.username || '未知用户'
  const accountEmojis = quotedPost.value.account?.emojis || []

  return replaceEmojisFromStore(displayName, mastoStore.customEmojis, accountEmojis)
})

// 获取父消息的内容（替换 emoji）
const parentPostContent = computed(() => {
  if (!parentPost.value) return ''

  const content = parentPost.value.content || ''
  const parentEmojis = parentPost.value.emojis || []

  return replaceEmojisFromStore(content, mastoStore.customEmojis, parentEmojis)
})

// 获取父消息作者显示名称（替换 emoji）
const parentPostAuthorName = computed(() => {
  if (!parentPost.value) return ''

  const displayName = parentPost.value.account?.displayName || parentPost.value.account?.username || '未知用户'
  const accountEmojis = parentPost.value.account?.emojis || []

  return replaceEmojisFromStore(displayName, mastoStore.customEmojis, accountEmojis)
})

const handlePreviewImage = (imageUrl) => {
  console.log('PostContent: handlePreviewImage', imageUrl)
  emit('preview-image', imageUrl)
}

// 处理卡片图片点击
const handleCardImageClick = (event, imageUrl) => {
  event.preventDefault()
  emit('preview-image', imageUrl)
}

// 切换内容警告显示
const toggleSpoiler = () => {
  showSpoilerContent.value = !showSpoilerContent.value
}

// 获取父级消息
async function fetchParentPost() {
  if (!isReply.value || !replyToId.value) return

  loadingParent.value = true
  try {
    const parent = await apis.masto.v1.statuses.$select(replyToId.value).fetch()
    parentPost.value = parent
  } catch (error) {
    console.error('获取父级消息失败:', error)
    parentPost.value = null
  } finally {
    loadingParent.value = false
  }
}

// 组件挂载时获取父级消息
onMounted(() => {
  if (isReply.value) {
    fetchParentPost()
  }
  // 添加全局点击监听，用于关闭转发菜单
  document.addEventListener('click', handleClickOutside)

  // 初始化本地状态
  localReblogged.value = props.post.reblog ? !!props.post.reblog.reblogged : !!props.post.reblogged
  localReblogCount.value = totalReblogsCount.value
})

onBeforeUnmount(() => {
  // 移除全局点击监听
  document.removeEventListener('click', handleClickOutside)
})

// 处理点击外部关闭菜单
function handleClickOutside(event) {
  const target = event.target
  // 如果点击的不是转发按钮或菜单内部，则关闭菜单
  if (!target.closest('.reblog-menu-container')) {
    closeReblogMenu()
  }
}

// 获取原始消息 ID（去掉前缀）
function getOriginalPostId() {
  const id = safeGet(props.post, 'reblog.id', safeGet(props.post, 'id', ''))
  return id.replace(/^mast_/, '')
}

// 切换转发菜单
function toggleReblogMenu() {
  showReblogMenu.value = !showReblogMenu.value
}

// 关闭转发菜单
function closeReblogMenu() {
  showReblogMenu.value = false
}

// 执行转发（Reblog/Boost）或取消转发
async function handleReblog() {
  if (isReblogging.value) return

  closeReblogMenu()
  isReblogging.value = true

  try {
    const postId = getOriginalPostId()
    const wasReblogged = isReblogged.value

    // 使用本地状态进行UI更新，确保响应式
    localReblogged.value = !wasReblogged
    localReblogCount.value = Math.max(0, localReblogCount.value + (wasReblogged ? -1 : 1))

    // 执行API调用
    if (wasReblogged) {
      // 取消转发
      await apis.masto.v1.statuses.$select(postId).unreblog()
    } else {
      // 执行转发
      await apis.masto.v1.statuses.$select(postId).reblog()
    }

    // 更新原始对象（虽然可能不会触发响应式更新）
    if (props.post.reblog) {
      props.post.reblog.reblogged = !wasReblogged
      props.post.reblog.reblogsCount = Math.max(0, (props.post.reblog.reblogsCount || 0) + (wasReblogged ? -1 : 1))
    } else {
      props.post.reblogged = !wasReblogged
      props.post.reblogsCount = Math.max(0, (props.post.reblogsCount || 0) + (wasReblogged ? -1 : 1))
    }

    // 发出成功事件
    emit('reblog-success')

  } catch (error) {
    console.error('转发操作失败:', error)

    // 发生错误时回滚本地状态
    localReblogged.value = !localReblogged.value
    localReblogCount.value = Math.max(0, localReblogCount.value + (localReblogged.value ? 1 : -1))

    alert(localReblogged.value ? '取消转发失败，请重试' : '转发失败，请重试')
  } finally {
    isReblogging.value = false
  }
}

// 打开引用弹窗
function openQuoteModal() {
  quoteContent.value = ''
  showQuoteModal.value = true
  closeReblogMenu()
}

// 关闭引用弹窗
function closeQuoteModal() {
  showQuoteModal.value = false
  quoteContent.value = ''
}

// 执行引用
async function handleQuote() {
  if (isReblogging.value) return

  const content = quoteContent.value.trim()
  if (!content) {
    alert('请输入引用内容')
    return
  }

  if (content.length > 500) {
    alert('内容超过500字限制')
    return
  }

  isReblogging.value = true
  try {
    const postId = getOriginalPostId()

    await apis.masto.v1.statuses.create({
      status: content,
      language: 'zh-cn',
      sensitive: false,
      mediaIds: [],
      quotedStatusId: postId,
      quoteApprovalPolicy: 'public',
      visibility: 'public'
    })

    // 更新本地计数（引用也算在转发计数中）
    localReblogCount.value = (localReblogCount.value || 0) + 1

    closeQuoteModal()
    emit('reblog-success')
  } catch (error) {
    console.error('引用失败:', error)
    alert('引用失败，请重试')
  } finally {
    isReblogging.value = false
  }
}

</script>

<template>
  <div>
    <!-- 回复的原始消息 -->
    <div v-if="isReply && parentPost" class="mb-3 p-3 bg-gray-800 bg-opacity-50 rounded-lg border-l-2 border-blue-500">
      <div class="flex items-center space-x-2 mb-2 text-xs text-gray-400">
        <i class="fas fa-reply"></i>
        <span>回复</span>
      </div>

      <div class="flex items-start space-x-2">
        <!-- 父消息头像 -->
        <img v-if="parentPost.account?.avatar" :src="parentPost.account.avatar" class="w-6 h-6 rounded-full flex-shrink-0"
          :alt="parentPost.account.displayName || parentPost.account.username" />

        <div class="flex-1 min-w-0">
          <!-- 父消息作者 -->
          <div class="flex items-center space-x-2 mb-1">
            <span class="font-medium text-sm truncate" v-html="parentPostAuthorName"></span>
            <span class="text-xs text-gray-500 truncate">
              @{{ parentPost.account?.acct || '' }}
            </span>
            <span class="text-xs text-gray-500">·</span>
            <span class="text-xs text-gray-500 flex-shrink-0">
              {{ formatTimeAgo(parentPost.createdAt) }}
            </span>
          </div>

          <!-- 父消息内容（截断显示） -->
          <div class="text-sm text-gray-300 line-clamp-3" v-html="parentPostContent"></div>
        </div>
      </div>
    </div>

    <!-- 加载父消息中 -->
    <div v-else-if="isReply && loadingParent"
      class="mb-3 p-3 bg-gray-800 bg-opacity-50 rounded-lg border-l-2 border-blue-500">
      <div class="flex items-center space-x-2 text-xs text-gray-400">
        <i class="fas fa-spinner fa-spin"></i>
        <span>加载回复的消息...</span>
      </div>
    </div>

    <!-- 用户信息 -->
    <div class="flex items-center space-x-2 flex-wrap">
      <span class="font-bold truncate" v-html="userDisplayName"></span>
      <span class="text-sm text-gray-500">
        @{{ safeGet(post, 'reblog.account.acct', safeGet(post, 'account.acct', '')) }}
      </span>
      <span class="text-sm text-gray-500">·</span>
      <span class="text-sm text-gray-500">
        {{ formatTimeAgo(post.createdAt) }}
      </span>

      <!-- 编辑标记 -->
      <span v-if="safeGet(post, 'reblog.editedAt', safeGet(post, 'editedAt'))" class="text-sm text-gray-500" title="已编辑">
        <i class="fas fa-pen text-xs"></i>
      </span>

      <!-- 可见性标记 -->
      <span v-if="safeGet(post, 'reblog.visibility', safeGet(post, 'visibility')) === 'unlisted'"
        class="text-sm text-gray-500" title="不公开列出">
        <i class="fas fa-lock-open text-xs"></i>
      </span>
      <span v-else-if="safeGet(post, 'reblog.visibility', safeGet(post, 'visibility')) === 'private'"
        class="text-sm text-gray-500" title="仅关注者">
        <i class="fas fa-lock text-xs"></i>
      </span>
      <span v-else-if="safeGet(post, 'reblog.visibility', safeGet(post, 'visibility')) === 'direct'"
        class="text-sm text-gray-500" title="私信">
        <i class="fas fa-envelope text-xs"></i>
      </span>

      <!-- 敏感内容标记 -->
      <span v-if="safeGet(post, 'reblog.sensitive', safeGet(post, 'sensitive', false))" class="text-sm text-orange-500"
        title="敏感内容">
        <i class="fas fa-eye-slash text-xs"></i>
      </span>

      <Icon name="mastodon" class="text-purple-500 ml-1" />

      <!-- 转发标记 -->
      <span v-if="post.reblog" class="ml-2 text-sm text-gray-500">
        <i class="fas fa-retweet mr-1"></i> 转发
      </span>
    </div>

    <!-- 发布应用信息 -->
    <div v-if="safeGet(post, 'reblog.application.name', safeGet(post, 'application.name'))"
      class="text-xs text-gray-500 mt-1">
      通过 {{ safeGet(post, 'reblog.application.name', safeGet(post, 'application.name', '')) }}
    </div>

    <!-- 转发者信息 -->
    <div v-if="post.reblog" class="mt-1 text-sm text-gray-500">
      <span class="font-medium">{{ safeGet(post, 'account.displayName', safeGet(post, 'account.username', ''))
      }}</span> 转发了
    </div>

    <!-- 内容警告 / Spoiler Text -->
    <div v-if="safeGet(post, 'reblog.spoilerText', safeGet(post, 'spoilerText'))" class="mt-2">
      <div class="p-3 bg-yellow-500 bg-opacity-10 border border-yellow-500 border-opacity-30 rounded-lg">
        <div class="flex items-start space-x-2">
          <i class="fas fa-exclamation-triangle text-yellow-500 mt-1"></i>
          <div class="flex-1">
            <div class="font-medium text-sm mb-2">
              {{ safeGet(post, 'reblog.spoilerText', safeGet(post, 'spoilerText', '')) }}
            </div>
            <button @click="toggleSpoiler" class="text-sm text-blue-400 hover:text-blue-300 flex items-center space-x-1">
              <span>{{ showSpoilerContent ? '隐藏内容' : '显示内容' }}</span>
              <i :class="showSpoilerContent ? 'fas fa-chevron-up' : 'fas fa-chevron-down'"></i>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 消息内容 -->
    <div class="mt-2 text-sm md:text-base"
      v-show="!safeGet(post, 'reblog.spoilerText', safeGet(post, 'spoilerText')) || showSpoilerContent">
      <!-- Mastodon 内容 - 需要处理HTML -->
      <div v-html="currentPostContent"></div>

      <!-- 媒体附件 -->
      <MediaAttachment :media="getMediaAttachments(post)" @preview-image="handlePreviewImage" />

      <!-- 卡片 -->
      <div v-if="(safeGet(post, 'reblog.card') || safeGet(post, 'card'))"
        class="mt-3 border border-gray-700 rounded-lg overflow-hidden">
        <a :href="safeGet(post, 'reblog.card.url', safeGet(post, 'card.url', '#'))" target="_blank" class="block">
          <div v-if="safeGet(post, 'reblog.card.image', safeGet(post, 'card.image'))"
            class="aspect-video bg-gray-800 max-h-64 overflow-hidden">
            <img :src="safeGet(post, 'reblog.card.image', safeGet(post, 'card.image'))"
              class="w-full object-cover cursor-pointer"
              :alt="safeGet(post, 'reblog.card.title', safeGet(post, 'card.title', ''))"
              @click="(e) => handleCardImageClick(e, safeGet(post, 'reblog.card.image', safeGet(post, 'card.image')))">
          </div>
          <div class="p-3">
            <div class="font-bold text-sm">{{ safeGet(post, 'reblog.card.title', safeGet(post, 'card.title', '')) }}
            </div>
            <div class="text-xs text-gray-500 mt-1 line-clamp-2">
              {{ safeGet(post, 'reblog.card.description', safeGet(post, 'card.description', '')) }}
            </div>
            <div class="text-xs text-gray-500 mt-1">
              {{ safeGet(post, 'reblog.card.providerName', safeGet(post, 'card.providerName', '')) }}
            </div>
          </div>
        </a>
      </div>

      <!-- 投票 -->
      <div v-if="safeGet(post, 'reblog.poll') || safeGet(post, 'poll')"
        class="mt-3 p-3 border border-gray-700 rounded-lg">
        <div v-for="(option, index) in safeGet(post, 'reblog.poll.options', safeGet(post, 'poll.options', []))"
          :key="index" class="mb-2 last:mb-0">
          <div class="flex items-center">
            <div class="w-full bg-gray-700 rounded-full h-6 mr-2 relative overflow-hidden">
              <div class="bg-blue-600 h-full rounded-full" :style="{
                width: `${option.votesCount / safeGet(post, 'reblog.poll.votesCount', safeGet(post, 'poll.votesCount', 1)) * 100}%`
              }"></div>
              <div class="absolute inset-0 flex items-center px-3">
                <span class="text-xs text-white">{{ option.title }}</span>
              </div>
            </div>
            <span class="text-xs whitespace-nowrap">
              {{ option.votesCount }} 票
            </span>
          </div>
        </div>
        <div class="text-xs text-gray-500 mt-2">
          总票数: {{ safeGet(post, 'reblog.poll.votesCount', safeGet(post, 'poll.votesCount', 0)) }}
          <span v-if="safeGet(post, 'reblog.poll.expired', safeGet(post, 'poll.expired', false))" class="ml-2">
            (已结束)
          </span>
          <span v-else class="ml-2">
            ({{ formatTimeAgo(safeGet(post, 'reblog.poll.expiresAt', safeGet(post, 'poll.expiresAt', '')))
            }}结束)
          </span>
        </div>
      </div>

      <!-- 引用消息 -->
      <div v-if="hasQuote && quotedPost"
        class="mt-3 p-3 bg-gray-800 bg-opacity-50 rounded-lg border-l-2 border-green-500">
        <div class="flex items-center space-x-2 mb-2 text-xs text-gray-400">
          <i class="fas fa-quote-left"></i>
          <span>引用</span>
        </div>

        <div class="flex items-start space-x-2">
          <!-- 引用消息头像 -->
          <img v-if="quotedPost.account?.avatar" :src="quotedPost.account.avatar"
            class="w-6 h-6 rounded-full flex-shrink-0"
            :alt="quotedPost.account.displayName || quotedPost.account.username" />

          <div class="flex-1 min-w-0">
            <!-- 引用消息作者 -->
            <div class="flex items-center space-x-2 mb-1">
              <span class="font-medium text-sm truncate" v-html="quotedPostAuthorName"></span>
              <span class="text-xs text-gray-500 truncate">
                @{{ quotedPost.account?.acct || '' }}
              </span>
              <span class="text-xs text-gray-500">·</span>
              <span class="text-xs text-gray-500 flex-shrink-0">
                {{ formatTimeAgo(quotedPost.createdAt) }}
              </span>
            </div>

            <!-- 引用消息内容 -->
            <div class="text-sm text-gray-300" v-html="quotedPostContent"></div>

            <!-- 引用消息的媒体附件（如果有） -->
            <div v-if="quotedPost.mediaAttachments && quotedPost.mediaAttachments.length > 0" class="mt-2">
              <div class="grid grid-cols-2 gap-2">
                <img v-for="media in quotedPost.mediaAttachments.slice(0, 4)" :key="media.id"
                  :src="media.previewUrl || media.url" class="rounded w-full h-24 object-cover cursor-pointer"
                  :alt="media.description || ''" @click="handlePreviewImage(media.url)" />
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 交互按钮 -->
    <div class="flex flex-wrap gap-4 mt-3 text-sm">
      <button @click="emit('toggle-comments')" class="flex items-center space-x-2 hover:text-blue-400 transition-colors">
        <i class="far fa-comment"></i>
        <span>{{ post.reblog ? post.reblog.repliesCount || 0 : post.repliesCount || 0 }}</span>
      </button>

      <!-- 转发按钮（带下拉菜单） -->
      <div class="relative reblog-menu-container">
        <button @click="toggleReblogMenu" :disabled="!canReblog" :class="[
          'flex items-center space-x-2 transition-colors',
          canReblog ? (isReblogged ? 'text-green-500 hover:text-green-400' : 'hover:text-green-400') : 'opacity-50 cursor-not-allowed'
        ]" :title="!canReblog ? '此消息不可转发' : ''">
          <i class="fas fa-retweet"></i>
          <span>{{ totalReblogsCount }}</span>
        </button>

        <!-- 转发菜单 -->
        <div v-if="showReblogMenu && canReblog"
          class="absolute bottom-full mb-2 left-0 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg shadow-lg py-1 z-10 min-w-[120px]">
          <button @click="handleReblog"
            class="w-full text-left px-4 py-2 text-sm hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors flex items-center space-x-2"
            :disabled="isReblogging">
            <i class="fas fa-retweet text-green-500"></i>
            <span>{{ isReblogged ? '取消转发' : '转发' }}</span>
          </button>
          <button @click="openQuoteModal"
            class="w-full text-left px-4 py-2 text-sm hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors flex items-center space-x-2"
            :disabled="isReblogging">
            <i class="fas fa-quote-left text-blue-500"></i>
            <span>引用</span>
          </button>
        </div>
      </div>

      <button class="flex items-center space-x-2 hover:text-red-400">
        <i class="far fa-heart"></i>
        <span>{{ post.reblog ? post.reblog.favouritesCount || 0 : post.favouritesCount || 0 }}</span>
      </button>
      <button class="flex items-center space-x-2 hover:text-blue-400">
        <i class="far fa-bookmark"></i>
      </button>
    </div>

    <!-- 引用弹窗 -->
    <Teleport to="body">
      <div v-if="showQuoteModal" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4"
        @click.self="closeQuoteModal">
        <div class="bg-white dark:bg-gray-800 rounded-lg shadow-xl w-full max-w-lg">
          <!-- 弹窗头部 -->
          <div class="flex justify-between items-center p-4 border-b border-gray-200 dark:border-gray-700">
            <h2 class="text-lg font-bold dark:text-white">引用嘟文</h2>
            <button @click="closeQuoteModal"
              class="text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200">
              <i class="fas fa-times"></i>
            </button>
          </div>

          <!-- 弹窗内容 -->
          <div class="p-4">
            <!-- 引用内容输入框 -->
            <textarea v-model="quoteContent"
              class="w-full bg-gray-100 dark:bg-gray-700 rounded-lg p-3 text-sm resize-none focus:outline-none focus:ring-2 focus:ring-blue-500 dark:text-white"
              placeholder="说点什么..." rows="4" :disabled="isReblogging"></textarea>

            <!-- 字数统计 -->
            <div class="text-right text-xs mt-1" :class="quoteContent.length > 500 ? 'text-red-500' : 'text-gray-500'">
              {{ quoteContent.length }}/500
            </div>

            <!-- 原消息预览 -->
            <div class="mt-3 p-3 bg-gray-100 dark:bg-gray-700 rounded-lg border-l-2 border-green-500">
              <div class="text-xs text-gray-500 mb-1">引用的嘟文</div>
              <div class="flex items-start space-x-2">
                <img v-if="safeGet(post, 'reblog.account.avatar', safeGet(post, 'account.avatar'))"
                  :src="safeGet(post, 'reblog.account.avatar', safeGet(post, 'account.avatar'))"
                  class="w-6 h-6 rounded-full flex-shrink-0" />
                <div class="flex-1 min-w-0">
                  <div class="font-medium text-sm truncate" v-html="userDisplayName"></div>
                  <div class="text-sm text-gray-600 dark:text-gray-300 line-clamp-3" v-html="currentPostContent"></div>
                </div>
              </div>
            </div>
          </div>

          <!-- 弹窗底部 -->
          <div class="flex justify-end gap-3 p-4 border-t border-gray-200 dark:border-gray-700">
            <button @click="closeQuoteModal"
              class="px-4 py-2 text-gray-600 dark:text-gray-300 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-lg transition-colors"
              :disabled="isReblogging">
              取消
            </button>
            <button @click="handleQuote" :disabled="isReblogging || !quoteContent.trim() || quoteContent.length > 500"
              class="px-4 py-2 bg-blue-500 hover:bg-blue-600 disabled:bg-gray-400 disabled:cursor-not-allowed text-white rounded-lg transition-colors">
              <span v-if="isReblogging"><i class="fas fa-spinner fa-spin mr-1"></i>引用中...</span>
              <span v-else>引用</span>
            </button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>