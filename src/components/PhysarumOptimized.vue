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
  isPlaying: { type: Boolean, default: true },
  colorRemap: { type: Number, default: 0 },
  hueOffset: { type: Number, default: 0.0 },
  hueSpeed: { type: Number, default: 0.01 },
  saturation: { type: Number, default: 0.8 },
  brightness: { type: Number, default: 1.0 },
  contrast: { type: Number, default: 1.0 }
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

const emit = defineEmits(['ready'])

// Basic vertex shader for full-screen quad
const vertexShader = `#version 300 es
in vec2 a_position;
out vec2 v_texCoord;

void main() {
  gl_Position = vec4(a_position, 0.0, 1.0);
  v_texCoord = (a_position + 1.0) * 0.5;
}`

// Particle simulation shader - this runs per pixel and simulates particles
const particleShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform vec2 u_resolution;
uniform float u_time;
uniform float u_deltaTime;
uniform float u_sensorDistance;
uniform float u_sensorAngle;
uniform float u_rotationAngle;
uniform float u_moveDistance;
uniform vec2 u_mousePos;
uniform bool u_mouseDown;

in vec2 v_texCoord;
out vec4 fragColor;

// Hash function for random numbers
float hash(float n) { return fract(sin(n) * 1e4); }
float hash(vec2 p) { return fract(1e4 * sin(17.0 * p.x + p.y * 0.1) * (0.1 + abs(sin(p.y * 13.0 + p.x)))); }

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
  // Wrap coordinates
  texCoord = fract(texCoord);
  return texture(u_trailTexture, texCoord).r;
}

void main() {
  vec2 pixelCoord = v_texCoord * u_resolution;

  // Convert pixel coordinate to particle index
  float particleIndex = floor(pixelCoord.y) * u_resolution.x + floor(pixelCoord.x);
  float totalParticles = u_resolution.x * u_resolution.y;

  // Skip if we exceed the desired number of particles
  if (particleIndex >= 100000.0) {
    fragColor = vec4(0.0);
    return;
  }

  // Generate deterministic random seed for this particle
  vec2 seed = vec2(particleIndex * 0.1, particleIndex * 0.13);
  float randomSeed = hash(seed + floor(u_time * 0.1));

  // Initialize or update particle position and heading
  vec2 pos;
  float heading;

  if (u_time < 0.1) {
    // Initialize particles in a circle at the center
    float angle = randomSeed * 6.28318;
    float radius = 50.0 + noise(seed) * 100.0;
    pos = u_resolution * 0.5 + vec2(cos(angle), sin(angle)) * radius;
    heading = angle + 3.14159; // Point towards center initially
  } else {
    // Get current particle state from texture (simplified approach)
    float t = u_time * 0.01;
    float spiralRadius = 100.0 + sin(t + randomSeed * 6.28) * 50.0;
    float spiralAngle = t * 2.0 + randomSeed * 6.28;

    pos = u_resolution * 0.5 + vec2(cos(spiralAngle), sin(spiralAngle)) * spiralRadius;
    heading = spiralAngle + 1.57; // Perpendicular to spiral

    // Add some noise to create organic movement
    pos += vec2(noise(seed + t), noise(seed + t + 100.0)) * 20.0;
    heading += (noise(seed + t + 200.0) - 0.5) * 0.5;
  }

  // Physarum sensing behavior
  vec2 forward = vec2(cos(heading), sin(heading));
  vec2 left = vec2(cos(heading + u_sensorAngle), sin(heading + u_sensorAngle));
  vec2 right = vec2(cos(heading - u_sensorAngle), sin(heading - u_sensorAngle));

  float forwardSense = sampleTrail(pos + forward * u_sensorDistance);
  float leftSense = sampleTrail(pos + left * u_sensorDistance);
  float rightSense = sampleTrail(pos + right * u_sensorDistance);

  // Decision making
  float newHeading = heading;
  if (forwardSense > leftSense && forwardSense > rightSense) {
    // Continue forward - no change needed
  } else if (forwardSense < leftSense && forwardSense < rightSense) {
    // Forward is worst, make random turn
    newHeading += (hash(pos + u_time) - 0.5) * u_rotationAngle * 2.0;
  } else if (leftSense > rightSense) {
    // Turn left
    newHeading += u_rotationAngle;
  } else {
    // Turn right
    newHeading -= u_rotationAngle;
  }

  // Mouse interaction
  if (u_mouseDown) {
    vec2 toMouse = u_mousePos - pos;
    float mouseDist = length(toMouse);
    if (mouseDist > 0.0 && mouseDist < 150.0) {
      vec2 mouseDir = normalize(toMouse);
      float mouseAngle = atan(mouseDir.y, mouseDir.x);
      newHeading = mix(newHeading, mouseAngle, 0.05);
    }
  }

  // Move particle
  vec2 newPos = pos + vec2(cos(newHeading), sin(newHeading)) * u_moveDistance;

  // Boundary conditions - wrap around
  newPos = mod(newPos + u_resolution, u_resolution);

  // Calculate particle density at this pixel
  vec2 particlePixel = newPos / u_resolution;
  float distToPixel = length(particlePixel - v_texCoord);

  // Output particle density (used for trail deposition)
  float density = 0.0;
  if (distToPixel < 0.003) { // Adjust this for particle size
    density = 1.0 - (distToPixel / 0.003);
  }

  fragColor = vec4(density, density, density, 1.0);
}
`

// Trail processing shader (diffusion and decay)
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

  // Diffusion with 3x3 kernel
  float sum = 0.0;
  float weightSum = 0.0;

  for (int x = -1; x <= 1; x++) {
    for (int y = -1; y <= 1; y++) {
      vec2 offset = vec2(float(x), float(y)) * texelSize;
      vec2 samplePos = v_texCoord + offset;

      // Handle wrapping
      samplePos = fract(samplePos);

      float weight = (x == 0 && y == 0) ? 0.4 : 0.075; // Center weight higher
      sum += texture(u_trailTexture, samplePos).r * weight;
      weightSum += weight;
    }
  }

  float diffused = sum / weightSum;

  // Add new particle deposits
  float newDeposit = texture(u_particleTexture, v_texCoord).r * u_depositAmount;

  // Apply decay and add deposit
  float newTrail = (diffused * u_decayFactor) + newDeposit;

  fragColor = vec4(newTrail, newTrail, newTrail, 1.0);
}
`

