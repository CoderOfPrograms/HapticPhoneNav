<script setup>
import { computed, onBeforeUnmount, reactive, ref } from 'vue'
import { useWebHaptics } from 'web-haptics/vue'

const hapticsOptions = reactive({ showSwitch: false, debug: false })
const { trigger, cancel, isSupported } = useWebHaptics(hapticsOptions)

const audioDebug = ref(false)

const permissionState = ref('unknown')
const sensorListening = ref(false)

const heading = ref(null)
const tilt = ref({ beta: null, gamma: null })
const gravity = ref({ x: null, y: null, z: null })

const baselineHeading = ref(null)
const baselineGravity = ref(null)

const navMode = ref('stopped')
const autoWrongWay = ref(false)
const offCourseSince = ref(null)

const currentSignal = ref('idle')

const routeScript = ref([
  { mode: 'forward', durationMs: 8000, label: 'Walk straight' },
  { mode: 'turn-left', durationMs: 3500, label: 'Turn left' },
  { mode: 'forward', durationMs: 7000, label: 'Continue straight' },
  { mode: 'turn-right', durationMs: 3500, label: 'Turn right' },
  { mode: 'forward', durationMs: 6500, label: 'Approach destination' },
])

const scriptStatus = ref('idle')
const scriptStepIndex = ref(-1)
const scriptMessage = ref('Loaded demo route script.')

const originQuery = ref('Times Square, New York')
const destinationQuery = ref('Central Park, New York')
const routeLoading = ref(false)
const routeError = ref('')
const routeSummary = ref('')

let listenersAttached = false
let hapticLoop = null
let scriptTimer = null

const HAPTIC_PATTERNS = {
  forward: [
    { duration: 90, intensity: 0.35 },
    { delay: 220, duration: 90, intensity: 0.35 },
  ],
  left: [
    { duration: 55, intensity: 1 },
    { delay: 80, duration: 25, intensity: 0.5 },
    { delay: 80, duration: 55, intensity: 1 },
  ],
  right: [
    { duration: 25, intensity: 0.5 },
    { delay: 80, duration: 55, intensity: 1 },
    { delay: 80, duration: 25, intensity: 0.5 },
    { delay: 80, duration: 55, intensity: 1 },
  ],
  wrong: [{ duration: 650, intensity: 1 }],
}

const HAPTIC_INTERVAL_MS = {
  forward: 950,
  left: 900,
  right: 900,
  wrong: 1300,
}

const modeLabel = {
  forward: 'Forward',
  'turn-left': 'Turn Left',
  'turn-right': 'Turn Right',
  'wrong-way': 'Wrong Way',
}

const canCalibrate = computed(
  () => heading.value !== null && gravity.value.x !== null,
)

const expectedHeading = computed(() => {
  if (baselineHeading.value === null) return null

  if (navMode.value === 'turn-left') {
    return normalizeAngle(baselineHeading.value - 90)
  }

  if (navMode.value === 'turn-right') {
    return normalizeAngle(baselineHeading.value + 90)
  }

  return baselineHeading.value
})

const headingError = computed(() => {
  if (heading.value === null || expectedHeading.value === null) return null
  return signedHeadingError(heading.value, expectedHeading.value)
})

const liveGravityLabel = computed(() => describeUpDown(gravity.value))
const calibratedGravityLabel = computed(() => describeUpDown(baselineGravity.value))

const guidanceLabel = computed(() => {
  if (navMode.value === 'stopped') return 'Stopped'
  if (autoWrongWay.value) return 'Wrong-way alert'

  if (navMode.value === 'turn-left') return 'Turn left now'
  if (navMode.value === 'turn-right') return 'Turn right now'

  if (headingError.value === null) return 'Move forward'

  const drift = headingError.value
  if (Math.abs(drift) < 15) return 'On course'

  return drift > 0 ? 'Drifting right' : 'Drifting left'
})

