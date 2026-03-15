<script setup>
import { ref, computed } from 'vue'

const WORK_TIME = 120 // 2 minutes (120s)
const REST_TIME = 30  // 30 seconds (30s)
const TOTAL_WORK_GOAL = 600 // 10 minutes (600s)

const phase = ref('idle') // 'idle', 'work', 'rest', 'done'
const timeLeft = ref(WORK_TIME)
const totalWorkPassed = ref(0)
const isRunning = ref(false)

let timerInterval = null
let audioCtx = null

const initAudio = () => {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  }
  if (audioCtx.state === 'suspended') {
    audioCtx.resume()
  }
}

const playSound = (freq = 440, duration = 300, type = 'sine') => {
  if (!audioCtx) initAudio()
  
  const oscillator = audioCtx.createOscillator()
  const gainNode = audioCtx.createGain()
  
  oscillator.type = type
  oscillator.frequency.setValueAtTime(freq, audioCtx.currentTime)
  
  gainNode.gain.setValueAtTime(0.3, audioCtx.currentTime)
  gainNode.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration / 1000)
  
  oscillator.connect(gainNode)
  gainNode.connect(audioCtx.destination)
  
  oscillator.start()
  oscillator.stop(audioCtx.currentTime + duration / 1000)
}

const playWorkBeep = () => {
  playSound(880, 200, 'triangle') // high beep
}

const playRestBeep = () => {
  playSound(440, 300, 'triangle') // low beep
}

const playDoneBeep = () => {
  playSound(523.25, 200, 'square') // C5
  setTimeout(() => playSound(659.25, 200, 'square'), 200) // E5
  setTimeout(() => playSound(783.99, 400, 'square'), 400) // G5
}

const formatTime = (seconds) => {
  const m = Math.floor(seconds / 60).toString().padStart(2, '0')
  const s = (seconds % 60).toString().padStart(2, '0')
  return `${m}:${s}`
}

const toggleTimer = () => {
  initAudio()
  if (phase.value === 'idle' || phase.value === 'done') {
    startWorkout()
  } else if (isRunning.value) {
    pauseWorkout()
  } else {
    resumeWorkout()
  }
}

const startWorkout = () => {
  phase.value = 'work'
  timeLeft.value = WORK_TIME
  totalWorkPassed.value = 0
  isRunning.value = true
  playWorkBeep()
  runTick()
}

const pauseWorkout = () => {
  isRunning.value = false
  clearInterval(timerInterval)
}

const resumeWorkout = () => {
  isRunning.value = true
  runTick()
}

const runTick = () => {
  clearInterval(timerInterval)
  timerInterval = setInterval(() => {
    if (timeLeft.value > 0) {
      timeLeft.value--
      if (phase.value === 'work') {
        totalWorkPassed.value++
      }
    } else {
      handlePhaseChange()
    }
  }, 1000)
}

const handlePhaseChange = () => {
  if (phase.value === 'work') {
    // Total passed should be checked. 
    // Wait, the total work passed was already incremented for the last second.
    if (totalWorkPassed.value >= TOTAL_WORK_GOAL) {
      phase.value = 'done'
      isRunning.value = false
      clearInterval(timerInterval)
      playDoneBeep()
      return
    }
    // Switch to rest
    phase.value = 'rest'
    timeLeft.value = REST_TIME
    playRestBeep()
  } else {
    // Switch to work
    phase.value = 'work'
    timeLeft.value = WORK_TIME
    playWorkBeep()
  }
}

const progressPercent = computed(() => {
  if (phase.value === 'idle') return 0
  if (phase.value === 'done') return 100
  const maxTime = phase.value === 'work' ? WORK_TIME : REST_TIME
  return ((maxTime - timeLeft.value) / maxTime) * 100
})

const overallProgressPercent = computed(() => {
  return (totalWorkPassed.value / TOTAL_WORK_GOAL) * 100
})
</script>

<template>
  <div class="app-container" :class="phase">
    <div class="glass-card">
      <h1 class="title">Intervals</h1>
      
      <div class="status">
        <span v-if="phase === 'idle'">Ready</span>
        <span v-else-if="phase === 'work'">WORK</span>
        <span v-else-if="phase === 'rest'">REST</span>
        <span v-else>DONE</span>
      </div>

      <div class="timer-display">
        {{ formatTime(timeLeft) }}
      </div>
      
      <div class="progress-bar-container current-phase">
        <div class="progress-bar" :style="{ width: progressPercent + '%' }"></div>
      </div>
      
      <div class="stats">
        <div class="stat-text">Total Work: {{ formatTime(totalWorkPassed) }} / {{ formatTime(TOTAL_WORK_GOAL) }}</div>
        <div class="progress-bar-container overall">
          <div class="progress-bar overall-bar" :style="{ width: overallProgressPercent + '%' }"></div>
        </div>
      </div>

      <button class="action-btn" @click="toggleTimer">
        <span v-if="phase === 'idle' || phase.value === 'done'">Start Workout</span>
        <span v-else-if="isRunning">Pause</span>
        <span v-else>Resume</span>
      </button>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100vw;
  height: 100vh;
  transition: background 0.5s ease-in-out;
  background: radial-gradient(circle at 50% 50%, #1a1c29 0%, #0d0e15 100%);
}

