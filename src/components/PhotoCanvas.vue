<template>
  <div id="container">
    <div>
      Number of photos:
      <select name='numOfPca' id='number-selector-pca' @change="onChange($event)" v-model="selectedNumber">
      <option value = '000'>1000</option>
      <option value = '005'>5999</option>
      <option value = '010'>10999</option>
      <option value = '100'>100999</option>
      <option value = '202'>202500</option>
    </select>
    </div>
    <div>
      <button id="btn-change-set" :disabled="selectedNumber !== '000'" v-on:click="onClick()">Change set</button>
    </div>
    <div id="scene-pca">PCA</div>
    <div id="scene-umap">UMAP</div>
  </div>
</template>
<script>
import * as THREE from 'three'

export default {
  name: 'PhotoCanvas',
  props: ['datasetHash', 'isCeleba'],
  data () {
    return {
      selectedNumber: '000'
    }
  },
  methods: {
    init: function () {
      console.log(this.datasetHash)
      const divPCA = document.getElementById('scene-pca')
      const divUMAP = document.getElementById('scene-umap')
      this.scenePca = new THREE.Scene()
      this.scenePca.background = new THREE.Color(0x191919)
      this.sceneUmap = new THREE.Scene()
      this.sceneUmap.background = new THREE.Color(0x0c0c0c)

      // this.cameraPca = new THREE.OrthographicCamera(
      //   (window.innerWidth / 2) / -2,
      //   (window.innerWidth / 2) / 2,
      //   (window.innerHeight * 0.8) / 2,
      //   (window.innerHeight * 0.8) / -2,
      //   1,
      //   1000
      // )

      this.cameraPca = new THREE.PerspectiveCamera(
        75,
        (window.innerWidth / 2) / (window.innerHeight * 0.8),
        0.1,
        1000
      )
      this.cameraUmap = new THREE.PerspectiveCamera(
        75,
        (window.innerWidth / 2) / (window.innerHeight * 0.8),
        0.1,
        1000
      )

      this.rendererPca = new THREE.WebGLRenderer()
      this.rendererUmap = new THREE.WebGLRenderer()
      this.rendererPca.setSize(window.innerWidth / 2, window.innerHeight * 0.8)
      divPCA.appendChild(this.rendererPca.domElement)
      this.rendererUmap.setSize(window.innerWidth / 2, window.innerHeight * 0.8)
      divUMAP.appendChild(this.rendererUmap.domElement)

      const light = new THREE.AmbientLight(0xffffff)
      this.scenePca.add(light)
      this.sceneUmap.add(light)

      this.cameraPca.position.z = 300
      this.cameraUmap.position.z = 300
    },
    animate: function () {
      requestAnimationFrame(this.animate)
      this.rendererPca.render(this.scenePca, this.cameraPca)
      this.rendererUmap.render(this.sceneUmap, this.cameraUmap)
    },
    getImages: async function (start, numOfPhotos) {
      this.imagePieces = []

      for (let i = start; i <= parseInt(numOfPhotos); i++) {
        const image = new Image()
        let xd
        if (i < 10) {
          xd = await fetch(
        `http://localhost:5000/tilemaps/00${i}`
          )
        } else if (i < 100) {
          xd = await fetch(
        `http://localhost:5000/tilemaps/0${i}`
          )
        } else {
          xd = await fetch(
        `http://localhost:5000/tilemaps/${i}`
          )
        }
        let xdblob = await xd.blob()

        xdblob = xdblob.slice(0, xdblob.size, 'image/jpeg')
        console.log(xdblob)
        const objurl = URL.createObjectURL(xdblob)
        image.src = objurl
        this.image = image

        this.image.onload = () => {
          for (let y = 0; y < 100; ++y) {
            for (let x = 0; x < 10; ++x) {
              const canvas = document.createElement('canvas')
              canvas.width = 50
              canvas.height = 50
              const context = canvas.getContext('2d')
              context.drawImage(image, x * 50, y * 50, 50, 50, 0, 0, 50, 50)
              const urled = canvas.toDataURL()
              if (i === 0 && y === 99 && x === 9) {
              } else {
                this.imagePieces.push(urled)
              }
            }
          }
          console.log('numOfURLs:', this.imagePieces.length)
        }
      }

      await this.getPCA(start, parseInt(numOfPhotos))
      await this.getUMAP(start, parseInt(numOfPhotos))

      const combined = this.imagePieces.map((image, i) => {
        return {
          image,
          pcaData: this.pcaData.features[i],
          umapData: this.umapData.features[i]
        }
      })

      combined.forEach((elem, i) => {
        var img = new THREE.MeshBasicMaterial({
          map: new THREE.TextureLoader().load(elem.image)
        })

        var planePca = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), img)
        planePca.overdraw = true

        var planeUmap = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), img)
        planeUmap.overdraw = true

        if (elem.pcaData) {
          planePca.position.x = elem.pcaData[0] * 10
          planePca.position.y = elem.pcaData[1] * 10
          planePca.position.z = elem.pcaData[2] * 10

          this.scenePca.add(planePca)
        }

        if (elem.umapData) {
          planeUmap.position.x = elem.umapData[0] * 10
          planeUmap.position.y = elem.umapData[1] * 10
          planeUmap.position.z = elem.umapData[2] * 10

          this.sceneUmap.add(planeUmap)
        }
      })
    },
    getPCA: async function (start, end) {
      const pca = await fetch(
        `http://localhost:5000/pca/${start}/${end}/standalone`
      )
      this.pcaData = await pca.json()
      console.log('pcaData', this.pcaData)
    },
    getUMAP: async function (start, end) {
      const umap = await fetch(
        `http://localhost:5000/umap/${start}/${end}/standalone`
      )
      this.umapData = await umap.json()
      console.log('umapData', this.umapData)
    },

    getImagesFromHash: async function () {
      if (!this.isCeleba) {
        this.imagePieces = []
        const image = new Image()

        const xd = await fetch(
        `http://localhost:5000/dataset/${this.datasetHash}/tilemap`
        )

        let xdblob = await xd.blob()

        xdblob = xdblob.slice(0, xdblob.size, 'image/jpeg')
        console.log(xdblob)
        const objurl = URL.createObjectURL(xdblob)
        image.src = objurl
        this.image = image

        this.image.onload = () => {
          for (let y = 0; y < 100; ++y) {
            for (let x = 0; x < 10; ++x) {
              const canvas = document.createElement('canvas')
              canvas.width = 50
              canvas.height = 50
              const context = canvas.getContext('2d')
              context.drawImage(image, x * 50, y * 50, 50, 50, 0, 0, 50, 50)
              const urled = canvas.toDataURL()
              if (y === 99 && x === 9) {
              } else {
                this.imagePieces.push(urled)
              }
            }
          }
          console.log('numOfURLs:', this.imagePieces.length)
        }

        await this.getPCAFromHash()
        await this.getUMAPFromHash()

        const combined = this.imagePieces.map((image, i) => {
          return {
            image,
            pcaData: this.pcaData.features[i],
            umapData: this.umapData.features[i]
          }
        })

        combined.forEach((elem, i) => {
          var img = new THREE.MeshBasicMaterial({
            map: new THREE.TextureLoader().load(elem.image)
          })

          var planePca = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), img)
          planePca.overdraw = true

          var planeUmap = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), img)
          planeUmap.overdraw = true

          if (elem.pcaData) {
            planePca.position.x = elem.pcaData[0] * 10
            planePca.position.y = elem.pcaData[1] * 10
            planePca.position.z = elem.pcaData[2] * 10

            this.scenePca.add(planePca)
          }

          if (elem.umapData) {
            planeUmap.position.x = elem.umapData[0] * 10
            planeUmap.position.y = elem.umapData[1] * 10
            planeUmap.position.z = elem.umapData[2] * 10

            this.sceneUmap.add(planeUmap)
          }
        })
      }
    },
    getPCAFromHash: async function () {
      if (!this.isCeleba) {
        const pca = await fetch(
          `http://localhost:5000/dataset/${this.datasetHash}/features/pca`
        )
        this.pcaData = await pca.json()
        console.log('pcaData', this.pcaData)
      }
    },
    getUMAPFromHash: async function () {
      if (!this.isCeleba) {
        const umap = await fetch(
          `http://localhost:5000/dataset/${this.datasetHash}/features/umap`
        )
        this.umapData = await umap.json()
        console.log('umapData', this.umapData)
      }
    },
    clearTree: function clearThree (obj) {
      while (obj.children.length > 0) {
        clearThree(obj.children[0])
        obj.remove(obj.children[0])
      }
      if (obj.geometry) obj.geometry.dispose()

      if (obj.material) {
        Object.keys(obj.material).forEach(prop => {
          if (!obj.material[prop]) { return }
          if (obj.material[prop] !== null && typeof obj.material[prop].dispose === 'function') { obj.material[prop].dispose() }
        })
        obj.material.dispose()
      }
    },
    onChange (event) {
      console.log(event.target.value)
      this.clearTree(this.scenePca)
      this.clearTree(this.sceneUmap)
      if (this.isCeleba) {
        this.getImages(0, event.target.value)
      } else {
        this.getImagesFromHash()
      }
      this.animate()
    },
    onClick: function () {
      const num = Math.floor(Math.random() * (202 - 0 + 1)) + 0
      console.log(num)
      this.clearTree(this.scenePca)
      this.clearTree(this.sceneUmap)
      if (this.isCeleba) {
        this.getImages(num, num)
      } else {
        this.getImagesFromHash()
      }
      this.animate()
    }
  },
  mounted () {
    this.$nextTick(function () {
      if (this.isCeleba) {
        this.getImages(0, '000')
      } else {
        this.getImagesFromHash()
      }
      this.init()
      this.animate()
    })
  }
}
</script>
<style lang="scss">
#container {
  overflow: hidden;
  font-size: 15px;
}
#scene-pca {
  float: left;
}
select {
  width: 80px;
  padding: 2px;
  font-size: 15px;
  cursor:pointer;
  display:inline-block;
  position:relative;
  color:#2c3e50;
  border:1px solid #e7e7e7;

  &:focus{
    outline: none;
  }
}
#btn-change-set {
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
