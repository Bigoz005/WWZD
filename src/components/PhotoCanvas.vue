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
      <p>Move: WASD, Zoom: QE, Rotate: UIOJKL</p>
    </div>
    <div id="scene-pca">PCA</div>
    <div id="scene-umap">UMAP</div>
  </div>
</template>
<script>
import * as THREE from 'three'

export default {
  name: 'PhotoCanvas',
  props: ['datasetHash', 'isCeleba', 'isColored'],
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
      this.cameraPca.near = 30

      this.cameraUmap = new THREE.PerspectiveCamera(
        75,
        (window.innerWidth / 2) / (window.innerHeight * 0.8),
        0.1,
        1000
      )
      this.cameraUmap.near = 30

      this.cameraPca.updateProjectionMatrix()
      this.cameraUmap.updateProjectionMatrix()

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

      window.addEventListener('keydown', this.onKeyDown, true)
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
    getColoredImages: async function (start, numOfPhotos) {
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
      var r = 255
      var g = 255
      var b = 255

      combined.forEach((elem, i) => {
        console.log(i + ': ' + elem.image)

        var imgPca = new THREE.MeshBasicMaterial({
          map: new THREE.TextureLoader().load(elem.image)
        })

        var imgUmap = new THREE.MeshBasicMaterial({
          map: new THREE.TextureLoader().load(elem.image)
        })

        var planePca = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), imgPca)
        planePca.overdraw = true

        var planeUmap = new THREE.Mesh(new THREE.PlaneGeometry(10, 10), imgUmap)
        planeUmap.overdraw = true

        var rgb

        if (elem.pcaData) {
          planePca.position.x = elem.pcaData[0] * 10
          planePca.position.y = elem.pcaData[1] * 10
          planePca.position.z = elem.pcaData[2] * 10

          if (planePca.position.x <= -30) {
            rgb = new THREE.Color('rgb(' + (r) + ',' + Math.round(0.5 * g) + ',' + Math.round(0.1 * b) + ')')
            imgPca.color = rgb
          } else
          if (planePca.position.x <= 200 && planePca.position.x > -30) {
            rgb = new THREE.Color('rgb(' + Math.round(0.5 * r) + ',' + (g) + ',' + Math.round(0.5 * b) + ')')
            imgPca.color = rgb
          } else
          if (planePca.position.x > 201) {
            rgb = new THREE.Color('rgb(' + Math.round(0.1 * r) + ',' + Math.round(0.5 * g) + ',' + (b) + ')')
            imgPca.color = rgb
          }
          this.scenePca.add(planePca)
        }

        if (elem.umapData) {
          planeUmap.position.x = elem.umapData[0] * 10
          planeUmap.position.y = elem.umapData[1] * 10
          planeUmap.position.z = elem.umapData[2] * 10

          // console.log(planeUmap.position.x)
          // console.log(planeUmap.position.y)
          // console.log(planeUmap.position.z)
          // var rgbx = new THREE.Color('rgb(' + Math.round(Math.abs(planeUmap.position.x) * r) + ',' + Math.round(Math.abs(planeUmap.position.x) * g) + ',' + Math.round(Math.abs(planeUmap.position.x) * b) + ')')
          // var rgby = new THREE.Color('rgb(' + Math.round(Math.abs(planeUmap.position.y) * r) + ',' + Math.round(Math.abs(planeUmap.position.y) * g) + ',' + Math.round(Math.abs(planeUmap.position.y) * b) + ')')
          // var rgbz = new THREE.Color('rgb(' + Math.round(Math.abs(planeUmap.position.z) * r) + ',' + Math.round(Math.abs(planeUmap.position.z) * g) + ',' + Math.round(Math.abs(planeUmap.position.z) * b) + ')')

          if (planeUmap.position.x <= 50) {
            rgb = new THREE.Color('rgb(' + (r) + ',' + Math.round(0.5 * g) + ',' + Math.round(0.1 * b) + ')')
            imgUmap.color = rgb
          } else
          if (planeUmap.position.x <= 90 && planeUmap.position.x > 50) {
            rgb = new THREE.Color('rgb(' + Math.round(0.5 * r) + ',' + (g) + ',' + Math.round(0.5 * b) + ')')
            imgUmap.color = rgb
          } else
          if (planeUmap.position.x > 91) {
            rgb = new THREE.Color('rgb(' + Math.round(0.1 * r) + ',' + Math.round(0.5 * g) + ',' + (b) + ')')
            imgUmap.color = rgb
          }

          this.sceneUmap.add(planeUmap)
        }
      })
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
    },
    onKeyDown: function (e) {
      console.log('keyDown')
      switch (e.keyCode) {
        case 83: // W UP
          this.cameraPca.position.y += 5
          this.cameraUmap.position.y += 5
          break
        case 87: // S DOWN
          this.cameraPca.position.y -= 5
          this.cameraUmap.position.y -= 5
          break
        case 65: // A LEFT
          this.cameraPca.position.x += 5
          this.cameraUmap.position.x += 5
          break
        case 68: // D RIGHT
          this.cameraPca.position.x -= 5
          this.cameraUmap.position.x -= 5
          break
        case 81: // Q IN
          this.cameraPca.position.z += 1
          this.cameraUmap.position.z += 1
          break
        case 69: // E OUT
          this.cameraPca.position.z -= 1
          this.cameraUmap.position.z -= 1
          break
        // ROTATION
        case 74: // J LEFT
          this.cameraPca.rotation.z -= 0.01
          this.cameraUmap.rotation.z -= 0.01
          break
        case 76: // L RIGHT
          this.cameraPca.rotation.z += 0.01
          this.cameraUmap.rotation.z += 0.01
          break
        case 73: // I UP
          this.cameraPca.rotation.x -= 0.01
          this.cameraUmap.rotation.x -= 0.01
          break
        case 75: // K DOWN
          this.cameraPca.rotation.x += 0.01
          this.cameraUmap.rotation.x += 0.01
          break
        case 85: // U DOWN
          this.cameraPca.rotation.y += 0.01
          this.cameraUmap.rotation.y += 0.01
          break
        case 79: // O DOWN
          this.cameraPca.rotation.y -= 0.01
          this.cameraUmap.rotation.y -= 0.01
          break
      }
    },
    channelMixer (channelA, channelB, channelC, amount) {
      const a = channelA * (1 - amount)
      const b = channelB * amount
      const c = channelC * (1 + amount)
      return parseInt(a + b + c, 10)
    },
    blendColors (colorA, colorB, colorC, amount = 0.33) {
      return [0, 1, 2].map(i => this.channelMixer(colorA[i], colorB[i], colorC[i], amount))
    }
  },
  mounted () {
    this.$nextTick(function () {
      if (this.isCeleba) {
        if (!this.isColored) {
          this.getImages(0, '000')
        } else {
          this.getColoredImages(0, '000')
        }
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
