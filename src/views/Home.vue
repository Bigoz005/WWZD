<template>
  <div class='home'>
    <div v-if='!fileSubmited'>
    <ImportFile @catchEmit='catchEmit'/>
    <br>
    <button id="btn-celeba" v-on:click="onClick()">Click for CelebA dataset</button>
    </div>
    <PhotoCanvas v-else :datasetHash='datasetHash' :isCeleba='isCeleba'/>
  </div>
</template>

<script lang='ts'>
import { defineComponent } from 'vue'
import PhotoCanvas from '@/components/PhotoCanvas.vue'
import ImportFile from '@/components/ImportFile.vue'

export default defineComponent({
  name: 'Home',
  data () {
    return {
      fileSubmited: false,
      isCeleba: false,
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
    },
    onClick () {
      this.isCeleba = true
      this.fileSubmited = true
    }
  }
})
</script>

<style>
#btn-celeba {
  padding: 5px 10px;
  margin-top: 5px;
  border-radius:0.15em;
  background-color: white;
  color: #2c3e50;
  text-align: center;
  text-decoration: none;
  display: inline-block;
  font-size: 15px;
  border: 2px solid #e7e7e7;
  border-bottom-color: #727272;
  border-right-color: #727272;
  transition: all 0.4s;
  cursor: pointer;

  &:hover {
    background-color: #e7e7e7;
  }

  &:active {
    transform: translateY(1px);
    transform: translateX(1px);
    border-color: #e7e7e7;
  }

  &:disabled {
    color: #727272;
    background-color: #e7e7e7;
    border: 2px solid #e7e7e7;
    cursor: not-allowed;
  }
}
</style>
