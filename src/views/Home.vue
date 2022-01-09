<template>
  <div class='home'>
    <ImportFile v-if='!fileSubmited' @catchEmit='catchEmit'/>
    <PhotoCanvas v-else :datasetHash='datasetHash'/>
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
      fileSubmited: false,
      datasetHash: ''
    }
  },
  components: {
    PhotoCanvas,
    ImportFile
  },
  methods: {
    catchEmit (data: string) {
      this.fileSubmited = true
      this.datasetHash = data
    }
  }
})
</script>