const currentScriptStep = computed(() => {
  if (scriptStepIndex.value < 0 || scriptStepIndex.value >= routeScript.value.length) {
    return null
  }
  return routeScript.value[scriptStepIndex.value]
})

const scriptStepSummary = computed(() => {
  if (!currentScriptStep.value) return 'No active step'
  const step = currentScriptStep.value
  return `${modeLabel[step.mode]} for ${Math.round(step.durationMs / 1000)}s`
})

const routeHasScript = computed(() => routeScript.value.length > 0)

function normalizeAngle(value) {
  return ((value % 360) + 360) % 360
}

function signedHeadingError(current, expected) {
  let delta = normalizeAngle(current - expected)
  if (delta > 180) delta -= 360
  return delta
}

function getDominantAxis(vector) {
  if (!vector || vector.x === null || vector.y === null || vector.z === null) {
    return null
  }

  const entries = [
    { axis: 'X', value: vector.x },
    { axis: 'Y', value: vector.y },
    { axis: 'Z', value: vector.z },
  ]

  entries.sort((a, b) => Math.abs(b.value) - Math.abs(a.value))
  return entries[0]
}

function describeUpDown(vector) {
  const dominant = getDominantAxis(vector)
  if (!dominant) return 'Waiting for motion data'

  const downSign = dominant.value >= 0 ? '+' : '-'
  const upSign = dominant.value >= 0 ? '-' : '+'
  return `Down: ${downSign}${dominant.axis} | Up: ${upSign}${dominant.axis}`
}

function clearHapticLoop() {
  if (hapticLoop) {
    window.clearInterval(hapticLoop)
    hapticLoop = null
  }
}

function clearScriptTimer() {
  if (scriptTimer) {
    window.clearTimeout(scriptTimer)
    scriptTimer = null
  }
}

function setAudioDebug(enabled) {
  audioDebug.value = enabled
  hapticsOptions.debug = enabled
}

function setSignal(nextSignal) {
  if (currentSignal.value === nextSignal) return

  currentSignal.value = nextSignal
  clearHapticLoop()
  cancel()

  if (nextSignal === 'idle' || !isSupported) return

  const fire = () => {
    void trigger(HAPTIC_PATTERNS[nextSignal])
  }

  fire()
  hapticLoop = window.setInterval(fire, HAPTIC_INTERVAL_MS[nextSignal])
}

function refreshGuidanceSignal() {
  if (navMode.value === 'stopped') {
    setSignal('idle')
    return
  }

  if (autoWrongWay.value || navMode.value === 'wrong-way') {
    setSignal('wrong')
    return
  }

  if (navMode.value === 'turn-left') {
    setSignal('left')
    return
  }

  if (navMode.value === 'turn-right') {
    setSignal('right')
    return
  }

  setSignal('forward')
}

function ensureBaselineForGuidance() {
  if (baselineHeading.value === null) {
    baselineHeading.value = heading.value ?? 0
  }

  if (baselineGravity.value === null && gravity.value.x !== null) {
    baselineGravity.value = { ...gravity.value }
  }
}

function evaluateRouteProgress() {
  if (scriptStatus.value === 'running') {
    refreshGuidanceSignal()
    return
  }

  if (heading.value === null || baselineHeading.value === null) return
  if (navMode.value === 'stopped' || navMode.value === 'wrong-way') return

  const now = Date.now()

  if (navMode.value === 'forward') {
    const errorAbs = Math.abs(signedHeadingError(heading.value, baselineHeading.value))

    if (errorAbs > 70) {
      if (!offCourseSince.value) offCourseSince.value = now

      if (now - offCourseSince.value > 1400) {
        autoWrongWay.value = true
      }
    } else {
      offCourseSince.value = null
      if (autoWrongWay.value && errorAbs < 25) {
        autoWrongWay.value = false
      }
    }
  }

  if (navMode.value === 'turn-left' || navMode.value === 'turn-right') {
    const target = expectedHeading.value
    if (target === null) return

    const turnError = signedHeadingError(heading.value, target)
    if (Math.abs(turnError) < 18) {
      baselineHeading.value = heading.value
      navMode.value = 'forward'
      autoWrongWay.value = false
      offCourseSince.value = null
    }
  }

  refreshGuidanceSignal()
}

