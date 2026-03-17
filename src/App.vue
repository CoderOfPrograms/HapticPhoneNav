<script setup>
import { computed, onBeforeUnmount, reactive, ref } from 'vue'
import { useWebHaptics } from 'web-haptics/vue'

const hapticsOptions = reactive({ showSwitch: false, debug: false })
const { trigger, cancel, isSupported } = useWebHaptics(hapticsOptions)

const audioDebug = ref(false)
const permissionState = ref('unknown')
const locationState = ref('idle')
const sensorListening = ref(false)

const tilt = ref({ beta: null, gamma: null })
const gravity = ref({ x: null, y: null, z: null })

const gps = ref({
  lat: null,
  lon: null,
  heading: null,
  speed: null,
  accuracy: null,
})

const baselineCourse = ref(null)
const pocketAnchor = ref('top')
const pocketStatus = ref('Run countdown to detect top/bottom pocket direction.')
const pocketCountdown = ref(0)

const navMode = ref('stopped')
const autoWrongWay = ref(false)
const offCourseSince = ref(null)
const currentSignal = ref('idle')
const flashingButtonId = ref('')

const routeScript = ref([
  { mode: 'straight', durationMs: 6500, label: 'Go straight' },
  { mode: 'left', durationMs: 2800, label: 'Turn left' },
  { mode: 'straight', durationMs: 5200, label: 'Keep going straight' },
  { mode: 'right', durationMs: 2800, label: 'Turn right' },
  { mode: 'wrong', durationMs: 3200, label: 'Wrong-way warning' },
])

const scriptStatus = ref('idle')
const scriptStepIndex = ref(-1)
const scriptMessage = ref('Loaded demo script: straight, left, straight, right, wrong-way.')

let listenersAttached = false
let hapticLoop = null
let scriptTimer = null
let countdownTimer = null
let flashTimer = null
let locationWatchId = null
let lastFix = null

const HAPTIC_PATTERNS = {
  straight: [
    { duration: 80, intensity: 0.7 },
    { delay: 150, duration: 80, intensity: 0.7 },
  ],
  left: [
    { duration: 200, intensity: 1 },
    { delay: 45, duration: 90, intensity: 0.6 },
    { delay: 45, duration: 55, intensity: 0.35 },
  ],
  right: [
    { duration: 55, intensity: 0.35 },
    { delay: 45, duration: 90, intensity: 0.6 },
    { delay: 45, duration: 200, intensity: 1 },
  ],
  wrong: [
    { duration: 420, intensity: 1 },
    { delay: 100, duration: 420, intensity: 1 },
  ],
}

const HAPTIC_INTERVAL_MS = {
  straight: 920,
  left: 1100,
  right: 1100,
  wrong: 1450,
}

const canCalibratePocket = computed(() => gravity.value.x !== null)

const expectedHeading = computed(() => {
  if (baselineCourse.value === null) return null
  if (navMode.value === 'turn-left') return normalizeAngle(baselineCourse.value - 90)
  if (navMode.value === 'turn-right') return normalizeAngle(baselineCourse.value + 90)
  return baselineCourse.value
})

const courseError = computed(() => {
  if (gps.value.heading === null || expectedHeading.value === null) return null
  return signedHeadingError(gps.value.heading, expectedHeading.value)
})

const liveGravityLabel = computed(() => describeUpDown(gravity.value))

const headingLabel = computed(() => {
  if (gps.value.heading === null) return 'No GPS heading yet'
  return `${gps.value.heading.toFixed(1)} deg`
})

const guidanceLabel = computed(() => {
  if (navMode.value === 'stopped') return 'Stopped'
  if (autoWrongWay.value || navMode.value === 'wrong-way') return 'Wrong-way alert active'
  if (navMode.value === 'turn-left') return 'Turn left now'
  if (navMode.value === 'turn-right') return 'Turn right now'

  if (courseError.value === null) return 'Straight cue active'

  const absError = Math.abs(courseError.value)
  if (absError <= 55) return 'GPS heading is in the right direction'
  if (absError <= 95) return 'Still acceptable, keep moving forward'
  return 'Wrong direction detected, rotate back'
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
  return `${step.label} (${Math.round(step.durationMs / 1000)}s)`
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

function toRadians(value) {
  return (value * Math.PI) / 180
}

function toDegrees(value) {
  return (value * 180) / Math.PI
}

function distanceMeters(lat1, lon1, lat2, lon2) {
  const earthRadius = 6371000
  const dLat = toRadians(lat2 - lat1)
  const dLon = toRadians(lon2 - lon1)

  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRadians(lat1)) *
      Math.cos(toRadians(lat2)) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2)

  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return earthRadius * c
}

