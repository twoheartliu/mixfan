<script setup>
import useCompose from '@/composables/useCompose'
import LoadingSpinner from '@/components/common/LoadingSpinner.vue'

const {
  inputValue,
  countdown,
  status,
  isLoading,
  fileList,
  updateChars,
  sendMessage,
  handleFileUpload,
  handleRemoveImg,
  handleKeyDown
} = useCompose()
</script>

<template>
  <div class="compose-box rounded-lg p-4 border border-gray-200 dark:border-gray-700">
    <textarea v-model="inputValue" @keydown="handleKeyDown" @input="updateChars"
      class="w-full h-24 bg-transparent resize-none focus:outline-none dark:text-white placeholder:text-gray-400 dark:placeholder:text-gray-500" placeholder="有什么新鲜事想分享给大家？"></textarea>
    <div class="status text-sm mt-1 text-gray-600 dark:text-gray-400" v-if="status">
      {{ status }}
      <LoadingSpinner v-if="isLoading" size="xs" class="inline-block ml-1" />
    </div>
    <div class="flex justify-between items-center mt-4">
      <div class="flex space-x-4">
        <div class="relative">
          <button class="p-2 rounded-full hover:bg-gray-200 dark:hover:bg-gray-700 text-gray-600 dark:text-gray-400">
            <i class="fas fa-image"></i>
          </button>
          <input @change="handleFileUpload" title="上传图片，最大2MB" class="absolute inset-0 opacity-0 cursor-pointer"
            type="file" accept="image/jpeg,image/png,image/gif" />
        </div>
        <span v-if="fileList.length" class="flex items-center text-gray-700 dark:text-gray-300">
          <span class="truncate max-w-[100px]">{{ fileList[0].name }}</span>
          <button @click="handleRemoveImg" class="ml-2 text-red-500 dark:text-red-400">
            <i class="fas fa-times"></i>
          </button>
        </span>
      </div>
      <div class="flex items-center space-x-4">
        <span :class="countdown < 0 ? 'text-red-500 dark:text-red-400' : 'text-gray-400 dark:text-gray-500'">{{ countdown }}</span>
        <button @click="sendMessage" :disabled="!inputValue || isLoading || countdown < 0"
          class="px-6 py-2 rounded-full bg-gradient-to-r from-blue-600 to-purple-600 text-white disabled:opacity-50">
          <LoadingSpinner v-if="isLoading" size="xs" class="inline-block mr-1" />
          {{ isLoading ? '发送中...' : '发送' }}
        </button>
      </div>
    </div>
  </div>
</template>