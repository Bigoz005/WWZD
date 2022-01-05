<template>
  <div class='home'>
    <PhotoCanvas />
  </div>
</template>

<script lang='ts'>
import { defineComponent } from 'vue'
import PhotoCanvas from '@/components/PhotoCanvas.vue'

export default defineComponent({
  name: 'Home',
  components: {
    PhotoCanvas
  }
})
</script>
