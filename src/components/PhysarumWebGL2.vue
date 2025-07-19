<template>
  <div class="physarum-container">
    <canvas
      ref="canvas"
      class="physarum-canvas"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
      @wheel="onWheel"
    ></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'

const canvas = ref(null)
const props = defineProps({
  numParticles: { type: Number, default: 100000 },
  sensorDistance: { type: Number, default: 9.0 },
  sensorAngle: { type: Number, default: 0.5 },
  rotationAngle: { type: Number, default: 0.3 },
  moveDistance: { type: Number, default: 1.0 },
  decayFactor: { type: Number, default: 0.9 },
  depositAmount: { type: Number, default: 5.0 },
  isPlaying: { type: Boolean, default: true }
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
let mousePos = { x: 0, y: 0 }
let time = 0
let particleData = null

const emit = defineEmits(['ready'])

// Vertex shader for full-screen quad
const quadVertexShader = `#version 300 es
in vec2 a_position;
out vec2 v_texCoord;

void main() {
  gl_Position = vec4(a_position, 0.0, 1.0);
  v_texCoord = (a_position + 1.0) * 0.5;
}`

// Particle update compute shader
const particleComputeShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform vec2 u_resolution;
uniform float u_time;
uniform float u_sensorDistance;
uniform float u_sensorAngle;
uniform float u_rotationAngle;
uniform float u_moveDistance;
uniform vec2 u_mousePos;
uniform bool u_mouseDown;

layout(location = 0) out vec4 fragColor;

in vec2 v_texCoord;

// Random function
float random(vec2 st) {
  return fract(sin(dot(st.xy, vec2(12.9898, 78.233))) * 43758.5453123);
}

float hash(float n) {
  return fract(sin(n) * 1e4);
}

float hash(vec2 p) {
  return fract(1e4 * sin(17.0 * p.x + p.y * 0.1) * (0.1 + abs(sin(p.y * 13.0 + p.x))));
}

float noise(vec2 x) {
  vec2 i = floor(x);
  vec2 f = fract(x);

  float a = hash(i);
  float b = hash(i + vec2(1.0, 0.0));
  float c = hash(i + vec2(0.0, 1.0));
  float d = hash(i + vec2(1.0, 1.0));

  vec2 u = f * f * (3.0 - 2.0 * f);
  return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) + (d - b) * u.x * u.y;
}

float sampleTrail(vec2 pos) {
  vec2 texCoord = pos / u_resolution;
  if (texCoord.x < 0.0 || texCoord.x > 1.0 || texCoord.y < 0.0 || texCoord.y > 1.0) {
    return 0.0;
  }
  return texture(u_trailTexture, texCoord).r;
}

void main() {
  vec2 pixelCoord = v_texCoord * u_resolution;
  vec2 particleId = floor(pixelCoord);

  // Initialize particle data based on pixel position
  float seed = hash(particleId + u_time * 0.001);

  // Get current particle state (this would typically come from a texture in a real implementation)
  vec2 pos;
  float heading;

  if (u_time < 1.0) {
    // Initialize particles
    pos = vec2(
      u_resolution.x * 0.5 + cos(seed * 6.28) * 100.0 * noise(particleId * 0.01),
      u_resolution.y * 0.5 + sin(seed * 6.28) * 100.0 * noise(particleId * 0.01 + 100.0)
    );
    heading = seed * 6.28318;
  } else {
    // This is a simplified version - in a real implementation, we'd read from previous frame
    pos = vec2(
      u_resolution.x * 0.5 + cos(u_time * 0.01 + seed * 6.28) * 200.0,
      u_resolution.y * 0.5 + sin(u_time * 0.01 + seed * 6.28) * 200.0
    );
    heading = u_time * 0.02 + seed * 6.28318;
  }

  // Sensing
  vec2 forward = vec2(cos(heading), sin(heading));
  vec2 left = vec2(cos(heading + u_sensorAngle), sin(heading + u_sensorAngle));
  vec2 right = vec2(cos(heading - u_sensorAngle), sin(heading - u_sensorAngle));

  float forwardSense = sampleTrail(pos + forward * u_sensorDistance);
  float leftSense = sampleTrail(pos + left * u_sensorDistance);
  float rightSense = sampleTrail(pos + right * u_sensorDistance);

  // Rotation logic
  float newHeading = heading;
  if (forwardSense > leftSense && forwardSense > rightSense) {
    // Continue forward
  } else if (forwardSense < leftSense && forwardSense < rightSense) {
    // Random turn when forward is worst
    newHeading += (random(pos + u_time) - 0.5) * u_rotationAngle * 2.0;
  } else if (leftSense > rightSense) {
    newHeading += u_rotationAngle;
  } else if (rightSense > leftSense) {
    newHeading -= u_rotationAngle;
  }

  // Movement
  vec2 newPos = pos + vec2(cos(newHeading), sin(newHeading)) * u_moveDistance;

  // Mouse attraction
  if (u_mouseDown) {
    vec2 toMouse = u_mousePos - newPos;
    float mouseDist = length(toMouse);
    if (mouseDist > 0.0 && mouseDist < 200.0) {
      vec2 mouseDir = normalize(toMouse);
      newHeading = mix(newHeading, atan(mouseDir.y, mouseDir.x), 0.1);
    }
  }

  // Boundary wrapping
  newPos.x = mod(newPos.x + u_resolution.x, u_resolution.x);
  newPos.y = mod(newPos.y + u_resolution.y, u_resolution.y);

  // Check if this pixel should render a particle
  vec2 particlePixel = floor(newPos);
  float shouldRender = 0.0;

  if (distance(particlePixel, pixelCoord) < 1.0) {
    shouldRender = 1.0;
  }

  fragColor = vec4(shouldRender, newPos.x / u_resolution.x, newPos.y / u_resolution.y, newHeading / 6.28318);
}
`

// Trail diffusion and decay shader
const trailShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform sampler2D u_particleTexture;
uniform vec2 u_resolution;
uniform float u_decayFactor;
uniform float u_depositAmount;

in vec2 v_texCoord;
out vec4 fragColor;

void main() {
  vec2 texelSize = 1.0 / u_resolution;

  // Sample current trail value
  float currentTrail = texture(u_trailTexture, v_texCoord).r;

  // Diffusion with 3x3 kernel
  float sum = 0.0;
  float weight = 0.0;

  for (int x = -1; x <= 1; x++) {
    for (int y = -1; y <= 1; y++) {
      vec2 offset = vec2(float(x), float(y)) * texelSize;
      vec2 samplePos = v_texCoord + offset;

      if (samplePos.x >= 0.0 && samplePos.x <= 1.0 && samplePos.y >= 0.0 && samplePos.y <= 1.0) {
        float w = (x == 0 && y == 0) ? 0.2 : 0.1;
        sum += texture(u_trailTexture, samplePos).r * w;
        weight += w;
      }
    }
  }

  float diffused = sum / weight;

  // Add particle deposits
  float particleDeposit = texture(u_particleTexture, v_texCoord).r * u_depositAmount;

  // Apply decay and add deposit
  float newTrail = (diffused * u_decayFactor) + particleDeposit;

  fragColor = vec4(newTrail, newTrail, newTrail, 1.0);
}
`

// Display shader with colorful rendering
const displayShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform vec2 u_resolution;
uniform float u_time;

in vec2 v_texCoord;
out vec4 fragColor;

vec3 hsv2rgb(vec3 c) {
  vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
  vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
  return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

void main() {
  float trail = texture(u_trailTexture, v_texCoord).r;

  if (trail < 0.001) {
    fragColor = vec4(0.0, 0.0, 0.05, 1.0); // Dark blue background
    return;
  }

  // Create dynamic color based on trail intensity and position
  float hue = fract(trail * 2.0 + v_texCoord.x * 0.1 + u_time * 0.05);
  float sat = 0.8 + trail * 0.2;
  float val = min(trail * 3.0, 1.0);

  vec3 color = hsv2rgb(vec3(hue, sat, val));

  // Add some bloom effect
  if (trail > 0.5) {
    color += vec3(0.2) * (trail - 0.5) * 2.0;
  }

  fragColor = vec4(color, 1.0);
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

function createTexture(gl, width, height, internalFormat = gl.RGBA, format = gl.RGBA, type = gl.UNSIGNED_BYTE, data = null) {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, texture)

  gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, width, height, 0, format, type, data)

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

  if (gl.checkFramebufferStatus(gl.FRAMEBUFFER) !== gl.FRAMEBUFFER_COMPLETE) {
    console.error('Framebuffer not complete')
  }

  return framebuffer
}

function setupWebGL() {
  gl = canvas.value.getContext('webgl2')

  if (!gl) {
    console.error('WebGL2 not supported')
    return false
  }

  // Enable necessary extensions
  const ext = gl.getExtension('EXT_color_buffer_float')
  if (!ext) {
    console.error('EXT_color_buffer_float not supported')
    return false
  }

  gl.enable(gl.BLEND)
  gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)

  // Create shaders
  const vertexShader = createShader(gl, gl.VERTEX_SHADER, quadVertexShader)
  const particleFS = createShader(gl, gl.FRAGMENT_SHADER, particleComputeShader)
  const trailFS = createShader(gl, gl.FRAGMENT_SHADER, trailShader)
  const displayFS = createShader(gl, gl.FRAGMENT_SHADER, displayShader)

  if (!vertexShader || !particleFS || !trailFS || !displayFS) {
    return false
  }

  // Create programs
  programs.particle = createProgram(gl, vertexShader, particleFS)
  programs.trail = createProgram(gl, vertexShader, trailFS)
  programs.display = createProgram(gl, vertexShader, displayFS)

  if (!programs.particle || !programs.trail || !programs.display) {
    return false
  }

  // Create textures
  textures.trail = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
  textures.trailTemp = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
  textures.particles = createTexture(gl, width, height, gl.RGBA32F, gl.RGBA, gl.FLOAT)

  // Create framebuffers
  frameBuffers.trail = createFramebuffer(gl, textures.trail)
  frameBuffers.trailTemp = createFramebuffer(gl, textures.trailTemp)
  frameBuffers.particles = createFramebuffer(gl, textures.particles)

  // Create quad buffer
  const quadVertices = new Float32Array([
    -1, -1,
     1, -1,
    -1,  1,
     1,  1
  ])

  buffers.quad = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.bufferData(gl.ARRAY_BUFFER, quadVertices, gl.STATIC_DRAW)

  // Clear initial textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trailTemp)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  return true
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  const devicePixelRatio = window.devicePixelRatio || 1

  width = Math.floor(rect.width * devicePixelRatio)
  height = Math.floor(rect.height * devicePixelRatio)

  canvas.value.width = width
  canvas.value.height = height
  canvas.value.style.width = rect.width + 'px'
  canvas.value.style.height = rect.height + 'px'

  if (gl) {
    gl.viewport(0, 0, width, height)

    // Recreate textures with new size
    if (textures.trail) {
      gl.deleteTexture(textures.trail)
      gl.deleteTexture(textures.trailTemp)
      gl.deleteTexture(textures.particles)
    }

    textures.trail = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
    textures.trailTemp = createTexture(gl, width, height, gl.R32F, gl.RED, gl.FLOAT)
    textures.particles = createTexture(gl, width, height, gl.RGBA32F, gl.RGBA, gl.FLOAT)

    // Recreate framebuffers
    if (frameBuffers.trail) {
      gl.deleteFramebuffer(frameBuffers.trail)
      gl.deleteFramebuffer(frameBuffers.trailTemp)
      gl.deleteFramebuffer(frameBuffers.particles)
    }

    frameBuffers.trail = createFramebuffer(gl, textures.trail)
    frameBuffers.trailTemp = createFramebuffer(gl, textures.trailTemp)
    frameBuffers.particles = createFramebuffer(gl, textures.particles)
  }
}

