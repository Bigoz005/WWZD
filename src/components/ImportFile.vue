<template>
<div class="div1">
  <div class="div2">
    <div v-bind="getRootProps()">
      <input accept =".zip" v-bind="getInputProps()" />
      <p v-if="isDragActive">Drop the files here ...</p>
      <p v-else>Drag 'n' drop files here(only first zip file will be processed)</p>
    </div>
  </div>
  <br>
  <span>OR</span>
  <br>
  <div>
    <input id="file-button" type="file" @change="onFileSelected" accept =".zip"/>
  </div>
  <br>
  <b>If proper zip file will be uploaded graphs will show after file processing</b>
</div>
</template>

<script>
import axios from 'axios'
import { useDropzone } from 'vue3-dropzone'
export default {
  name: 'ImportFiles',
  emits: ['catchEmit'],
  setup (props, { emit }) {
    let disabled = true
    let selectedFile = null
    let responseStatus = null
    const formDataToSend = new FormData()

    function checkResponse (data) {
      if (responseStatus === 200 || responseStatus === 304 || responseStatus === '200' || responseStatus === '304') {
        emit('catchEmit', data)
      }
    }

    function onDrop (acceptFiles, rejectReasons) {
      selectedFile = null
      disabled = true
      console.log('AcceptFiles: ', acceptFiles)
      let found = false
      acceptFiles.forEach(element => {
        if (element.type === 'application/x-zip-compressed' && found === false) {
          selectedFile = element
          found = true
          formDataToSend.append('file', selectedFile)
          console.log('axios: ', formDataToSend.get('file'))
          axios.post('http://localhost:5000/dataset', formDataToSend, {
            headers: {
              'Content-Type': 'multipart/form-data'
            // 'Content-Type': 'application/x-zip-compressed'
            }
          }).then(response => {
            responseStatus = response.status
            checkResponse(response.data)
          }).catch(error => {
            responseStatus = error
            console.log(error)
          })
        }
      })
    }
    const { getRootProps, getInputProps, ...rest } = useDropzone({ onDrop })
    return {
      getRootProps,
      getInputProps,
      disabled,
      selectedFile,
      responseStatus,
      formDataToSend,
      ...rest
    }
  },
  methods: {
    onFileSelected (event) {
      this.selectedFile = event.target.files[0]
      console.log(this.disabled)
      if (this.selectedFile != null) {
        this.disabled = false
        this.uploadFile()
      } else {
        this.disabled = true
      }
    },
    checkResponse (data) {
      if (this.responseStatus === 200 || this.responseStatus === 304 || this.responseStatus === '200' || this.responseStatus === '304') {
        this.$emit('catchEmit', data)
      }
    },
    uploadFile () {
      this.formDataToSend = new FormData()
      this.formDataToSend.append('file', this.selectedFile)
      console.log(this.formDataToSend.get('file'))
      axios.post('http://localhost:5000/dataset', this.formDataToSend, {
        headers: {
          'Content-Type': 'multipart/form-data'
        //   'Content-Type': 'application/x-zip-compressed'
        }
      })
        .then(response => {
          this.responseStatus = response.status
          this.checkResponse(response.data)
        }).catch(error => {
          this.responseStatus = error.response.status
          console.log(error)
        })
    }
  }
}
</script>

<style>
.div2 {
  height: 100px;
  padding: 35px;
  display: flex;
  text-align: center;
  justify-content: center;
  border: 5px solid grey;
}
.div1 {
  padding: 35px;
  border: 1px solid grey;
}
#file-button {
  cursor: pointer;
}
</style>