function onOrientation(event) {
  if (typeof event.beta === 'number' || typeof event.gamma === 'number') {
    tilt.value = {
      beta: typeof event.beta === 'number' ? Number(event.beta.toFixed(1)) : null,
      gamma:
        typeof event.gamma === 'number' ? Number(event.gamma.toFixed(1)) : null,
    }
  }

  let headingValue = null

  if (typeof event.webkitCompassHeading === 'number') {
    headingValue = event.webkitCompassHeading
  } else if (typeof event.alpha === 'number') {
    headingValue = event.alpha
  }

  if (headingValue !== null) {
    heading.value = Number(normalizeAngle(headingValue).toFixed(1))
  }

  evaluateRouteProgress()
}

function onMotion(event) {
  const source = event.accelerationIncludingGravity || event.acceleration
  if (!source) return

  if (
    typeof source.x !== 'number' ||
    typeof source.y !== 'number' ||
    typeof source.z !== 'number'
  ) {
    return
  }

  gravity.value = {
    x: Number(source.x.toFixed(2)),
    y: Number(source.y.toFixed(2)),
    z: Number(source.z.toFixed(2)),
  }
}

function attachSensorListeners() {
  if (listenersAttached) return

  window.addEventListener('deviceorientation', onOrientation, true)
  window.addEventListener('devicemotion', onMotion, true)
  listenersAttached = true
  sensorListening.value = true
}

function detachSensorListeners() {
  if (!listenersAttached) return

  window.removeEventListener('deviceorientation', onOrientation, true)
  window.removeEventListener('devicemotion', onMotion, true)
  listenersAttached = false
  sensorListening.value = false
}

async function enableSensors() {
  try {
    const outcomes = []

    if (
      typeof DeviceOrientationEvent !== 'undefined' &&
      typeof DeviceOrientationEvent.requestPermission === 'function'
    ) {
      outcomes.push(await DeviceOrientationEvent.requestPermission())
    }

    if (
      typeof DeviceMotionEvent !== 'undefined' &&
      typeof DeviceMotionEvent.requestPermission === 'function'
    ) {
      outcomes.push(await DeviceMotionEvent.requestPermission())
    }

    if (outcomes.some((value) => value !== 'granted')) {
      permissionState.value = 'denied'
      return
    }

    permissionState.value = 'granted'
    attachSensorListeners()
  } catch {
    permissionState.value = 'denied'
  }
}

function autoStartSensorsIfPossible() {
  const orientationNeedsGesture =
    typeof DeviceOrientationEvent !== 'undefined' &&
    typeof DeviceOrientationEvent.requestPermission === 'function'
  const motionNeedsGesture =
    typeof DeviceMotionEvent !== 'undefined' &&
    typeof DeviceMotionEvent.requestPermission === 'function'

  if (orientationNeedsGesture || motionNeedsGesture) {
    permissionState.value = 'prompt'
    return
  }

  permissionState.value = 'granted'
  attachSensorListeners()
}

function calibratePocketFrame() {
  if (!canCalibrate.value) return

  baselineHeading.value = heading.value
  baselineGravity.value = { ...gravity.value }
  offCourseSince.value = null
  autoWrongWay.value = false
  navMode.value = 'forward'
  refreshGuidanceSignal()
}

function startForward() {
  if (baselineHeading.value === null && canCalibrate.value) {
    calibratePocketFrame()
    return
  }

  if (baselineHeading.value === null) return

  navMode.value = 'forward'
  offCourseSince.value = null
  autoWrongWay.value = false
  refreshGuidanceSignal()
}

function cueLeftTurn() {
  ensureBaselineForGuidance()
  navMode.value = 'turn-left'
  autoWrongWay.value = false
  refreshGuidanceSignal()
}

