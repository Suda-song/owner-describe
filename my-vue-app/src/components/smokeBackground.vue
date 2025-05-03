<script setup lang="ts">
import { defineComponent, onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
defineComponent({
  name: 'SmokeBackground'
})
const canvasRef = ref<HTMLCanvasElement>()
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let smokeParticles: THREE.Mesh[] = []
let clock: THREE.Clock

const smokeVertexShader = `
varying vec2 vUv;
varying vec3 vPosition;
void main() {
  vUv = uv;
  vPosition = position;
  gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(position, 1.0);
}
`
const smokeFragmentShader = `
uniform float time;
uniform sampler2D smokeTexture;
varying vec2 vUv;
varying vec3 vPosition;

// 噪声函数
float random(vec2 st) {
    return fract(sin(dot(st.xy, vec2(12.9898,78.233))) * 43758.5453123);
}

float noise(vec2 st) {
    vec2 i = floor(st);
    vec2 f = fract(st);

    float a = random(i);
    float b = random(i + vec2(1.0, 0.0));
    float c = random(i + vec2(0.0, 1.0));
    float d = random(i + vec2(1.0, 1.0));

    vec2 u = f * f * (3.0 - 2.0 * f);

    return mix(a, b, u.x) +
            (c - a)* u.y * (1.0 - u.x) +
            (d - b) * u.x * u.y;
}

// 更复杂的FBM函数，创建更细致的云纹理
float fbm(vec2 uv) {
    float value = 0.0;
    float amplitude = 0.5;
    float frequency = 2.0;
    
    // 使用更多层噪声
    for (int i = 0; i < 6; i++) {
        value += amplitude * noise(uv * frequency);
        frequency *= 2.1; // 稍微改变频率倍数，增加不规则性
        amplitude *= 0.5;
        
        // 扭曲各层之间的关系
        uv += vec2(value * 0.1, value * 0.08);
    }
    return value;
}

// 添加扭曲函数，使云形状更有机
vec2 distort(vec2 uv, float time) {
    vec2 distortion;
    distortion.x = fbm(uv + vec2(time * 0.01, 0.0)) * 0.2;
    distortion.y = fbm(uv + vec2(0.0, time * 0.015)) * 0.2;
    return distortion;
}

void main() {
    vec2 uv = vUv;
    float noiseTime = time * 0.03;
    
    // 扭曲UV坐标，创造不规则边缘
    vec2 distortedUV = uv + distort(uv, noiseTime) * 0.15;
    
    // 水平方向的距离计算
    float centerX = 0.5;
    float horizontalDist = abs(distortedUV.x - centerX);

    // 云的形状控制参数
    float minWidth = 0.08;        // 中心区域最小宽度
    float expandStart = 0.2;      // 开始扩展的位置(屏幕20%)
    float expansionPower = 2.5;   // 扩展速率指数
    float maxWidth = 0.5;         // 最大扩展宽度
    float cloudWidth;

    if (horizontalDist < expandStart) {  // 这里应使用expandStart而非edgeDistance
        // 中间区域 - 固定较窄宽度
        cloudWidth = minWidth;
    } else {
        // 两侧区域 - 使用指数函数快速扩展
        float t = (horizontalDist - expandStart) / (0.5 - expandStart); 
        t = pow(t, expansionPower); 
        cloudWidth = minWidth + t * (maxWidth - minWidth);
    }

    // 这部分变量与后面重复，应移除或合并
    // float yCenter = 0.5;
    // float verticalDist = abs(distortedUV.y - yCenter);
    float verticalDist = abs(distortedUV.y - 0.5);
    float verticalFade = smoothstep(cloudWidth, cloudWidth * 0.8, verticalDist);

    // 水平方向上的额外淡出，确保边缘平滑
    float horizontalEdgeFade = smoothstep(0.48, 0.4, horizontalDist);

    // 组合淡出效果
    float shapeFade = verticalFade * horizontalEdgeFade;

    // 替换原来的horizontalFade
    float horizontalFade = shapeFade;
    // 创建多层变化的噪声，使云的内部结构更复杂
    float noise1 = fbm(distortedUV * 2.5 + noiseTime * 0.1);
    float noise2 = fbm(distortedUV * 4.0 - noiseTime * 0.05);
    float noise3 = fbm(distortedUV * 8.0 + noiseTime * 0.2) * 0.5;
    
    // 不同比例混合噪声
    float combinedNoise = 
        noise1 * 0.5 + 
        noise2 * 0.3 + 
        noise3 * 0.2;
    
    // 为云的形状添加更多扰动
    float cloudShape = fbm(distortedUV * 1.5 + fbm(distortedUV * 3.0) * 0.2);
    
    // 添加一些小尺度的细节
    float details = fbm(distortedUV * 16.0 + combinedNoise * 0.1);
    
    // 创建云的基本形状 - 使用更加不规则的高度
    float baseHeight = 0.12 + cloudShape * 0.06;
    float vertDistortion = fbm(vec2(distortedUV.x * 4.0, noiseTime * 0.2)) * 0.06;
    
    // 通过扭曲y坐标来创建不规则的云顶和云底
    float yCenter = 0.5 + sin(distortedUV.x * 5.0 + noiseTime) * 0.02;
    float yDist = abs(distortedUV.y - yCenter) - vertDistortion; 
    
    // 计算垂直方向的衰减
    float verticalShape = smoothstep(baseHeight + 0.02, baseHeight - 0.08, yDist);
    
    // 使用非线性公式计算密度
    float density = verticalShape * (0.4 + combinedNoise * 0.6);
    
    // 添加小尺度的密度变化
    density *= (0.7 + details * 0.3);
    
    // 添加密度变化，使云更"蓬松"，有空隙
    density *= (0.7 + 0.3 * fbm(distortedUV * 12.0 + noiseTime * 0.3));
    
    // 应用水平方向的淡出，并使用非线性函数使其更自然
    density *= horizontalFade;
    
    // 额外的调整，使密度在水平边缘处更加不均匀
    float edgeNoise = fbm(vec2(distortedUV.y * 3.0, distortedUV.x * 2.0 + noiseTime * 0.1));
    float edgeFactor = 1.0 - smoothstep(0.25, 0.4, horizontalDist);
    density *= mix(edgeNoise * 0.5 + 0.5, 1.0, edgeFactor);
    
    // 云的颜色渐变 - 添加更多阴影变化
    vec3 cloudColor = vec3(0.98, 0.98, 1.0);
    vec3 cloudShadowColor = vec3(0.75, 0.82, 0.95);
    
    // 使用复杂的噪声函数创建云内部的明暗变化
    float brightnessFactor = fbm(distortedUV * 3.5 + vec2(noiseTime * 0.05, -noiseTime * 0.03));
    brightnessFactor = brightnessFactor * 0.6 + 0.4; // 调整范围，使颜色不会太暗
    
    vec3 finalCloudColor = mix(cloudShadowColor, cloudColor, brightnessFactor);
    
    // 调整透明度，增加不透明度的变化
    float alpha = density * (0.6 + combinedNoise * 0.4);
    
    // 额外的透明度调整，减少实心感
    alpha *= (0.7 + 0.3 * sin(distortedUV.x * 10.0 + distortedUV.y * 8.0 + noiseTime * 0.2));
    
    // 最终输出
    gl_FragColor = vec4(finalCloudColor, alpha * 0.85); // 略微降低整体不透明度
}
`
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

const init = () => {
  if (!canvasRef.value) return

  scene = new THREE.Scene()
  camera = new THREE.PerspectiveCamera(
    75,
    sizes.width / sizes.height,
    0.1,
    1000
  )
  camera.position.z = 15
  scene.add(camera)
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    alpha: true
  })
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  clock = new THREE.Clock()
  createSmoke()
}