// Display shader with beautiful colors
const displayShader = `#version 300 es
precision highp float;

uniform sampler2D u_trailTexture;
uniform sampler2D u_particleTexture;
uniform vec2 u_resolution;
uniform float u_time;
uniform int u_colorRemap;
uniform float u_hueOffset;
uniform float u_hueSpeed;
uniform float u_saturation;
uniform float u_brightness;
uniform float u_contrast;

in vec2 v_texCoord;
out vec4 fragColor;

vec3 hsv2rgb(vec3 c) {
  vec4 K = vec4(1.0, 2.0 / 3.0, 1.0 / 3.0, 3.0);
  vec3 p = abs(fract(c.xxx + K.xyz) * 6.0 - K.www);
  return c.z * mix(K.xxx, clamp(p - K.xxx, 0.0, 1.0), c.y);
}

// Color remapping function - maps 0-1 hue input to different color gradients
float remapHue(float hue, int remapIndex) {
  float t = fract(hue);

  if (remapIndex == 0) {
    return t; // Rainbow (default)
  } else if (remapIndex == 1) {
    return t * 0.167; // Fire: Red → Orange → Yellow
  } else if (remapIndex == 2) {
    return 0.583 - t * 0.167; // Ocean: Deep Blue → Cyan → Blue-Green
  } else if (remapIndex == 3) {
    return 0.333 - t * 0.125; // Forest: Dark Green → Lime → Yellow-Green
  } else if (remapIndex == 4) {
    return 0.75 + t * 0.167; // Purple Dream: Deep Purple → Magenta → Pink
  } else if (remapIndex == 5) {
    // Sunset: Orange → Pink → Purple
    if (t < 0.5) {
      return 0.083 - t * 0.166;
    } else {
      return 0.917 - (t - 0.5) * 0.222;
    }
  } else if (remapIndex == 6) {
    return 0.667 - t * 0.111; // Ice: Blue → Cyan → White-Blue
  }

  return t; // Fallback
}

void main() {
  float trail = texture(u_trailTexture, v_texCoord).r;
  float particles = texture(u_particleTexture, v_texCoord).r;

  float intensity = max(trail, particles * 2.0);

  if (intensity < 0.01) {
    // Dark background with subtle gradient
    vec2 center = v_texCoord - 0.5;
    float dist = length(center);
    vec3 bg = vec3(0.0, 0.0, 0.1 * (1.0 - dist * 0.5));
    fragColor = vec4(bg, 1.0);
    return;
  }

  // Apply contrast
  intensity = pow(intensity, 1.0 / max(u_contrast, 0.1));

  // Color mapping with dynamic hue
  float baseHue = fract(
    u_hueOffset +
    intensity * 1.5 +
    v_texCoord.x * 0.05 +
    v_texCoord.y * 0.03 +
    u_time * u_hueSpeed
  );

  // Apply color remapping
  float remappedHue = remapHue(baseHue, u_colorRemap);

  float saturation = u_saturation * (0.7 + intensity * 0.3);
  float brightness = u_brightness * min(intensity * 2.0, 1.0);

  // Add some bloom for high intensity areas
  if (intensity > 0.8) {
    brightness += (intensity - 0.8) * 0.5;
    saturation = min(saturation + (intensity - 0.8), 1.0);
  }

  vec3 color = hsv2rgb(vec3(remappedHue, saturation, brightness));

  // Add particles as bright spots
  if (particles > 0.1) {
    color = mix(color, vec3(u_brightness), particles * 0.3);
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
    console.error('Shader source:', source)
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

function createTexture(gl, width, height, internalFormat = gl.RGBA, format = gl.RGBA, type = gl.UNSIGNED_BYTE) {
  const texture = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, texture)

  gl.texImage2D(gl.TEXTURE_2D, 0, internalFormat, width, height, 0, format, type, null)

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
  gl = canvas.value.getContext('webgl2', {
    antialias: false,
    depth: false,
    stencil: false,
    preserveDrawingBuffer: false,
    powerPreference: 'high-performance'
  })

  if (!gl) {
    console.error('WebGL2 not supported')
    return false
  }

  // Check for required extensions
  const floatExt = gl.getExtension('EXT_color_buffer_float')
  if (!floatExt) {
    console.warn('EXT_color_buffer_float not available, using UNSIGNED_BYTE')
  }

  // Create shaders
  const vs = createShader(gl, gl.VERTEX_SHADER, vertexShader)
  const particleFS = createShader(gl, gl.FRAGMENT_SHADER, particleShader)
  const trailFS = createShader(gl, gl.FRAGMENT_SHADER, trailShader)
  const displayFS = createShader(gl, gl.FRAGMENT_SHADER, displayShader)

  if (!vs || !particleFS || !trailFS || !displayFS) {
    return false
  }

  // Create programs
  programs.particle = createProgram(gl, vs, particleFS)
  programs.trail = createProgram(gl, vs, trailFS)
  programs.display = createProgram(gl, vs, displayFS)

  if (!programs.particle || !programs.trail || !programs.display) {
    return false
  }

  // Create textures
  const useFloatTextures = floatExt !== null
  const internalFormat = useFloatTextures ? gl.R32F : gl.RGBA
  const format = useFloatTextures ? gl.RED : gl.RGBA
  const type = useFloatTextures ? gl.FLOAT : gl.UNSIGNED_BYTE

  textures.trail1 = createTexture(gl, width, height, internalFormat, format, type)
  textures.trail2 = createTexture(gl, width, height, internalFormat, format, type)
  textures.particles = createTexture(gl, width, height, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE)

  // Create framebuffers
  frameBuffers.trail1 = createFramebuffer(gl, textures.trail1)
  frameBuffers.trail2 = createFramebuffer(gl, textures.trail2)
  frameBuffers.particles = createFramebuffer(gl, textures.particles)

  // Create quad buffer
  const quadVertices = new Float32Array([
    -1, -1,  1, -1,  -1, 1,   1, 1
  ])

  buffers.quad = gl.createBuffer()
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.bufferData(gl.ARRAY_BUFFER, quadVertices, gl.STATIC_DRAW)

  // Clear textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail1)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  return true
}

function resizeCanvas() {
  const rect = canvas.value.getBoundingClientRect()
  const pixelRatio = Math.min(window.devicePixelRatio || 1, 2) // Cap at 2x for performance

  width = Math.floor(rect.width * pixelRatio)
  height = Math.floor(rect.height * pixelRatio)

  canvas.value.width = width
  canvas.value.height = height
  canvas.value.style.width = rect.width + 'px'
  canvas.value.style.height = rect.height + 'px'

  if (gl) {
    gl.viewport(0, 0, width, height)

    // Recreate textures and framebuffers with new size
    if (textures.trail1) {
      gl.deleteTexture(textures.trail1)
      gl.deleteTexture(textures.trail2)
      gl.deleteTexture(textures.particles)
      gl.deleteFramebuffer(frameBuffers.trail1)
      gl.deleteFramebuffer(frameBuffers.trail2)
      gl.deleteFramebuffer(frameBuffers.particles)

      const floatExt = gl.getExtension('EXT_color_buffer_float')
      const useFloatTextures = floatExt !== null
      const internalFormat = useFloatTextures ? gl.R32F : gl.RGBA
      const format = useFloatTextures ? gl.RED : gl.RGBA
      const type = useFloatTextures ? gl.FLOAT : gl.UNSIGNED_BYTE

      textures.trail1 = createTexture(gl, width, height, internalFormat, format, type)
      textures.trail2 = createTexture(gl, width, height, internalFormat, format, type)
      textures.particles = createTexture(gl, width, height, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE)

      frameBuffers.trail1 = createFramebuffer(gl, textures.trail1)
      frameBuffers.trail2 = createFramebuffer(gl, textures.trail2)
      frameBuffers.particles = createFramebuffer(gl, textures.particles)
    }
  }
}

let lastTime = 0
function render(currentTime) {
  if (!gl || !props.isPlaying) {
    if (props.isPlaying) {
      animationId = requestAnimationFrame(render)
    }
    return
  }

  const deltaTime = (currentTime - lastTime) * 0.001
  lastTime = currentTime
  time = currentTime * 0.001

  // 1. Simulate particles
  gl.useProgram(programs.particle)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particles)
  gl.viewport(0, 0, width, height)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  // Set particle simulation uniforms
  gl.uniform2f(gl.getUniformLocation(programs.particle, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_time'), time)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_deltaTime'), deltaTime)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_sensorDistance'), props.sensorDistance)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_sensorAngle'), props.sensorAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_rotationAngle'), props.rotationAngle)
  gl.uniform1f(gl.getUniformLocation(programs.particle, 'u_moveDistance'), props.moveDistance)
  gl.uniform2f(gl.getUniformLocation(programs.particle, 'u_mousePos'), mousePos.x, mousePos.y)
  gl.uniform1i(gl.getUniformLocation(programs.particle, 'u_mouseDown'), isMouseDown ? 1 : 0)

  // Bind trail texture for sensing
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.particle, 'u_trailTexture'), 0)

  // Draw full-screen quad
  gl.bindBuffer(gl.ARRAY_BUFFER, buffers.quad)
  gl.enableVertexAttribArray(0)
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // 2. Update trails (diffusion + decay + deposit)
  gl.useProgram(programs.trail)
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)

  gl.uniform2f(gl.getUniformLocation(programs.trail, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_decayFactor'), props.decayFactor)
  gl.uniform1f(gl.getUniformLocation(programs.trail, 'u_depositAmount'), props.depositAmount)

  // Bind textures
  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_trailTexture'), 0)

  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.particles)
  gl.uniform1i(gl.getUniformLocation(programs.trail, 'u_particleTexture'), 1)

  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  // Swap trail buffers
  const tempTex = textures.trail1
  const tempFB = frameBuffers.trail1
  textures.trail1 = textures.trail2
  frameBuffers.trail1 = frameBuffers.trail2
  textures.trail2 = tempTex
  frameBuffers.trail2 = tempFB

  // 3. Final display render
  gl.useProgram(programs.display)
  gl.bindFramebuffer(gl.FRAMEBUFFER, null)
  gl.viewport(0, 0, width, height)

  gl.uniform2f(gl.getUniformLocation(programs.display, 'u_resolution'), width, height)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_time'), time)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_colorRemap'), props.colorRemap)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_hueOffset'), props.hueOffset)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_hueSpeed'), props.hueSpeed)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_saturation'), props.saturation)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_brightness'), props.brightness)
  gl.uniform1f(gl.getUniformLocation(programs.display, 'u_contrast'), props.contrast)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, textures.trail1)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_trailTexture'), 0)

  gl.activeTexture(gl.TEXTURE1)
  gl.bindTexture(gl.TEXTURE_2D, textures.particles)
  gl.uniform1i(gl.getUniformLocation(programs.display, 'u_particleTexture'), 1)

  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)

  animationId = requestAnimationFrame(render)
}

function resetSimulation() {
  if (!gl) return

  time = 0
  lastTime = 0

  // Clear all textures
  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail1)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.trail2)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)

  gl.bindFramebuffer(gl.FRAMEBUFFER, frameBuffers.particles)
  gl.clearColor(0, 0, 0, 1)
  gl.clear(gl.COLOR_BUFFER_BIT)
}

function onMouseDown(event) {
  isMouseDown = true
  updateMousePos(event)
}

function onMouseMove(event) {
  updateMousePos(event)
}

function onMouseUp() {
  isMouseDown = false
}

function onWheel(event) {
  event.preventDefault()
}

function updateMousePos(event) {
  const rect = canvas.value.getBoundingClientRect()
  const pixelRatio = Math.min(window.devicePixelRatio || 1, 2)

  mousePos.x = (event.clientX - rect.left) * pixelRatio
  mousePos.y = height - (event.clientY - rect.top) * pixelRatio // Flip Y coordinate
}

// Watch for play state changes
watch(() => props.isPlaying, (isPlaying) => {
  if (isPlaying && !animationId) {
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
  background: #000;
}

.physarum-canvas {
  width: 100%;
  height: 100%;
  display: block;
  background: #000;
  cursor: crosshair;
  image-rendering: pixelated;
}
</style>
