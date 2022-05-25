<script setup>
import { onMounted } from 'vue'
import * as THREE from 'three'
onMounted(() => {
  const scene = new THREE.Scene()

  /**
   * Camera
   **/
  const frustumSize = 30
  const aspect = window.innerWidth / window.innerHeight
  const camera = new THREE.OrthographicCamera(
    (frustumSize * aspect) / -2,
    (frustumSize * aspect) / 2,
    frustumSize / 2,
    frustumSize / -2,
    1,
    1000
  )

  camera.position.set(10, 10, 20)
  camera.lookAt(scene.position)

  /**
   * Renderer
   **/
  const renderer = new THREE.WebGLRenderer({ antialias: true })

  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setClearColor(0xffffff, 1.0)
  renderer.shadowMap.enabled = true

  document.getElementById('three').appendChild(renderer.domElement)

  /**
   * Geometries
   **/
  const loader = new THREE.TextureLoader()
  const imgMaterial = new THREE.MeshStandardMaterial({
    map: loader.load(
      'http://ddragon.leagueoflegends.com/cdn/6.8.1/img/map/map11.png'
    )
  })
  const material = new THREE.MeshStandardMaterial({ color: 0x333333 })

  const cubeMaterial = []
  for (let i = 0; i < 6; i++) {
    cubeMaterial.push(i === 4 ? imgMaterial : material)
  }

  const maps = []
  for (let i = 0; i < 5; i++) {
    const geometry = new THREE.BoxGeometry(10, 10, .4)
    const map = new THREE.Mesh(geometry, cubeMaterial)
    map.castShadow = true
    map.receiveShadow = true
    map.position.set(0, 0, -i * 5)
    // custom attributes
    map.isMap = true
    map.isHovered = false
    maps.push(map)
  }
  maps.forEach(map => scene.add(map))

  /**
   * Lights
   **/
  let light = new THREE.DirectionalLight(0xffffff, 0.5)

  light.position.set(0, 10, 10)
  light.castShadow = true
  light.shadow.camera.top = 15 // default
  light.shadow.camera.near = 0.5 // default
  light.shadow.camera.far = 40 // default
  scene.add(light)
  
  light = new THREE.AmbientLight(0xffffff, 0.5)
  scene.add(light)

  /**
   * Helpers
   **/
  scene.add(new THREE.AxesHelper(50))

  /**
   * Render!
   **/
  function animate() {
    requestAnimationFrame(animate)
    // maps.forEach((map, index) => map.rotation.z += (index + 1) * 0.005)
    renderer.render(scene, camera)
  }
  animate()

  const sphere = new THREE.Mesh(new THREE.SphereGeometry(0.5, 16, 8), material)
  sphere.position.z = 5
  scene.add(sphere)

  const raycaster = new THREE.Raycaster()
  const mouse = { position: new THREE.Vector2() }
  document.getElementById('three').addEventListener('mousemove', event => {
    mouse.position.x =  (event.clientX / window.innerWidth) * 2 - 1
    mouse.position.y = -(event.clientY / window.innerHeight) * 2 + 1

    raycaster.setFromCamera(mouse.position, camera)
    const found = raycaster.intersectObjects(scene.children)
    if (found.length && found[0].object.isMap) {
      sphere.position.set(...found[0].point.toArray())
      const foundMap = found[0].object
      maps.forEach(map => {
        if (map !== foundMap) {
          if (map.isHovered) {
            map.position.x -= 0.5
          }
          map.isHovered = false
        }
      })
      if (!foundMap.isHovered) {
        foundMap.isHovered = true
        foundMap.position.x += 0.5
      }
    }
  })
})
</script>

<template>
  <div id="three"></div>
</template>