.app-container.work {
  background: radial-gradient(circle at 50% 50%, #4a2123 0%, #130303 100%);
}

.app-container.rest {
  background: radial-gradient(circle at 50% 50%, #163645 0%, #030e15 100%);
}

.app-container.done {
  background: radial-gradient(circle at 50% 50%, #1b4522 0%, #031308 100%);
}

.glass-card {
  background: rgba(255, 255, 255, 0.03);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 40px;
  padding: 3.5rem 4rem;
  text-align: center;
  box-shadow: 0 30px 60px -12px rgba(0, 0, 0, 0.7);
  display: flex;
  flex-direction: column;
  gap: 2rem;
  width: 90%;
  max-width: 480px;
  transform: translateY(0);
  animation: float 6s ease-in-out infinite;
}

@keyframes float {
  0% { transform: translateY(0px); }
  50% { transform: translateY(-10px); }
  100% { transform: translateY(0px); }
}

.title {
  font-size: 1.2rem;
  font-weight: 500;
  letter-spacing: 6px;
  text-transform: uppercase;
  margin: 0;
  color: rgba(255, 255, 255, 0.6);
}

.status {
  font-size: 3rem;
  font-weight: 800;
  letter-spacing: 4px;
  text-transform: uppercase;
  height: 3.5rem;
  text-shadow: 0 0 20px rgba(255,255,255,0.2);
  transition: color 0.3s;
}

.app-container.work .status { color: #ff6b6b; text-shadow: 0 0 30px rgba(255, 107, 107, 0.4); }
.app-container.rest .status { color: #4facfe; text-shadow: 0 0 30px rgba(79, 172, 254, 0.4); }
.app-container.done .status { color: #43e97b; text-shadow: 0 0 30px rgba(67, 233, 123, 0.4); }

.timer-display {
  font-size: 8rem;
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  line-height: 1;
  text-shadow: 0 10px 30px rgba(0,0,0,0.4);
}

.progress-bar-container {
  width: 100%;
  border-radius: 10px;
  overflow: hidden;
  background: rgba(255,255,255,0.05);
}

.current-phase {
  height: 8px;
  box-shadow: inset 0 2px 5px rgba(0,0,0,0.3);
}

.overall {
  height: 4px;
  margin-top: 0.8rem;
  box-shadow: inset 0 1px 3px rgba(0,0,0,0.3);
}

.progress-bar {
  height: 100%;
  background: #ffffff;
  transition: width 1s linear, background-color 0.3s ease;
}

.app-container.work .current-phase .progress-bar { background: #ff6b6b; box-shadow: 0 0 15px #ff6b6b; }
.app-container.rest .current-phase .progress-bar { background: #4facfe; box-shadow: 0 0 15px #4facfe; }
.app-container.done .current-phase .progress-bar { background: #43e97b; box-shadow: 0 0 15px #43e97b; }

.overall-bar {
  background: rgba(255,255,255,0.6);
  transition: width 1s linear;
}

.stats {
  font-size: 1rem;
  color: rgba(255, 255, 255, 0.7);
  display: flex;
  flex-direction: column;
  align-items: center;
}

.stat-text {
  font-weight: 300;
  letter-spacing: 1px;
}

.action-btn {
  background: rgba(255, 255, 255, 0.9);
  color: #000;
  border: none;
  padding: 1.2rem 2.5rem;
  border-radius: 40px;
  font-size: 1.2rem;
  font-weight: 700;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  margin-top: 1rem;
  letter-spacing: 1px;
  text-transform: uppercase;
}

.action-btn:hover {
  transform: scale(1.05);
  box-shadow: 0 15px 30px rgba(255,255,255,0.2);
  background: #fff;
}

.action-btn:active {
  transform: scale(0.95);
}

@media (max-width: 480px) {
  .glass-card {
    padding: 2.5rem 1.5rem;
    gap: 1.5rem;
    border-radius: 32px;
    width: 90%;
  }

  .title {
    font-size: 1rem;
    letter-spacing: 4px;
  }

  .status {
    font-size: 2rem;
    height: 2.5rem;
  }

  .timer-display {
    font-size: 5.5rem;
  }

  .stats {
    font-size: 0.9rem;
  }

  .action-btn {
    padding: 1rem 2rem;
    font-size: 1.1rem;
  }
}

@media (max-width: 360px) {
  .timer-display {
    font-size: 4.5rem;
  }
  
  .status {
    font-size: 1.8rem;
  }
}
</style>
