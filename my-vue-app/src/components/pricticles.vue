<template>
  <canvas ref="canvas" id="particles"></canvas>
</template>

<script setup lang="ts">
import * as THREE from 'three'
import { ref, onMounted } from 'vue'
defineOptions({
  name: 'Particles'
})
let canvas = ref<HTMLCanvasElement>()
const scene = ref<THREE.Scene>()
const camera = ref<THREE.PerspectiveCamera>()
const renderer = ref<THREE.WebGLRenderer>()
const geometry = ref<THREE.BufferGeometry>()
const count = ref<number>(500)
const positions = ref<Float32Array>()
const smokeParticles = []
const clock = new THREE.Clock()

const createSmoke = () => {
  const particleCount = 1000
  const particlesGeometry = new THREE.BufferGeometry()
  const positions = new Float32Array(particleCount * 3)
  const sizes = new Float32Array(particleCount)
  const colors = new Float32Array(particleCount * 3)
  for (let i = 0; i < particleCount; i++) {
    positions[i * 3] = Math.random() * 3
    positions[i * 3 + 1] = Math.random() * 3
    positions[i * 3 + 2] = Math.random() * 3
    sizes[i] = Math.random() * 0.5
    colors[i * 3] = Math.random()
    colors[i * 3 + 1] = Math.random()
    colors[i * 3 + 2] = Math.random()
  }
  particlesGeometry.setAttribute(
    'position',
    new THREE.BufferAttribute(positions, 3)
  )
  particlesGeometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1))
  particlesGeometry.setAttribute('color', new THREE.BufferAttribute(colors, 3))
  const material = new THREE.PointsMaterial({
    size: 0.2,
    sizeAttenuation: true,
    transparent: true,
    opacity: 0.6,
    depthWrite: false,
    blending: THREE.AdditiveBlending,
    vertexColors: true
  })
  const partticlesSystem = new THREE.Points(particlesGeometry, material)
  scene.value.add(partticlesSystem)
  smokeParticles.push({
    system: partticlesSystem,
    geometry: particlesGeometry,
    positions: positions
  })
}

const animate = () => {
  if (!renderer || !scene || !camera) return

  const elapsedTime = clock.getElapsedTime()

  smokeParticles.forEach(particle => {
    const positions = particle.positions
    for (let i = 0; i < positions.length; i += 3) {
      positions[i + 1] += 0.01 * Math.random()

      if (positions[i + 1] > 10) {
        positions[i + 1] = -5
      }

      positions[i] += 0.002 * (Math.random() - 0.5)
      positions[i + 2] += 0.002 * (Math.random() - 0.5)
    }

    particle.system.geometry.attributes.position.needsUpdate = true

    particle.system.rotation.y += 0.0005
  })

  requestAnimationFrame(animate)
  renderer.value.render(scene.value, camera.value)
}

const init = () => {
  const dpr = window.devicePixelRatio || 1
  canvas.value.width = window.innerWidth * dpr
  canvas.value.height = window.innerHeight * dpr

  scene.value = new THREE.Scene()
  camera.value = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  )
  renderer.value = new THREE.WebGLRenderer()
  renderer.value.setSize(window.innerWidth, window.innerHeight)
  document.body.appendChild(renderer.value.domElement)
}
onMounted(() => {
  init()
  createSmoke()
  animate()
})
</script>
<style>
body {
  margin: 0;
  padding: 0;
}
html,
body {
  overflow: hidden;
}
canvas {
  position: fixed;
  top: 0;
  left: 0;
  outline: none;
  background-color: #000;
}
</style>
