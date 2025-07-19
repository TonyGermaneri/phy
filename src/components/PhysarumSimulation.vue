<template>
  <div class="physarum-container">
    <canvas
      ref="canvas"
      class="physarum-canvas"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
    ></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const canvas = ref(null)
const props = defineProps({
  numParticles: { type: Number, default: 100000 },
  sensorDistance: { type: Number, default: 9.0 },
  sensorAngle: { type: Number, default: 0.5 },
  rotationAngle: { type: Number, default: 0.3 },
  moveDistance: { type: Number, default: 1.0 },
  decayFactor: { type: Number, default: 0.9 },
  depositAmount: { type: Number, default: 5.0 }
})

let gl = null
let animationId = null
let programs = {}
let textures = {}
let buffers = {}
let frameBuffers = {}
let width = 0
let height = 0
let isMouseDown = false

const emit = defineEmits(['ready'])

// Shader sources
const vertexShaderSource = `
  attribute vec2 a_position;
  varying vec2 v_texCoord;

  void main() {
    gl_Position = vec4(a_position, 0.0, 1.0);
    v_texCoord = (a_position + 1.0) * 0.5;
  }
`

const particleVertexShader = `
  attribute vec2 a_position;
  attribute vec2 a_velocity;
  attribute float a_heading;

  uniform float u_time;
  uniform vec2 u_resolution;
  uniform sampler2D u_trailTexture;
  uniform float u_sensorDistance;
  uniform float u_sensorAngle;
  uniform float u_rotationAngle;
  uniform float u_moveDistance;

  varying vec2 v_position;
  varying vec2 v_velocity;
  varying float v_heading;

  float random(vec2 st) {
    return fract(sin(dot(st.xy, vec2(12.9898, 78.233))) * 43758.5453123);
  }

  float sampleTrail(vec2 pos) {
    vec2 texCoord = pos / u_resolution;
    if (texCoord.x < 0.0 || texCoord.x > 1.0 || texCoord.y < 0.0 || texCoord.y > 1.0) {
      return 0.0;
    }
    return texture2D(u_trailTexture, texCoord).r;
  }

  void main() {
    vec2 pos = a_position;
    float heading = a_heading;

    // Sensing
    vec2 forward = vec2(cos(heading), sin(heading));
    vec2 left = vec2(cos(heading + u_sensorAngle), sin(heading + u_sensorAngle));
    vec2 right = vec2(cos(heading - u_sensorAngle), sin(heading - u_sensorAngle));

    float forwardSense = sampleTrail(pos + forward * u_sensorDistance);
    float leftSense = sampleTrail(pos + left * u_sensorDistance);
    float rightSense = sampleTrail(pos + right * u_sensorDistance);

    // Rotation based on sensing
    if (forwardSense > leftSense && forwardSense > rightSense) {
      // Continue forward
    } else if (forwardSense < leftSense && forwardSense < rightSense) {
      // Random turn when forward is worst
      heading += (random(pos + u_time) - 0.5) * u_rotationAngle * 2.0;
    } else if (leftSense > rightSense) {
      heading += u_rotationAngle;
    } else if (rightSense > leftSense) {
      heading -= u_rotationAngle;
    }

    // Movement
    pos += vec2(cos(heading), sin(heading)) * u_moveDistance;

    // Boundary conditions (wrap around)
    pos.x = mod(pos.x + u_resolution.x, u_resolution.x);
    pos.y = mod(pos.y + u_resolution.y, u_resolution.y);

    v_position = pos;
    v_velocity = a_velocity;
    v_heading = heading;

    gl_Position = vec4((pos / u_resolution) * 2.0 - 1.0, 0.0, 1.0);
    gl_PointSize = 1.0;
  }
`

const particleFragmentShader = `
  precision mediump float;

  void main() {
    gl_FragColor = vec4(1.0);
  }
`