function bearingBetween(lat1, lon1, lat2, lon2) {
  const phi1 = toRadians(lat1)
  const phi2 = toRadians(lat2)
  const dLon = toRadians(lon2 - lon1)

  const y = Math.sin(dLon) * Math.cos(phi2)
  const x =
    Math.cos(phi1) * Math.sin(phi2) -
    Math.sin(phi1) * Math.cos(phi2) * Math.cos(dLon)

  return normalizeAngle(toDegrees(Math.atan2(y, x)))
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

function inferPocketAnchor(vector) {
  const dominant = getDominantAxis(vector)
  if (!dominant) return null

  if (dominant.axis === 'Y') {
    return dominant.value >= 0 ? 'bottom' : 'top'
  }

  return dominant.value >= 0 ? 'top' : 'bottom'
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

function clearCountdownTimer() {
  if (countdownTimer) {
    window.clearInterval(countdownTimer)
    countdownTimer = null
  }
}

function clearFlashTimer() {
  if (flashTimer) {
    window.clearTimeout(flashTimer)
    flashTimer = null
  }
}

function flashButton(buttonId) {
  flashingButtonId.value = buttonId
  clearFlashTimer()
  flashTimer = window.setTimeout(() => {
    flashingButtonId.value = ''
  }, 220)
}

function signalButtonId(signal) {
  if (signal === 'straight') {
    return pocketAnchor.value === 'bottom' ? 'straight-bottom' : 'straight-top'
  }

  if (signal === 'left') return 'left'
  if (signal === 'right') return 'right'
  return 'wrong'
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
    flashButton(signalButtonId(nextSignal))
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

  setSignal('straight')
}

function ensureBaselineCourse() {
  if (baselineCourse.value === null && gps.value.heading !== null) {
    baselineCourse.value = gps.value.heading
  }
}

function evaluateRouteProgress() {
  if (scriptStatus.value === 'running') {
    refreshGuidanceSignal()
    return
  }

  if (gps.value.heading === null || baselineCourse.value === null) {
    refreshGuidanceSignal()
    return
  }

  if (navMode.value === 'stopped' || navMode.value === 'wrong-way') {
    refreshGuidanceSignal()
    return
  }

  const now = Date.now()

  if (navMode.value === 'straight') {
    const errorAbs = Math.abs(signedHeadingError(gps.value.heading, baselineCourse.value))

    if (errorAbs > 110) {
      if (!offCourseSince.value) offCourseSince.value = now
      if (now - offCourseSince.value > 3000) {
        autoWrongWay.value = true
      }
    } else {
      offCourseSince.value = null
      if (autoWrongWay.value && errorAbs < 70) {
        autoWrongWay.value = false
      }
    }
  }

  if (navMode.value === 'turn-left' || navMode.value === 'turn-right') {
    const target = expectedHeading.value
    if (target !== null) {
      const error = signedHeadingError(gps.value.heading, target)
      if (Math.abs(error) < 32) {
        baselineCourse.value = gps.value.heading
        navMode.value = 'straight'
        autoWrongWay.value = false
        offCourseSince.value = null
      }
    }
  }

  refreshGuidanceSignal()
}

function onOrientation(event) {
  if (typeof event.beta === 'number' || typeof event.gamma === 'number') {
    tilt.value = {
      beta: typeof event.beta === 'number' ? Number(event.beta.toFixed(1)) : null,
      gamma: typeof event.gamma === 'number' ? Number(event.gamma.toFixed(1)) : null,
    }
  }
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

function onLocation(position) {
  const { latitude, longitude, heading, speed, accuracy } = position.coords
  let nextHeading = typeof heading === 'number' && !Number.isNaN(heading) ? normalizeAngle(heading) : null

  if (nextHeading === null && lastFix) {
    const moved = distanceMeters(lastFix.lat, lastFix.lon, latitude, longitude)
    if (moved >= 3) {
      nextHeading = bearingBetween(lastFix.lat, lastFix.lon, latitude, longitude)
    }
  }

  gps.value = {
    lat: Number(latitude.toFixed(6)),
    lon: Number(longitude.toFixed(6)),
    heading: nextHeading === null ? gps.value.heading : Number(nextHeading.toFixed(1)),
    speed: typeof speed === 'number' && !Number.isNaN(speed) ? Number(speed.toFixed(2)) : null,
    accuracy: typeof accuracy === 'number' ? Number(accuracy.toFixed(1)) : null,
  }

  lastFix = { lat: latitude, lon: longitude }
  locationState.value = 'watching'
  evaluateRouteProgress()
}

function onLocationError() {
  locationState.value = 'denied'
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

function attachLocationWatcher() {
  if (!('geolocation' in navigator)) {
    locationState.value = 'unsupported'
    return
  }

  if (locationWatchId !== null) return

  locationState.value = 'watching'
  locationWatchId = navigator.geolocation.watchPosition(onLocation, onLocationError, {
    enableHighAccuracy: true,
    maximumAge: 1000,
    timeout: 12000,
  })
}

function detachLocationWatcher() {
  if (locationWatchId === null) return
  navigator.geolocation.clearWatch(locationWatchId)
  locationWatchId = null
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
    attachLocationWatcher()
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
  attachLocationWatcher()
}

function setPocketAnchor(anchor) {
  pocketAnchor.value = anchor
  pocketStatus.value =
    anchor === 'top'
      ? 'Top emitter selected for straight pulses.'
      : 'Bottom emitter selected for straight pulses.'
}

function finalizePocketCalibration() {
  const inferred = inferPocketAnchor(gravity.value)
  if (!inferred) {
    pocketStatus.value = 'Could not detect orientation yet. Keep phone still and retry.'
    return
  }

  setPocketAnchor(inferred)
}

function startPocketCountdown() {
  if (!canCalibratePocket.value) {
    pocketStatus.value = 'Waiting for motion data before countdown.'
    return
  }

  clearCountdownTimer()
  pocketCountdown.value = 3
  pocketStatus.value = 'Hold still. Detecting pocket direction...'

  countdownTimer = window.setInterval(() => {
    pocketCountdown.value -= 1

    if (pocketCountdown.value <= 0) {
      clearCountdownTimer()
      finalizePocketCalibration()
    }
  }, 1000)
}

function startStraight() {
  ensureBaselineCourse()

  navMode.value = 'straight'
  offCourseSince.value = null
  autoWrongWay.value = false
  refreshGuidanceSignal()
}

function startStraightFromAnchor(anchor) {
  setPocketAnchor(anchor)
  startStraight()
}

function cueLeftTurn() {
  ensureBaselineCourse()
  navMode.value = 'turn-left'
  autoWrongWay.value = false
  refreshGuidanceSignal()
}

function cueRightTurn() {
  ensureBaselineCourse()
  navMode.value = 'turn-right'
  autoWrongWay.value = false
  refreshGuidanceSignal()
}

function triggerWrongWay() {
  ensureBaselineCourse()
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
  return 'straight'
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

  startStraight()
}

function runScriptStep(index) {
  if (scriptStatus.value !== 'running') return

  if (index >= routeScript.value.length) {
    scriptStatus.value = 'finished'
    scriptStepIndex.value = -1
    scriptMessage.value = 'Demo script finished.'
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
    scriptMessage.value = 'No script loaded.'
    return
  }

  ensureBaselineCourse()
  scriptStatus.value = 'running'
  runScriptStep(0)
}

function stopRouteScript() {
  clearScriptTimer()
  scriptStatus.value = 'idle'
  scriptStepIndex.value = -1
  scriptMessage.value = 'Demo script stopped.'
  stopGuidance()
}

function resetDemoScript() {
  routeScript.value = [
    { mode: 'straight', durationMs: 6500, label: 'Go straight' },
    { mode: 'left', durationMs: 2800, label: 'Turn left' },
    { mode: 'straight', durationMs: 5200, label: 'Keep going straight' },
    { mode: 'right', durationMs: 2800, label: 'Turn right' },
    { mode: 'wrong', durationMs: 3200, label: 'Wrong-way warning' },
  ]

  scriptMessage.value = 'Loaded demo script: straight, left, straight, right, wrong-way.'
}

autoStartSensorsIfPossible()

onBeforeUnmount(() => {
  clearScriptTimer()
  clearHapticLoop()
  clearCountdownTimer()
  clearFlashTimer()
  cancel()
  detachSensorListeners()
  detachLocationWatcher()
})
</script>

<template>
  <main class="app-shell">
    <section class="panel hero">
      <p class="eyebrow">TextHeartbeat</p>
      <h1>4-Signal Pocket Navigator</h1>
      <p class="lead">
        Four haptic signals only: straight, left, right, and wrong-way. Straight pulses
        auto-emit from the top or bottom button based on pocket direction.
      </p>
      <div class="pill-row">
        <span class="pill" :class="{ ok: sensorListening }">
          Sensors: {{ sensorListening ? 'Live' : 'Not listening' }}
        </span>
        <span class="pill" :class="{ ok: permissionState === 'granted' }">
          Motion Permission: {{ permissionState }}
        </span>
        <span class="pill" :class="{ ok: locationState === 'watching' }">
          GPS: {{ locationState }}
        </span>
        <span class="pill" :class="{ ok: isSupported }">
          Vibration API: {{ isSupported ? 'Supported' : 'Unavailable' }}
        </span>
      </div>
      <div class="pill-row">
        <span class="pill">GPS Heading: {{ headingLabel }}</span>
        <span class="pill">Accuracy: {{ gps.accuracy !== null ? `${gps.accuracy}m` : '-' }}</span>
        <span class="pill">Pocket Anchor: {{ pocketAnchor === 'top' ? 'Top' : 'Bottom' }}</span>
      </div>
      <div class="button-grid">
        <button class="btn" @click="setAudioDebug(!audioDebug)">
          {{ audioDebug ? 'Disable' : 'Enable' }} Audio Debug Mode
        </button>
        <button class="btn btn-primary" @click="enableSensors">
          Enable Motion + GPS
        </button>
      </div>
    </section>

    <section class="panel stats">
      <h2>Pocket Direction Setup</h2>
      <div class="grid">
        <article>
          <h3>Live Up/Down Axis</h3>
          <p>{{ liveGravityLabel }}</p>
        </article>
        <article>
          <h3>Tilt</h3>
          <p>
            beta {{ tilt.beta !== null ? `${tilt.beta} deg` : '-' }}
            | gamma {{ tilt.gamma !== null ? `${tilt.gamma} deg` : '-' }}
          </p>
        </article>
      </div>
      <div class="button-grid">
        <button class="btn btn-primary" :disabled="!canCalibratePocket" @click="startPocketCountdown">
          {{ pocketCountdown > 0 ? `Detecting in ${pocketCountdown}` : 'Start 3s Pocket Countdown' }}
        </button>
        <button class="btn" @click="setPocketAnchor('top')">Use Top Anchor</button>
        <button class="btn" @click="setPocketAnchor('bottom')">Use Bottom Anchor</button>
      </div>
      <p class="hint">{{ pocketStatus }}</p>
    </section>

    <section class="panel controls">
      <h2>Haptic Signal Console</h2>
      <p class="current-state">{{ guidanceLabel }}</p>

      <div class="signal-pad">
        <button
          class="btn btn-primary"
          :class="{
            'btn-live': flashingButtonId === 'straight-top',
            'btn-anchor': pocketAnchor === 'top',
          }"
          @click="startStraightFromAnchor('top')"
        >
          Top Straight Emitter
        </button>

        <div class="turn-row">
          <button
            class="btn"
            :class="{ 'btn-live': flashingButtonId === 'left' }"
            @click="cueLeftTurn"
          >
            Strong Left
          </button>
          <button
            class="btn"
            :class="{ 'btn-live': flashingButtonId === 'right' }"
            @click="cueRightTurn"
          >
            Strong Right
          </button>
        </div>

        <button
          class="btn btn-primary"
          :class="{
            'btn-live': flashingButtonId === 'straight-bottom',
            'btn-anchor': pocketAnchor === 'bottom',
          }"
          @click="startStraightFromAnchor('bottom')"
        >
          Bottom Straight Emitter
        </button>

        <button
          class="btn btn-danger"
          :class="{ 'btn-live': flashingButtonId === 'wrong' }"
          @click="triggerWrongWay"
        >
          Wrong-Way Alert
        </button>

        <button class="btn" @click="stopGuidance">Stop All Buzzing</button>
      </div>

      <p class="hint">
        Left and right patterns are intentionally asymmetric and high contrast. GPS course is
        treated with broad tolerance, so slight heading wobble does not trigger side drift cues.
      </p>
    </section>

    <section class="panel controls">
      <h2>Demo Script</h2>
      <p class="current-state">{{ scriptStatus === 'running' ? scriptStepSummary : scriptMessage }}</p>
      <div class="button-grid">
        <button class="btn btn-primary" :disabled="scriptStatus === 'running' || !routeHasScript" @click="startRouteScript">
          Start Demo Playback
        </button>
        <button class="btn" :disabled="scriptStatus !== 'running'" @click="stopRouteScript">
          Stop Demo Playback
        </button>
        <button class="btn" :disabled="scriptStatus === 'running'" @click="resetDemoScript">
          Reload Demo Sequence
        </button>
      </div>
      <p class="hint">Sequence: straight -> left -> straight -> right -> wrong-way.</p>
    </section>
  </main>
</template>
