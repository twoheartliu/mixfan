<script setup>
const props = defineProps({
  media: {
    type: Array,
    required: true
  }
})

// 定义事件，用于向父组件传递预览图片的请求
const emit = defineEmits(['preview-image'])

// 点击图片时触发预览
const handleImageClick = (item) => {
  const imageUrl = item.largeurl || item.url
  console.log('MediaAttachment: handleImageClick', imageUrl)
  emit('preview-image', imageUrl)
}

</script>

<template>
  <div v-if="media.length > 0" class="mt-2 grid grid-cols-2 gap-2">
    <div v-for="item in media" :key="item.id || item.url" class="block cursor-pointer max-h-64 overflow-hidden">
      <img :src="item.largeurl || item.thumburl || item.url" class="w-full h-auto rounded-lg object-cover"
        :alt="item.description || ''" @click="handleImageClick(item)">
    </div>
  </div>
</template>