function cueRightTurn() {
  ensureBaselineForGuidance()
  navMode.value = 'turn-right'
  autoWrongWay.value = false
  refreshGuidanceSignal()
}

function triggerWrongWay() {
  ensureBaselineForGuidance()
  navMode.value = 'wrong-way'
  autoWrongWay.value = true
  refreshGuidanceSignal()
}

function stopGuidance() {
  navMode.value = 'stopped'
  autoWrongWay.value = false
  offCourseSince.value = null
  refreshGuidanceSignal()
}

function normalizeScriptMode(mode) {
  if (mode === 'left' || mode === 'turn-left') return 'turn-left'
  if (mode === 'right' || mode === 'turn-right') return 'turn-right'
  if (mode === 'wrong' || mode === 'wrong-way') return 'wrong-way'
  return 'forward'
}

function applyGuidanceMode(mode) {
  const normalized = normalizeScriptMode(mode)

  if (normalized === 'turn-left') {
    cueLeftTurn()
    return
  }

  if (normalized === 'turn-right') {
    cueRightTurn()
    return
  }

  if (normalized === 'wrong-way') {
    triggerWrongWay()
    return
  }

  startForward()
}

function runScriptStep(index) {
  if (scriptStatus.value !== 'running') return

  if (index >= routeScript.value.length) {
    scriptStatus.value = 'finished'
    scriptStepIndex.value = -1
    scriptMessage.value = 'Route script finished.'
    stopGuidance()
    clearScriptTimer()
    return
  }

  const step = routeScript.value[index]
  scriptStepIndex.value = index
  applyGuidanceMode(step.mode)
  scriptMessage.value = `${index + 1}/${routeScript.value.length}: ${step.label}`

  clearScriptTimer()
  scriptTimer = window.setTimeout(() => {
    runScriptStep(index + 1)
  }, Math.max(500, step.durationMs))
}

function startRouteScript() {
  if (!routeHasScript.value) {
    scriptMessage.value = 'No route script loaded.'
    return
  }

  ensureBaselineForGuidance()
  scriptStatus.value = 'running'
  runScriptStep(0)
}

function stopRouteScript() {
  clearScriptTimer()
  scriptStatus.value = 'idle'
  scriptStepIndex.value = -1
  scriptMessage.value = 'Route script stopped.'
  stopGuidance()
}

function resetDemoScript() {
  routeScript.value = [
    { mode: 'forward', durationMs: 8000, label: 'Walk straight' },
    { mode: 'turn-left', durationMs: 3500, label: 'Turn left' },
    { mode: 'forward', durationMs: 7000, label: 'Continue straight' },
    { mode: 'turn-right', durationMs: 3500, label: 'Turn right' },
    { mode: 'forward', durationMs: 6500, label: 'Approach destination' },
  ]
  routeSummary.value = 'Demo script loaded.'
  routeError.value = ''
  scriptMessage.value = 'Loaded demo route script.'
}

async function geocodePlace(query) {
  const url = `https://nominatim.openstreetmap.org/search?format=json&limit=1&q=${encodeURIComponent(query)}`
  const response = await fetch(url)

  if (!response.ok) {
    throw new Error(`Geocoding failed (${response.status})`)
  }

  const results = await response.json()
  if (!Array.isArray(results) || results.length === 0) {
    throw new Error(`No location found for "${query}"`)
  }

  const place = results[0]
  return {
    lat: Number(place.lat),
    lon: Number(place.lon),
    name: place.display_name,
  }
}

function mapManeuverToMode(step) {
  const modifier = String(step?.maneuver?.modifier ?? '').toLowerCase()
  const type = String(step?.maneuver?.type ?? '').toLowerCase()

  if (modifier.includes('left')) return 'turn-left'
  if (modifier.includes('right')) return 'turn-right'
  if (modifier.includes('uturn') || type === 'uturn') return 'wrong-way'
  return 'forward'
}

