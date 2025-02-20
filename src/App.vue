<template>
  <div class="container">
    <!-- Left side: text area -->
    <div v-show="showLeftPanel" class="left-pane">
      <textarea v-model="script" class="big-textarea" placeholder="Type here..."></textarea>
      <div class="btn-container">
        <button @click="runScript" class="action-btn">Run</button>
        <button v-show="!desktopView" @click="showLeftPanel = false" class="action-btn float-right">
          Hide
        </button>
      </div>
    </div>

    <!-- Right side: the Three.js canvas + buttons -->
    <div class="right-pane" ref="threeCanvas">
      <button
        v-show="!desktopView && !showLeftPanel"
        @click="showLeftPanel = true"
        class="action-btn float-left"
      >
        Show
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import axios from 'axios'
import { ref, shallowRef, onMounted, watch } from 'vue'
import * as THREE from 'three'
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader.js'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

const script = ref<string>(
  JSON.stringify(
    [
      {
        name: 'Box',
        id: '60b87694-eaa4-4f17-8740-42355979dd71',
        inputs: [
          {
            name: 'X',
            type: 'number',
            value: 1,
          },
          {
            name: 'Y',
            type: 'number',
            value: 1,
          },
          {
            name: 'Z',
            type: 'number',
            value: 1,
          },
        ],
      },
      {
        name: 'Vec',
        id: 'd0d21888-51e1-4aa9-ba6a-2da32d658f46',
        inputs: [
          {
            name: 'X',
            type: 'number',
            value: 0,
          },
          {
            name: 'Y',
            type: 'number',
            value: 2,
          },
          {
            name: 'Z',
            type: 'number',
            value: 2,
          },
        ],
      },
      {
        name: 'Move',
        inputs: [
          {
            name: 'T',
            type: 'link',
            value: 'd0d21888-51e1-4aa9-ba6a-2da32d658f46.V',
          },
          {
            name: 'G',
            type: 'link',
            value: '60b87694-eaa4-4f17-8740-42355979dd71.B',
          },
        ],
      },
    ],
    null,
    2,
  ),
)
const desktopView = ref<Boolean>(true)
const showLeftPanel = ref<Boolean>(true)

// // Three.js objects
const threeCanvas = ref(null) // Reference to the canvas container
const scene = shallowRef(new THREE.Scene())
const camera = shallowRef(new THREE.PerspectiveCamera(75, window.innerWidth / 400, 0.1, 1000))
const renderer = shallowRef(null)
const loader = shallowRef(null)

const meshGroup = shallowRef(new THREE.Group())
const outlineGroup = shallowRef(new THREE.Group()) // Group to hold outline edges

const controls = shallowRef(null) // OrbitControls instance

// Animation loop
const animate = () => {
  requestAnimationFrame(animate)
  if (renderer.value && scene.value && camera.value) {
    controls.value.update() // Update OrbitControls
    renderer.value.render(scene.value, camera.value)
  }
}

// Initialize Three.js scene
const initScene = () => {
  const size = 50 // overall grid size
  const divisions = 50 // number of divisions
  const gridColor = 0xc9c9c9 // a light gray for the main grid lines
  const centerLineColor = 0xababab // slightly darker (but still light) for center lines
  const gridHelper = new THREE.GridHelper(size, divisions, centerLineColor, gridColor)
  scene.value.add(gridHelper)

  const axesHelper = new THREE.AxesHelper(10)
  scene.value.add(axesHelper)

  // Renderer setup
  renderer.value = new THREE.WebGLRenderer({ antialias: true })
  renderer.value.setSize(window.innerWidth, window.innerHeight)
  renderer.value.shadowMap.enabled = true // Enable shadow maps
  renderer.value.shadowMap.type = THREE.PCFSoftShadowMap
  threeCanvas.value.appendChild(renderer.value.domElement)

  // Set background to white
  scene.value.background = new THREE.Color(0xffffff)

  camera.value.position.set(20, 20, 30)

  // OrbitControls setup
  controls.value = new OrbitControls(camera.value, renderer.value.domElement)
  controls.value.enableDamping = true // Smooth rotation
  controls.value.dampingFactor = 0.05

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 1) // Soft ambient light
  scene.value.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 2)
  directionalLight.position.set(150, 200, 100)
  directionalLight.castShadow = true
  directionalLight.shadow.mapSize.width = 4096 // Shadow map resolution
  directionalLight.shadow.mapSize.height = 4096
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

  // 5) Prepare STLLoader
  loader.value = new STLLoader()

  // Start animation
  animate()

  // Resize window
  onWindowResize()
}

/**
 * Calls the API, retrieves a Base64-encoded STL, and updates the 3D scene.
 */