const animate = () => {
  if (!renderer || !scene || !camera) return

  const elapsedTime = clock.getElapsedTime()

  smokeParticles.forEach(smoke => {
    const material = smoke.userData.material as THREE.ShaderMaterial
    material.uniforms.time.value = elapsedTime
  })

  requestAnimationFrame(animate)
  renderer.render(scene, camera)
}

const handleResize = () => {
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  if (camera && renderer) {
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()
    renderer.setSize(sizes.width, sizes.height)
  }

  smokeParticles.forEach(smoke => {
    if (smoke.geometry) smoke.geometry.dispose()
    if (smoke.userData.material) smoke.userData.material.dispose()
    scene.remove(smoke)
  })
  smokeParticles = []

  createSmoke()
}

const createSmokeTexture = () => {
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const context = canvas.getContext('2d')
  if (!context) return

  const gradient = context.createRadialGradient(256, 256, 0, 256, 256, 256)
  // 添加白色云的渐变色
  gradient.addColorStop(0, 'rgba(255, 255, 255, 0.9)')
  gradient.addColorStop(0.3, 'rgba(250, 250, 255, 0.8)')
  gradient.addColorStop(0.6, 'rgba(240, 245, 255, 0.6)')
  gradient.addColorStop(0.8, 'rgba(230, 240, 255, 0.3)')
  gradient.addColorStop(1, 'rgba(220, 230, 255, 0)')

  context.fillStyle = gradient
  context.fillRect(0, 0, 512, 512)

  const texture = new THREE.CanvasTexture(canvas)
  texture.needsUpdate = true
  return texture
}
/**
 * 粒子系统
 */