const trailFragmentShader = `
  precision mediump float;
  uniform sampler2D u_texture;
  uniform vec2 u_resolution;
  uniform float u_decayFactor;
  varying vec2 v_texCoord;

  void main() {
    vec2 texelSize = 1.0 / u_resolution;

    // Diffusion (3x3 kernel)
    float sum = 0.0;
    for (int x = -1; x <= 1; x++) {
      for (int y = -1; y <= 1; y++) {
        vec2 offset = vec2(float(x), float(y)) * texelSize;
        sum += texture2D(u_texture, v_texCoord + offset).r;
      }
    }

    float diffused = sum / 9.0;
    float decayed = diffused * u_decayFactor;

    gl_FragColor = vec4(decayed, decayed, decayed, 1.0);
  }
`

const displayFragmentShader = `
  precision mediump float;
  uniform sampler2D u_trailTexture;
  uniform sampler2D u_particleTexture;
  varying vec2 v_texCoord;

  void main() {
    float trail = texture2D(u_trailTexture, v_texCoord).r;
    float particles = texture2D(u_particleTexture, v_texCoord).r;

    // Color mapping
    float intensity = max(trail * 0.5, particles);

    // Create colorful gradient based on intensity
    vec3 color;
    if (intensity > 0.8) {
      color = mix(vec3(1.0, 0.8, 0.2), vec3(1.0, 1.0, 1.0), (intensity - 0.8) * 5.0);
    } else if (intensity > 0.5) {
      color = mix(vec3(0.8, 0.2, 0.8), vec3(1.0, 0.8, 0.2), (intensity - 0.5) * 3.33);
    } else if (intensity > 0.2) {
      color = mix(vec3(0.2, 0.4, 0.8), vec3(0.8, 0.2, 0.8), (intensity - 0.2) * 3.33);
    } else {
      color = mix(vec3(0.0, 0.0, 0.1), vec3(0.2, 0.4, 0.8), intensity * 5.0);
    }

    gl_FragColor = vec4(color, 1.0);
  }
`

function createShader(gl, type, source) {
  const shader = gl.createShader(type)
  gl.shaderSource(shader, source)
  gl.compileShader(shader)

  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    console.error('Shader compilation error:', gl.getShaderInfoLog(shader))
    gl.deleteShader(shader)
    return null
  }

  return shader
}

function createProgram(gl, vertexShader, fragmentShader) {
  const program = gl.createProgram()
  gl.attachShader(program, vertexShader)
  gl.attachShader(program, fragmentShader)
  gl.linkProgram(program)

  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    console.error('Program linking error:', gl.getProgramInfoLog(program))
    gl.deleteProgram(program)
    return null
  }

  return program
}

function createTexture(gl, width, height, data = null) {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, texture)

  gl.texImage2D(
    gl.TEXTURE_2D,
    0,
    gl.RGBA,
    width,
    height,
    0,
    gl.RGBA,
    gl.UNSIGNED_BYTE,
    data
  )

  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.REPEAT)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.REPEAT)

  return texture
}

function createFramebuffer(gl, texture) {
  const framebuffer = gl.createFramebuffer()
  gl.bindFramebuffer(gl.FRAMEBUFFER, framebuffer)
  gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0)
  return framebuffer
}

function initParticles() {
  const positions = new Float32Array(props.numParticles * 2)
  const velocities = new Float32Array(props.numParticles * 2)
  const headings = new Float32Array(props.numParticles)

  for (let i = 0; i < props.numParticles; i++) {
    // Random positions
    positions[i * 2] = Math.random() * width
    positions[i * 2 + 1] = Math.random() * height

    // Random velocities
    velocities[i * 2] = (Math.random() - 0.5) * 2
    velocities[i * 2 + 1] = (Math.random() - 0.5) * 2

    // Random headings
    headings[i] = Math.random() * Math.PI * 2
  }

  // Create and bind vertex buffer for positions
  buffers.positions = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.positions)
  gl.bufferData(gl.ARRAY_BUFFER, positions, gl.DYNAMIC_DRAW)

  // Create and bind vertex buffer for velocities
  buffers.velocities = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.velocities)
  gl.bufferData(gl.ARRAY_BUFFER, velocities, gl.STATIC_DRAW)

  // Create and bind vertex buffer for headings
  buffers.headings = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.headings)
  gl.bufferData(gl.ARRAY_BUFFER, headings, gl.DYNAMIC_DRAW)
}