function formatStepLabel(step, fallbackMode) {
  const instruction = step?.name ? `via ${step.name}` : 'follow route'

  if (fallbackMode === 'turn-left') return `Turn left ${instruction}`
  if (fallbackMode === 'turn-right') return `Turn right ${instruction}`
  if (fallbackMode === 'wrong-way') return `U-turn ${instruction}`

  const meters = Math.round(Number(step?.distance ?? 0))
  return `Go forward ${meters > 0 ? `${meters}m` : ''}`.trim()
}

function buildRouteScriptFromOsrm(steps) {
  const script = []

  for (const step of steps) {
    const durationMs = Math.round(Number(step.duration ?? 0) * 1000)
    const clampedForwardMs = Math.min(13000, Math.max(2500, durationMs))

    if (durationMs >= 1200) {
      script.push({
        mode: 'forward',
        durationMs: clampedForwardMs,
        label: formatStepLabel(step, 'forward'),
      })
    }

    const turnMode = mapManeuverToMode(step)
    if (turnMode !== 'forward') {
      script.push({
        mode: turnMode,
        durationMs: turnMode === 'wrong-way' ? 3000 : 2600,
        label: formatStepLabel(step, turnMode),
      })
    }
  }

  return script
}

async function loadRouteFromApi() {
  if (!originQuery.value.trim() || !destinationQuery.value.trim()) {
    routeError.value = 'Enter both origin and destination.'
    return
  }

  routeLoading.value = true
  routeError.value = ''

  try {
    const [origin, destination] = await Promise.all([
      geocodePlace(originQuery.value.trim()),
      geocodePlace(destinationQuery.value.trim()),
    ])

    const routeUrl =
      `https://router.project-osrm.org/route/v1/walking/` +
      `${origin.lon},${origin.lat};${destination.lon},${destination.lat}` +
      `?overview=false&steps=true`

    const response = await fetch(routeUrl)
    if (!response.ok) {
      throw new Error(`Routing failed (${response.status})`)
    }

    const payload = await response.json()
    const route = payload?.routes?.[0]
    const steps = route?.legs?.[0]?.steps

    if (!Array.isArray(steps) || steps.length === 0) {
      throw new Error('Route API returned no steps.')
    }

    const builtScript = buildRouteScriptFromOsrm(steps)
    if (builtScript.length === 0) {
      throw new Error('Could not derive haptic script from route steps.')
    }

    routeScript.value = builtScript
    routeSummary.value =
      `${Math.round(route.distance)}m, ${Math.round(route.duration / 60)} min, ` +
      `${builtScript.length} haptic steps`
    scriptMessage.value = 'Loaded script from live routing API.'
  } catch (error) {
    routeError.value = error instanceof Error ? error.message : 'Route fetch failed.'
  } finally {
    routeLoading.value = false
  }
}

autoStartSensorsIfPossible()

onBeforeUnmount(() => {
  clearScriptTimer()
  clearHapticLoop()
  cancel()
  detachSensorListeners()
})
</script>