async function runScript() {
  try {
    // 1) Call the API
    const payload = {
      script: script.value,
      format: 'STL',
    }
    const res = await axios.post<StlApiResponse>(
      'https://py-occ-server-1056113226980.us-central1.run.app/run-script',
      payload,
    )

    if (!res.data.outputs[0].value) {
      throw new Error('No stl_base64 field in server response')
    }

    // 2) Decode Base64 -> ArrayBuffer
    const stlB64 = res.data.outputs[0].value
    const raw = window.atob(stlB64)
    const rawLength = raw.length
    const arrayBuffer = new Uint8Array(new ArrayBuffer(rawLength))
    for (let i = 0; i < rawLength; i++) {
      arrayBuffer[i] = raw.charCodeAt(i)
    }

    // 3) Parse STL into geometry
    if (!loader.value) return
    const geometry = loader.value.parse(arrayBuffer.buffer)

    // 4) Create or replace the mesh
    const material = new THREE.MeshLambertMaterial({ color: 0xc892e0 })
    material.castShadow = true // Allow table to cast shadows

    const newMesh = new THREE.Mesh(geometry, material)
    newMesh.castShadow = true // Allow legs to cast shadows

    if (scene.value) {
      if (meshGroup.value) {
        scene.value.remove(meshGroup.value)
      }
      if (outlineGroup.value) {
        scene.value.remove(outlineGroup.value)
      }
    }

    const meshGroupLocal = new THREE.Group()
    const outlineGroupLocal = new THREE.Group()

    meshGroupLocal.add(newMesh)

    // Create outline for mesh
    const meshEdges = new THREE.EdgesGeometry(newMesh.geometry)
    const meshOutline = new THREE.LineSegments(
      meshEdges,
      new THREE.LineBasicMaterial({ color: 0x333333 }),
    )
    outlineGroupLocal.add(meshOutline)

    meshGroup.value = meshGroupLocal
    outlineGroup.value = outlineGroupLocal
    scene.value.add(meshGroup.value)
    scene.value.add(outlineGroup.value)

    // Compute bounding box to center the model
    geometry.computeBoundingBox()
    if (geometry.boundingBox) {
      const bbox = geometry.boundingBox
      const size = new THREE.Vector3().subVectors(bbox.max, bbox.min)
      const center = new THREE.Vector3().addVectors(bbox.min, bbox.max).multiplyScalar(0.5)

      const dist = Math.max(size.x, size.y, size.z) * 1.5

      // Adjust camera distance
      camera.value?.position.set(center.x - dist, center.y + dist, center.z + dist)
      camera.value?.lookAt(center)

      controls.value.target.copy(center)
      controls.value.update()
    }
  } catch (err) {
    console.error('Error loading STL:', err)
  }
}

// Handle window resizing
const onWindowResize = () => {
  // Only measure the 'right-pane'
  if (!threeCanvas.value || !renderer.value || !camera.value) return

  const width = threeCanvas.value.clientWidth
  const height = threeCanvas.value.clientHeight
  camera.value.aspect = width / height
  camera.value.updateProjectionMatrix()
  renderer.value.setSize(width, height)

  if (window.innerWidth > 800) {
    desktopView.value = true
    showLeftPanel.value = true
  } else {
    desktopView.value = false
  }
}

// Mount the scene
onMounted(() => {
  initScene()
  window.addEventListener('resize', onWindowResize)
  runScript()
})
</script>

<style>
#app {
  overflow: hidden;
  width: 100vw;
  height: 100vh;
  margin: 0;
}
</style>

<style scoped>
/* Container holds two panes side by side by default */
.container {
  display: flex;
  flex-direction: row;
  width: 100vw;
  height: 100vh;
  margin: 0;
}

/* Left pane: 50% width on desktop */
.left-pane {
  width: 50%;
  height: 100%;
  background-color: #fafafa;
  display: flex;
  flex-direction: column;
  /* gap: 1rem; */
  /* padding: 1rem; */
  border: 1px solid #ccc;
}

/* Right pane: 50% width on desktop */
.right-pane {
  width: 50%;
  height: 100%;
  position: relative; /* for absolutely positioned buttons */
  background-color: #ffffff;
}

/* A large text area filling left pane */
.big-textarea {
  flex: 1;
  width: 100%;
  resize: none;
  font-size: 16px;
  padding: 10px;
  box-sizing: border-box;
  border: none;
}
.big-textarea:focus {
  /* outline: 1rem solid #fffaad;  */
  outline: none; /* Removes the default outline */
  box-shadow: none; /* Removes any box shadow highlight */
}

.btn-container {
  position: relative;
  display: flex;
  justify-content: center;
  padding: 1rem;
}

.action-btn {
  padding: 5px 10px;
  cursor: pointer;
}

.float-right {
  position: absolute;
  right: 1rem;
}

.float-left {
  position: absolute;
  left: 1rem;
  bottom: 1rem;
}

/* ======================
   MEDIA QUERY:
   For screens < 800px
   - Right pane is full width
   - Left pane becomes an overlay
   ====================== */
@media (max-width: 800px) {
  .container {
    position: relative;
    flex-direction: column; /* or row, but we want the 3D behind the entire container */
    width: 100vw;
    height: 100vh;
  }

  /* Right pane: full bleed */
  .right-pane {
    width: 100%;
    height: 100%;
  }

  /* Left pane: absolute overlay */
  .left-pane {
    position: absolute;
    top: 0;
    left: 0;
    width: 400px; /* fixed width overlay */
    height: 100%;
    box-shadow: 2px 2px 8px rgba(0, 0, 0, 0.25);
    z-index: 999;
  }
}
</style>