const createSmoke = () => {
  const smokeTexture = createSmokeTexture()
  if (!smokeTexture) return

  const smokeMaterial = new THREE.ShaderMaterial({
    uniforms: {
      time: { value: 0 },
      smokeTexture: { value: smokeTexture }
    },
    vertexShader: smokeVertexShader,
    fragmentShader: smokeFragmentShader,
    transparent: true,
    depthWrite: false,
    blending: THREE.NormalBlending
  })

  // 计算浏览器宽度的1/2 - 实际宽度稍微小一点，看起来不那么宽
  const halfWidth = (window.innerWidth / 2) * 0.8

  // 创建固定形状的云块，不使用随机数
  const createCloudCluster = (
    centerX: number,
    centerY: number,
    scale: number,
    depth: number,
    variant: number = 0
  ) => {
    // 主体部分 - 使用固定值
    const mainWidth = halfWidth * 0.9 * scale
    // 根据变体编号选择固定的高度比例，而不是随机生成
    const heightFactors = [0.9, 1.0, 1.2, 0.85, 1.1]
    const mainHeight =
      40 * scale * heightFactors[variant % heightFactors.length]

    const mainGeometry = new THREE.PlaneGeometry(mainWidth, mainHeight)
    const mainMesh = new THREE.Mesh(mainGeometry, smokeMaterial.clone())

    const xOffsets = [0, -5, 8, -3, 6]
    const yOffsets = [0, 3, -2, 4, -3]
    const xOffset = centerX + xOffsets[variant % xOffsets.length]
    const yOffset = centerY + yOffsets[variant % yOffsets.length]

    mainMesh.position.set(xOffset, yOffset, depth)

    // 使用固定的旋转角度
    const rotations = [0, 0.03, -0.02, 0.01, -0.03]
    mainMesh.rotation.z = rotations[variant % rotations.length]

    mainMesh.userData.material = mainMesh.material as THREE.ShaderMaterial
    smokeParticles.push(mainMesh)
    scene.add(mainMesh)

    // 为每个主体添加固定数量的小云块
    const numClouds = variant % 2 === 0 ? 2 : 3

    // 定义固定的子云块属性
    const subPositions = [
      { scale: 0.4, xFactor: -0.3, yFactor: 0.1, rotation: 0.05, z: 0.02 },
      { scale: 0.5, xFactor: 0.35, yFactor: -0.15, rotation: -0.08, z: 0.05 },
      { scale: 0.3, xFactor: 0.1, yFactor: 0.2, rotation: 0.03, z: 0.08 }
    ]

    for (let i = 0; i < numClouds; i++) {
      const subProps = subPositions[i]
      const subScale = subProps.scale
      const subWidth = mainWidth * subScale
      const subHeight = mainHeight * subScale

      const subGeometry = new THREE.PlaneGeometry(subWidth, subHeight)
      const subMesh = new THREE.Mesh(subGeometry, smokeMaterial.clone())

      // 使用固定位置
      const subX = xOffset + subProps.xFactor * mainWidth
      const subY = yOffset + subProps.yFactor * mainHeight
      const subZ = depth - 0.01 - subProps.z

      subMesh.position.set(subX, subY, subZ)
      subMesh.rotation.z = subProps.rotation

      subMesh.userData.material = subMesh.material as THREE.ShaderMaterial
      smokeParticles.push(subMesh)
      scene.add(subMesh)
    }
  }
  const centerYOffset = -2
  // 创建固定数量的云团，使用固定的位置和变体
  createCloudCluster(0, centerYOffset, 1.0, 0, 0)
  createCloudCluster(-halfWidth * 0.3, centerYOffset + 3, 0.6, -0.1, 1)
  createCloudCluster(halfWidth * 0.25, centerYOffset - 4, 0.7, -0.1, 2)
  createCloudCluster(0, centerYOffset - 8, 0.5, -0.2, 3)
  createCloudCluster(0, centerYOffset + 10, 0.5, -0.2, 4)
}

onMounted(() => {
  init()
  animate()
  window.addEventListener('resize', handleResize)
})
onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  if (renderer) {
    renderer.dispose()
  }
  if (scene) {
    scene.clear()
  }
  smokeParticles.forEach(smoke => {
    if (smoke.geometry) smoke.geometry.dispose()
    if (smoke.userData.material) smoke.userData.material.dispose()
  })
})
</script>

<template>
  <canvas ref="canvasRef"></canvas>
</template>

<style scoped>
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
  width: 100vw;
  height: 100vh;
}
</style>