function render(timestamp) {
  if (!gl || !props.isPlaying) {
    if (props.isPlaying) {
      animationId = requestAnimationFrame(render)
    }
    return
  }

  time = timestamp * 0.001

  // 1. Update particles
  gl.useProgram(programs.particle)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particles)
  gl.viewport(0, 0, width, height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  // Set uniforms for particle update
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_time'), time)
  gl.uniform2f(gl.getUniformLocation(programs.particle, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_sensorDistance'), props.sensorDistance)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_sensorAngle'), props.sensorAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_rotationAngle'), props.rotationAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_moveDistance'), props.moveDistance)
  gl.uniform2f(gl.getUniformLocation(programs.particle, 'u_mousePos'), mousePos.x, mousePos.y)
  gl.uniform1i(gl.getUniformLocation(programs.particle, 'u_mouseDown'), isMouseDown ? 1 : 0)

  // Bind trail texture for sensing
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(programs.particle, 'u_trailTexture'), 0)

  // Render full-screen quad
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // 2. Update trail (diffusion, decay, and deposit)
  gl.useProgram(programs.trail)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trailTemp)

  gl.uniform2f(gl.getUniformLocation(programs.trail, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_decayFactor'), props.decayFactor)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_depositAmount'), props.depositAmount)

  // Bind textures
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_trailTexture'), 0)

  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.particles)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_particleTexture'), 1)

  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // Swap trail textures
  const tempTex = textures.trail
  const tempFB = frameBuffers.trail
  textures.trail = textures.trailTemp
  frameBuffers.trail = frameBuffers.trailTemp
  textures.trailTemp = tempTex
  frameBuffers.trailTemp = tempFB

  // 3. Final display
  gl.useProgram(programs.display)
  gl.bindFramebuffer(gl.FRAMEBUFFER, null)
  gl.viewport(0, 0, width, height)

  gl.uniform2f(gl.getUniformLocation(programs.display, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_time'), time)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_trailTexture'), 0)

  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  animationId = requestAnimationFrame(render)
}

function onMouseDown(event) {
  isMouseDown = true
  updateMousePos(event)
}

function onMouseMove(event) {
  updateMousePos(event)
}

function onMouseUp(event) {
  isMouseDown = false
}

function onWheel(event) {
  event.preventDefault()
}

function updateMousePos(event) {
  const rect = canvas.value.getBoundingClientRect()
  const devicePixelRatio = window.devicePixelRatio || 1

  mousePos.x = (event.clientX - rect.left) * devicePixelRatio
  mousePos.y = (rect.height - (event.clientY - rect.top)) * devicePixelRatio
}

function resetSimulation() {
  if (!gl) return

  time = 0

  // Clear trail textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trailTemp)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)
}

// Watch for parameter changes
watch(() => props, () => {
  // Parameters are automatically used in the next render frame
}, { deep: true })

watch(() => props.isPlaying, (newVal) => {
  if (newVal && !animationId) {
    animationId = requestAnimationFrame(render)
  }
})

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

defineExpose({
  resetSimulation
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
  cursor: crosshair;
}
</style>