function setupWebGL() {
  gl = canvas.value.getContext('webgl') || canvas.value.getContext('experimental-webgl')

  if (!gl) {
    console.error('WebGL not supported')
    return false
  }

  // Enable point sprites for particle rendering
  gl.enable(gl.BLEND)
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)

  // Create shaders and programs
  const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexShaderSource)
  const particleVS = createShader(gl, gl.VERTEX_SHADER, particleVertexShader)
  const particleFS = createShader(gl, gl.FRAGMENT_SHADER, particleFragmentShader)
  const trailFS = createShader(gl, gl.FRAGMENT_SHADER, trailFragmentShader)
  const displayFS = createShader(gl, gl.FRAGMENT_SHADER, displayFragmentShader)

  programs.particle = createProgram(gl, particleVS, particleFS)
  programs.trail = createProgram(gl, vertexShader, trailFS)
  programs.display = createProgram(gl, vertexShader, displayFS)

  // Create textures
  textures.trail = createTexture(gl, width, height)
  textures.trailTemp = createTexture(gl, width, height)
  textures.particles = createTexture(gl, width, height)

  // Create framebuffers
  frameBuffers.trail = createFramebuffer(gl, textures.trail)
  frameBuffers.trailTemp = createFramebuffer(gl, textures.trailTemp)
  frameBuffers.particles = createFramebuffer(gl, textures.particles)

  // Create quad buffer for full-screen rendering
  const quadVertices = new Float32Array([
    -1, -1,
     1, -1,
    -1,  1,
     1,  1
  ])

  buffers.quad = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.bufferData(gl.ARRAY_BUFFER, quadVertices, gl.STATIC_DRAW)

  initParticles()

  return true
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  width = rect.width
  height = rect.height
  canvas.value.width = width
  canvas.value.height = height

  if (gl) {
    gl.viewport(0, 0, width, height)
  }
}

function render(time) {
  if (!gl) return

  // Update particles (this is simplified - in real implementation would use transform feedback)
  gl.useProgram(programs.particle)

  // Set uniforms
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_time'), time * 0.001)
  gl.uniform2f(gl.getUniformLocation(programs.particle, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_sensorDistance'), props.sensorDistance)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_sensorAngle'), props.sensorAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_rotationAngle'), props.rotationAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_moveDistance'), props.moveDistance)

  // Bind trail texture
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(programs.particle, 'u_trailTexture'), 0)

  // Render to particle texture
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particles)
  gl.viewport(0, 0, width, height)
  gl.clear(gl.COLOR_BUFFER_BIT)

  // This is simplified - real implementation would update particle positions properly
  gl.drawArrays(gl.POINTS, 0, props.numParticles)

  // Trail diffusion and decay
  gl.useProgram(programs.trail)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trailTemp)

  gl.uniform2f(gl.getUniformLocation(programs.trail, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_decayFactor'), props.decayFactor)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_texture'), 0)

  // Render quad
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // Swap trail textures
  const temp = textures.trail
  textures.trail = textures.trailTemp
  textures.trailTemp = temp

  const tempFB = frameBuffers.trail
  frameBuffers.trail = frameBuffers.trailTemp
  frameBuffers.trailTemp = tempFB

  // Final display
  gl.useProgram(programs.display)
  gl.bindFramebuffer(gl.FRAMEBUFFER, null)
  gl.viewport(0, 0, width, height)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_trailTexture'), 0)

  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.particles)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_particleTexture'), 1)

  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  animationId = requestAnimationFrame(render)
}

function onMouseDown(event) {
  isMouseDown = true
}

function onMouseMove(event) {
  if (isMouseDown) {
    // Add interaction logic here
  }
}

function onMouseUp(event) {
  isMouseDown = false
}

onMounted(() => {
  resizeCanvas()
  if (setupWebGL()) {
    emit('ready')
    animationId = requestAnimationFrame(render)
  }

  window.addEventListener('resize', resizeCanvas)
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  window.removeEventListener('resize', resizeCanvas)
})
</script>

<style scoped>
.physarum-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  z-index: 0;
}

.physarum-canvas {
  width: 100%;
  height: 100%;
  display: block;
  background: #000;
}
</style>
