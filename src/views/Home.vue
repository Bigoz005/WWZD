<template>
  <div class='home'>
    <ImportFile v-if='!fileSubmited' @catchEmit='catchEmit'/>
    <PhotoCanvas v-else />
  </div>
</template>

<script lang='ts'>
import { defineComponent } from 'vue'
import PhotoCanvas from '@/components/PhotoCanvas.vue'
import ImportFile from '@/components/ImportFile.vue'

export default defineComponent({
  name: 'Home',
  // event: 'fileSubmited',
  data () {
    return {
      fileSubmited: false
    }
  },
  components: {
    PhotoCanvas,
    ImportFile
  },
  methods: {
    catchEmit () {
      this.fileSubmited = true
    }
  }
})
</script>
