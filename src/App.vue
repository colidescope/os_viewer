<template>
  <div id="app">
    <div class="controls">
      <label>
        Width:
        <input type="range" v-model="tableWidth" min="50" max="300" />
        {{ tableWidth }}
      </label>
      <label>
        Length:
        <input type="range" v-model="tableDepth" min="50" max="300" />
        {{ tableDepth }}
      </label>
    </div>
    <div ref="threeCanvas" class="three-container"></div>
  </div>
</template>

<script setup>
import { ref, shallowRef, onMounted, watch } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

// Reactive data for table dimensions
const tableWidth = ref(100)
const tableDepth = ref(100)

// Three.js objects
const threeCanvas = ref(null) // Reference to the canvas container
const scene = shallowRef(new THREE.Scene())
const camera = shallowRef(new THREE.PerspectiveCamera(75, window.innerWidth / 400, 0.1, 1000))
const renderer = shallowRef(null)
const table = shallowRef(null)
const outlineGroup = shallowRef(new THREE.Group()) // Group to hold outline edges
const controls = shallowRef(null) // OrbitControls instance

// Function to create the table
const createTable = () => {
  if (table.value) {
    scene.value.remove(table.value) // Remove the existing table
  }

  if (outlineGroup.value) {
    scene.value.remove(outlineGroup.value) // Remove the existing outlines
  }

  const material = new THREE.MeshStandardMaterial({ color: 0x8b4513 })
  material.castShadow = true // Allow table to cast shadows
  const tableGroup = new THREE.Group()
  const outlineGroupLocal = new THREE.Group()

  // Tabletop
  const tabletopGeometry = new THREE.BoxGeometry(tableWidth.value, 10, tableDepth.value)
  const tabletop = new THREE.Mesh(tabletopGeometry, material)
  tabletop.position.y = 55
  tabletop.castShadow = true // Allow tabletop to cast shadows
  tableGroup.add(tabletop)

  // Create outline for tabletop
  const tabletopEdges = new THREE.EdgesGeometry(tabletopGeometry)
  const tabletopOutline = new THREE.LineSegments(
    tabletopEdges,
    new THREE.LineBasicMaterial({ color: 0x000000 }),
  )
  tabletopOutline.position.y = 55
  outlineGroupLocal.add(tabletopOutline)

  // Table legs
  const legGeometry = new THREE.BoxGeometry(10, 50, 10)
  const legPositions = [
    [-tableWidth.value / 2 + 5, 25, -tableDepth.value / 2 + 5],
    [tableWidth.value / 2 - 5, 25, -tableDepth.value / 2 + 5],
    [-tableWidth.value / 2 + 5, 25, tableDepth.value / 2 - 5],
    [tableWidth.value / 2 - 5, 25, tableDepth.value / 2 - 5],
  ]

  legPositions.forEach(([x, y, z]) => {
    const leg = new THREE.Mesh(legGeometry, material)
    leg.position.set(x, y, z)
    leg.castShadow = true // Allow legs to cast shadows
    tableGroup.add(leg)

    // Create outline for each leg
    const legEdges = new THREE.EdgesGeometry(legGeometry)
    const legOutline = new THREE.LineSegments(
      legEdges,
      new THREE.LineBasicMaterial({ color: 0x000000 }),
    )
    legOutline.position.set(x, y, z)
    outlineGroupLocal.add(legOutline)
  })

  table.value = tableGroup
  outlineGroup.value = outlineGroupLocal
  scene.value.add(table.value)
  scene.value.add(outlineGroup.value)
}

// Animation loop
const animate = () => {
  requestAnimationFrame(animate)
  controls.value.update() // Update OrbitControls
  renderer.value.render(scene.value, camera.value)
}

// Initialize Three.js scene
const initScene = () => {
  // Renderer setup
  renderer.value = new THREE.WebGLRenderer({ antialias: true })
  renderer.value.setSize(window.innerWidth, window.innerHeight)
  renderer.value.shadowMap.enabled = true // Enable shadow maps
  renderer.value.shadowMap.type = THREE.PCFSoftShadowMap
  threeCanvas.value.appendChild(renderer.value.domElement)

  // Set background to white
  scene.value.background = new THREE.Color(0xffffff)

  // Camera setup
  camera.value.position.set(200, 200, 300)

  // OrbitControls setup
  controls.value = new OrbitControls(camera.value, renderer.value.domElement)
  controls.value.enableDamping = true // Smooth rotation
  controls.value.dampingFactor = 0.05

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 2) // Soft ambient light
  scene.value.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(150, 200, 100)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 2048 // Shadow map resolution
  directionalLight.shadow.mapSize.height = 2048
  directionalLight.shadow.camera.near = 0.5
  directionalLight.shadow.camera.far = 500
  directionalLight.shadow.camera.left = -200
  directionalLight.shadow.camera.right = 200
  directionalLight.shadow.camera.top = 200
  directionalLight.shadow.camera.bottom = -200
  scene.value.add(directionalLight)

  // Ground Plane
  const groundMaterial = new THREE.ShadowMaterial({ opacity: 0.3 })
  const groundGeometry = new THREE.PlaneGeometry(500, 500)
  const ground = new THREE.Mesh(groundGeometry, groundMaterial)
  ground.rotation.x = -Math.PI / 2 // Make the plane horizontal
  ground.position.y = 0
  ground.receiveShadow = true // Allow ground to receive shadows
  scene.value.add(ground)

  // Create initial table
  createTable()

  // Start animation
  animate()
}

// Handle window resizing
const onWindowResize = () => {
  camera.value.aspect = window.innerWidth / window.innerHeight
  camera.value.updateProjectionMatrix()
  renderer.value.setSize(window.innerWidth, window.innerHeight)
}

// Watch for changes in table dimensions
watch([tableWidth, tableDepth], createTable)

// Mount the scene
onMounted(() => {
  initScene()
  window.addEventListener('resize', onWindowResize)
  onWindowResize()
})
</script>

<style>
#app {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  width: 100vw;
  height: 100vh;
  margin: 0;
}

.controls {
  margin-top: 4rem;
  z-index: 50;
}

.controls label {
  color: black;
  display: block;
  margin: 10px 0;
}

.three-container {
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
}
</style>
