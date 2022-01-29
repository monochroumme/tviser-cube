<template>
  <div class="the-cube" id="cube" ref="cube">
    <button class="the-cube__btn" @click="handleFaceVisibility">HANDLE FACE VISIBILITY</button>
  </div>
</template>

<script>
import * as THREE from 'three'
// import * as math from 'mathjs'

export default {
  name: 'TheCube',

  data () {
    return {
      camera: null,
      scene: null,
      renderer: null,
      mesh: null,
      raycaster: null,
      canvas: null,
      allCompanies: [
        {
          filename: 'gazprom.jpg',
          link: 'https://www.gazprom.com/'
        },
        {
          filename: 'hyundai.jpg',
          link: 'https://www.hyundai.com/worldwide/en/'
        },
        {
          filename: 'lg.jpg',
          link: 'https://lg.com'
        },
        {
          filename: 'nestle.jpg',
          link: 'https://www.nestle.com/'
        },
        {
          filename: 'pikabu.jpg',
          link: 'https://pikabu.ru/'
        },
        {
          filename: 'mothercare.jpg',
          link: 'https://www.mothercare.az/'
        }
      ],
      visibleCompanies: null,
      arrowHelper: null,
      cameraDirection: new THREE.Vector3()
    }
  },

  async mounted () {
    this.setVisibleCompanies()
    await this.init()
    this.animate()
    this.onResize()

    window.addEventListener('resize', this.onResize)
    window.addEventListener('mouseup', this.onDocumentMouseDown)
    window.addEventListener('mousemove', this.onMouseMove)
  },

  methods: {
    setVisibleCompanies () {
      this.visibleCompanies = this.allCompanies.slice(0, 6)
    },

    onResize () {
      this.$refs.cube.style.width = this.$refs.cube.style.height =
        `${window.innerWidth > window.innerHeight ? window.innerHeight : window.innerWidth}px`
    },

    loadTextures (textureURLs, callback) {
      let loaded = 0
      function loadedOne () {
        loaded++
        if (callback && loaded === textureURLs.length) {
          for (let i = 0; i < textureURLs; i++) { textures[i].needsUpdate = true }
          callback()
        }
      }
      const textures = []
      for (let i = 0; i < textureURLs.length; i++) {
        const tex = new THREE.TextureLoader().load(textureURLs[i], undefined, loadedOne)
        tex.name = textureURLs[i]
        textures.push(tex)
      }
      return textures
    },

    init () {
      const frustumSize = 1024
      const boxSize = 400
      const aspect = 1
      const textureUrls = this.visibleCompanies.slice(0, 6).map(c => c.filename)
      const textures = this.loadTextures(textureUrls)
      const materials = []
      for (let i = 0; i < 6; i++) {
        materials.push(new THREE.MeshBasicMaterial({
          color: 'white',
          map: textures[i]
        }))
      }
      this.raycaster = new THREE.Raycaster()
      this.camera = new THREE.OrthographicCamera(frustumSize * aspect / -2, frustumSize * aspect / 2, frustumSize / 2, frustumSize / -2, 1, 1000)
      this.camera.position.z = 500
      this.scene = new THREE.Scene()
      const geometry = new THREE.BoxBufferGeometry(boxSize, boxSize, boxSize)
      this.mesh = new THREE.Mesh(geometry, materials)
      this.arrowHelper = new THREE.ArrowHelper(this.mesh.getWorldDirection(new THREE.Vector3()), this.mesh.position, 500, 0xff0000)
      this.scene.add(this.mesh)
      this.scene.add(this.arrowHelper)
      this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.setSize(boxSize, boxSize)
      document.getElementById('cube').appendChild(this.renderer.domElement)
      this.canvas = this.renderer.domElement
    },

    animate () {
      requestAnimationFrame(this.animate)
      if (this.mesh && this.renderer && this.scene && this.camera) {
        this.mesh.rotation.x += 0.003
        this.mesh.rotation.y += 0.004
        this.arrowHelper.setDirection(this.mesh.getWorldDirection(new THREE.Vector3()))
        this.renderer.render(this.scene, this.camera)
        // this.handleFaceVisibility()
      }
    },

    onDocumentMouseDown (event) {
      const pointer = new THREE.Vector2(
        (event.clientX / window.innerWidth) * 2 - 1,
        -(event.clientY / window.innerHeight) * 2 + 1
      )

      this.raycaster.setFromCamera(pointer, this.camera)

      const intersects = this.raycaster.intersectObject(this.mesh)
      if (intersects.length > 0) {
        const materialIndex = intersects[0]?.face.materialIndex
        const companyFilename = intersects[0]?.object?.material[materialIndex]?.map?.name
        const link = this.allCompanies.find(c => c.filename === companyFilename)?.link
        if (link) {
          this.openLink(link)
        }
      }
    },

    onMouseMove (event) {
      const pointer = new THREE.Vector2()

      pointer.x = (event.clientX / window.innerWidth) * 2 - 1
      pointer.y = -(event.clientY / window.innerHeight) * 2 + 1

      this.raycaster.setFromCamera(pointer, this.camera)

      const intersects = this.raycaster.intersectObject(this.mesh)

      if (intersects.length > 0 && this.canvas) {
        this.canvas.style.transform = 'scale(1.1)'
      } else {
        this.canvas.style.transform = 'scale(1)'
      }
    },

    openLink (url) {
      window.open(url, '_blank').focus()
    },

    dotProduct (v1, v2) {
      return new THREE.Vector3().copy(v1).dot(v2)
    },

    // n is camera direction
    // p0 is camera position
    // l0 is mesh position
    // l is ray direction
    intersectsPlane (n, p0, l0, l) {
      n = n.clone().normalize()
      p0 = p0.clone().normalize()
      l0 = l0.clone().normalize()
      l = l.clone().normalize()

      // console.log(n, p0, l0, l)

      const denom = this.dotProduct(n, l)
      // console.log('denom', denom)
      if (denom > 1e-6) {
        return true
        // const p0l0 = new THREE.Vector3().copy(p0).sub(l0) // p0 - l0
        // const t = this.dotProduct(p0l0, n) / denom
        // console.log('t', t)
        // return (t >= 0)
      }

      return false
    },

    handleFaceVisibility () {
      if (this.camera && this.mesh && this.cameraDirection) {
        // camera's position
        const cameraPosition = this.camera.position
        // camera's direction
        const cameraDirection = this.camera?.getWorldDirection(this.cameraDirection)

        if (cameraPosition && cameraDirection) {
          // project rays from the middle of the cube in order and determine
          // whether the rays intersect with the plane (using math, not actual rays)
          const meshDirection = this.mesh.getWorldDirection(new THREE.Vector3())
          const rays = [
            new THREE.Vector3().copy(meshDirection).negate(), // index 0
            new THREE.Vector3().copy(meshDirection) // index 5
          ]

          console.log(this.intersectsPlane(cameraDirection, cameraPosition, this.mesh.position, rays[1]))
        }
      }
    }
  },

  beforeDestroy () {
    window.removeEventListener('resize', this.onResize)
    window.removeEventListener('mouseup', this.onDocumentMouseDown)
    window.removeEventListener('mousemove', this.onMouseMove)
    if (this.mesh && this.mesh.dispose) {
      this.mesh.dispose()
    }
  }
}
</script>

<style lang="scss">
@import '@/assets/styles/elements/cube';
</style>