<template>
  <main class="app-shell">
    <section class="panel hero">
      <p class="eyebrow">TextHeartbeat</p>
      <h1>Pocket Navigation Haptics</h1>
      <p class="lead">
        Calibrate once while facing forward. The app keeps a continuous forward buzz,
        switches to left/right turn buzzes, and pushes a heavy wrong-way alert when
        your heading drifts too far.
      </p>
      <div class="pill-row">
        <span class="pill" :class="{ ok: sensorListening }">
          Sensors: {{ sensorListening ? 'Live' : 'Not listening' }}
        </span>
        <span class="pill" :class="{ ok: permissionState === 'granted' }">
          Permission: {{ permissionState }}
        </span>
        <span class="pill" :class="{ ok: isSupported }">
          Vibration API: {{ isSupported ? 'Supported' : 'Unavailable' }}
        </span>
        <span class="pill" :class="{ ok: audioDebug }">
          Audio Debug: {{ audioDebug ? 'On' : 'Off' }}
        </span>
      </div>
      <div class="button-grid one-line">
        <button class="btn" @click="setAudioDebug(!audioDebug)">
          {{ audioDebug ? 'Disable' : 'Enable' }} Audio Debug Mode
        </button>
      </div>
      <button
        v-if="permissionState !== 'granted'"
        class="btn btn-primary"
        @click="enableSensors"
      >
        Enable Motion Sensors
      </button>
    </section>

    <section class="panel stats">
      <h2>Live Orientation</h2>
      <div class="grid">
        <article>
          <h3>Heading</h3>
          <p>{{ heading !== null ? `${heading}°` : 'No data yet' }}</p>
        </article>
        <article>
          <h3>Tilt</h3>
          <p>
            beta {{ tilt.beta !== null ? `${tilt.beta}°` : '-' }}
            | gamma {{ tilt.gamma !== null ? `${tilt.gamma}°` : '-' }}
          </p>
        </article>
        <article>
          <h3>Detected Up/Down (live)</h3>
          <p>{{ liveGravityLabel }}</p>
        </article>
        <article>
          <h3>Calibrated Pocket Frame</h3>
          <p>{{ calibratedGravityLabel }}</p>
        </article>
      </div>
    </section>

    <section class="panel controls">
      <h2>Guidance Controls</h2>
      <p class="current-state">{{ guidanceLabel }}</p>
      <div class="button-grid">
        <button class="btn btn-primary" :disabled="!canCalibrate" @click="calibratePocketFrame">
          Calibrate Forward + Pocket Up/Down
        </button>
        <button class="btn" :disabled="baselineHeading === null" @click="startForward">
          Start Forward Pulse
        </button>
        <button class="btn" :disabled="baselineHeading === null" @click="cueLeftTurn">
          Cue Left Turn
        </button>
        <button class="btn" :disabled="baselineHeading === null" @click="cueRightTurn">
          Cue Right Turn
        </button>
        <button class="btn btn-danger" :disabled="baselineHeading === null" @click="triggerWrongWay">
          Trigger Wrong-Way Alert
        </button>
        <button class="btn" @click="stopGuidance">Stop All Buzzing</button>
      </div>
      <p class="hint">
        Wrong-way auto-triggers when forward heading is off by more than ~70 degrees
        for ~1.4s, and clears once you recover near the baseline.
      </p>
    </section>

    <section class="panel controls">
      <h2>Route Script Mode</h2>
      <p class="current-state">{{ scriptStatus === 'running' ? scriptStepSummary : scriptMessage }}</p>
      <div class="button-grid">
        <button class="btn btn-primary" :disabled="scriptStatus === 'running' || !routeHasScript" @click="startRouteScript">
          Start Script Playback
        </button>
        <button class="btn" :disabled="scriptStatus !== 'running'" @click="stopRouteScript">
          Stop Script Playback
        </button>
        <button class="btn" :disabled="scriptStatus === 'running'" @click="resetDemoScript">
          Load Demo Script
        </button>
      </div>
      <p class="hint">Current steps: {{ routeScript.length }}. {{ routeSummary || 'No route summary yet.' }}</p>
      <div class="grid route-grid">
        <article>
          <h3>Origin</h3>
          <input v-model="originQuery" class="route-input" type="text" placeholder="Start location" />
        </article>
        <article>
          <h3>Destination</h3>
          <input v-model="destinationQuery" class="route-input" type="text" placeholder="Destination" />
        </article>
      </div>
      <div class="button-grid one-line">
        <button class="btn btn-primary" :disabled="routeLoading || scriptStatus === 'running'" @click="loadRouteFromApi">
          {{ routeLoading ? 'Loading Route...' : 'Load Real Route (OSM + OSRM API)' }}
        </button>
      </div>
      <p v-if="routeError" class="error-text">{{ routeError }}</p>
      <p class="hint">This uses real routing APIs (Nominatim geocoding + OSRM directions), then converts maneuvers into haptic cues.</p>
    </section>
  </main>
</template>
