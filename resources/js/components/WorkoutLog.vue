<script setup>
import { ref, reactive, computed, watch } from 'vue'
import StatCard from './StatCard.vue'
import WorkoutRow from './WorkoutRow.vue'

/* ---------- CONSTANTS + ARRAYS ---------- */
const muscles = ['Chest', 'Back', 'Legs', 'Shoulders', 'Arms', 'Core']
const LB_PER_KG = 2.20462

// Safe loader for saved data (falls back if storage is empty or broken)
function load(key, fallback) {
  try {
    const raw = localStorage.getItem(key)
    return raw ? JSON.parse(raw) : fallback
  } catch {
    return fallback
  }
}

/* ---------- ref() ---------- */
const workouts = ref(load('workouts', []))
const unit = ref(load('unit', 'kg'))
const goalKg = ref(load('goalKg', 10000))
const showForm = ref(true)
const filterMuscle = ref('All')
const searchText = ref('')
const toast = ref('')

/* ---------- REACTIVE DATA + OBJECTS ---------- */
const form = reactive({ exercise: '', muscle: 'Chest', sets: '', reps: '', weight: '' })
const errors = reactive({ exercise: '', sets: '', reps: '', weight: '' })

/* ---------- COMPUTED / REACTIVE CALCULATIONS ---------- */
const factor = computed(() => (unit.value === 'kg' ? 1 : LB_PER_KG))

// Writable computed: the goal input shows the current unit but stores kg
const goalDisplay = computed({
  get: () => Math.round(goalKg.value * factor.value),
  set: (value) => {
    const n = Number(value)
    if (n > 0) goalKg.value = n / factor.value
  },
})

const volumeOf = (w) => w.sets * w.reps * w.weight // in kg

const filteredWorkouts = computed(() => {
  const term = searchText.value.trim().toLowerCase()
  return workouts.value
    .filter((w) => filterMuscle.value === 'All' || w.muscle === filterMuscle.value)
    .filter((w) => w.exercise.toLowerCase().includes(term))
    .reverse() // newest first
})

// Personal best = heaviest weight logged for each exercise
const pbIds = computed(() => {
  const best = {}
  for (const w of workouts.value) {
    const key = w.exercise.toLowerCase()
    if (w.weight > 0 && (!best[key] || w.weight > best[key].weight)) best[key] = w
  }
  return new Set(Object.values(best).map((w) => w.id))
})

const totalVolume = computed(() =>
  Math.round(filteredWorkouts.value.reduce((sum, w) => sum + volumeOf(w), 0) * factor.value)
)
const totalSets = computed(() => filteredWorkouts.value.reduce((sum, w) => sum + w.sets, 0))
const heaviest = computed(() => {
  if (!filteredWorkouts.value.length) return 0
  return Math.round(Math.max(...filteredWorkouts.value.map((w) => w.weight)) * factor.value * 10) / 10
})
const pbCount = computed(() => filteredWorkouts.value.filter((w) => pbIds.value.has(w.id)).length)

// Weekly goal uses ALL workouts, not the filtered list
const allVolumeKg = computed(() => workouts.value.reduce((sum, w) => sum + volumeOf(w), 0))
const goalPercent = computed(() =>
  goalKg.value > 0 ? Math.min(100, Math.round((allVolumeKg.value / goalKg.value) * 100)) : 0
)
const goalReached = computed(() => workouts.value.length > 0 && allVolumeKg.value >= goalKg.value)

// Object grouping: total volume per muscle group
const breakdown = computed(() => {
  const totals = {}
  for (const m of muscles) totals[m] = 0
  for (const w of workouts.value) totals[w.muscle] += volumeOf(w)
  const max = Math.max(...Object.values(totals), 1)
  return muscles.map((m) => ({
    muscle: m,
    volume: Math.round(totals[m] * factor.value),
    percent: Math.round((totals[m] / max) * 100),
  }))
})

/* ---------- FORM VALIDATION ---------- */
function validate() {
  errors.exercise = form.exercise.trim().length < 2 ? 'Enter an exercise name (2+ characters).' : ''

  const sets = Number(form.sets)
  errors.sets = form.sets === '' || !Number.isInteger(sets) || sets < 1 || sets > 10
    ? 'Sets must be a whole number from 1 to 10.' : ''

  const reps = Number(form.reps)
  errors.reps = form.reps === '' || !Number.isInteger(reps) || reps < 1 || reps > 100
    ? 'Reps must be a whole number from 1 to 100.' : ''

  const weight = Number(form.weight)
  errors.weight = form.weight === '' || Number.isNaN(weight) || weight < 0
    ? 'Weight must be 0 or more.' : ''

  return !errors.exercise && !errors.sets && !errors.reps && !errors.weight
}

/* ---------- EVENT METHODS / EVENT HANDLING ---------- */
function addWorkout() {
  if (!validate()) return

  workouts.value.push({
    id: Date.now() + Math.random(),
    exercise: form.exercise.trim(),
    muscle: form.muscle,
    sets: Number(form.sets),
    reps: Number(form.reps),
    weight: Math.round((Number(form.weight) / factor.value) * 100) / 100, // store in kg
    date: new Date().toLocaleDateString(),
  })

  toast.value = `${form.exercise.trim()} saved.`
  form.exercise = ''
  form.sets = ''
  form.reps = ''
  form.weight = ''
}

function removeWorkout(id) {
  workouts.value = workouts.value.filter((w) => w.id !== id)
}

function repeatWorkout(id) {
  const original = workouts.value.find((w) => w.id === id)
  if (original) {
    workouts.value.push({ ...original, id: Date.now() + Math.random(), date: new Date().toLocaleDateString() })
  }
}

function toggleUnit() {
  unit.value = unit.value === 'kg' ? 'lb' : 'kg'
}

function clearAll() {
  if (confirm('Remove all workouts?')) workouts.value = []
}

/* ---------- WATCHERS ---------- */
// 1. Auto-save whenever data changes
watch(workouts, (list) => localStorage.setItem('workouts', JSON.stringify(list)), { deep: true })
watch(unit, (u) => localStorage.setItem('unit', JSON.stringify(u)))
watch(goalKg, (g) => localStorage.setItem('goalKg', JSON.stringify(g)))

// 2. Clear an error as soon as the user edits that field
watch(() => form.exercise, () => { errors.exercise = '' })
watch(() => form.sets, () => { errors.sets = '' })
watch(() => form.reps, () => { errors.reps = '' })
watch(() => form.weight, () => { errors.weight = '' })

// 3. Celebrate when the weekly goal is reached
watch(goalReached, (reached) => {
  if (reached) toast.value = 'Weekly goal reached!'
})

// 4. Hide the toast after 2.5 seconds
watch(toast, (message) => {
  if (message) setTimeout(() => (toast.value = ''), 2500)
})
</script>

<template>
  <main class="page">
    <header class="head">
      <div>
        <h1>Workout log</h1>
        <!-- INTERPOLATION -->
        <p>{{ workouts.length }} exercise{{ workouts.length === 1 ? '' : 's' }} logged</p>
      </div>
      <button class="btn btn--ghost" @click="toggleUnit">Units: {{ unit }}</button>
    </header>

    <!-- v-show: toggles CSS display, element stays in the DOM -->
    <p v-show="toast" class="toast">{{ toast }}</p>

    <!-- ===== 1. DATA ENTRY FORM ===== -->
    <section class="panel">
      <div class="panel__top">
        <h2>Log an exercise</h2>
        <button class="btn btn--ghost" @click="showForm = !showForm">
          {{ showForm ? 'Hide form' : 'Show form' }}
        </button>
      </div>

      <!-- v-if -->
      <form v-if="showForm" class="form" novalidate @submit.prevent="addWorkout">
        <label class="wide">
          Exercise
          <!-- v-model: two-way binding -->
          <input
            v-model="form.exercise"
            type="text"
            placeholder="e.g. Bench press"
            :class="{ invalid: errors.exercise }"
            @keyup.enter="addWorkout"
          />
          <small v-if="errors.exercise" class="error">{{ errors.exercise }}</small>
        </label>

        <label>
          Muscle group
          <select v-model="form.muscle">
            <option v-for="m in muscles" :key="m" :value="m">{{ m }}</option>
          </select>
        </label>

        <label>
          Sets
          <input v-model="form.sets" type="number" placeholder="4" :class="{ invalid: errors.sets }" />
          <small v-if="errors.sets" class="error">{{ errors.sets }}</small>
        </label>

        <label>
          Reps
          <input v-model="form.reps" type="number" placeholder="8" :class="{ invalid: errors.reps }" />
          <small v-if="errors.reps" class="error">{{ errors.reps }}</small>
        </label>

        <label>
          Weight ({{ unit }})
          <input v-model="form.weight" type="number" placeholder="60" :class="{ invalid: errors.weight }" />
          <small v-if="errors.weight" class="error">{{ errors.weight }}</small>
        </label>

        <button class="btn btn--solid" type="submit">Add exercise</button>
      </form>
      <p v-else class="hint">The form is hidden. Select "Show form" to log another exercise.</p>
    </section>

    <!-- ===== WEEKLY GOAL (dynamic style) ===== -->
    <section class="panel">
      <div class="panel__top">
        <h2>Weekly volume goal</h2>
        <label class="goal">
          Goal ({{ unit }})
          <input v-model.number="goalDisplay" type="number" min="1" />
        </label>
      </div>
      <div class="meter" :title="goalPercent + '%'">
        <div
          class="meter__fill"
          :style="{ width: goalPercent + '%', backgroundColor: goalReached ? '#2f7d5b' : '#e8590c' }"
        ></div>
      </div>
      <p class="hint">{{ goalPercent }}% of your goal</p>
    </section>

    <!-- ===== 7. COMPUTED RESULTS (child components) ===== -->
    <section class="stats">
      <StatCard label="Total volume" :value="`${totalVolume.toLocaleString()} ${unit}`" tone="hot" />
      <StatCard label="Total sets" :value="totalSets" />
      <StatCard label="Heaviest lift" :value="`${heaviest} ${unit}`" tone="good" />
      <StatCard label="Personal bests" :value="pbCount" tone="gold" />
    </section>

    <!-- Breakdown by muscle group: v-for + dynamic style -->
    <section class="panel">
      <h2>Volume by muscle group</h2>
      <div v-for="item in breakdown" :key="item.muscle" class="break">
        <span class="break__name">{{ item.muscle }}</span>
        <div class="break__track">
          <div class="break__fill" :style="{ width: item.percent + '%' }"></div>
        </div>
        <span class="break__num">{{ item.volume.toLocaleString() }} {{ unit }}</span>
      </div>
    </section>

    <!-- ===== 2 + 5. DATA DISPLAY / LIST RENDERING ===== -->
    <section class="panel">
      <div class="panel__top">
        <h2>Workouts</h2>
                <div class="filters">
          <input v-model="searchText" type="search" class="search" placeholder="Search exercise..." aria-label="Search exercises" />
          <select v-model="filterMuscle" aria-label="Filter by muscle group">
            <option>All</option>
            <option v-for="m in muscles" :key="m">{{ m }}</option>
          </select>
          <button v-if="workouts.length" class="btn btn--ghost" @click="clearAll">Clear all</button>
        </div>
      </div>

      <!-- 4. CONDITIONAL RENDERING: v-if / v-else -->
      <ul v-if="filteredWorkouts.length" class="list">
        <WorkoutRow
          v-for="w in filteredWorkouts"
          :key="w.id"
          :workout="w"
          :unit="unit"
          :factor="factor"
          :is-pb="pbIds.has(w.id)"
          @remove="removeWorkout"
          @repeat="repeatWorkout"
        />
      </ul>
      <p v-else class="hint">
        No workouts yet{{ filterMuscle !== 'All' ? ' for ' + filterMuscle : '' }}. Log your first exercise above.
      </p>
    </section>
  </main>
</template>

<style>
:root { --ink: #1c232b; --green: #2f7d5b; --red: #c2362f; --gold: #e0a800; --accent: #e8590c; --paper: #eef0f3; }
body { margin: 0; background: var(--paper); color: #1c232b;
       font-family: "Segoe UI", system-ui, -apple-system, sans-serif; }
</style>

<style scoped>
.page { max-width: 900px; margin: 0 auto; padding: 32px 20px 64px; }
.head { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 16px; }
.head h1 { margin: 0; font-size: 2rem; }
.head p { margin: 4px 0 0; color: #5a6472; }
.panel { background: #fff; border: 1px solid #dfe3e8; margin-bottom: 20px; padding: 18px; }
.panel__top { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
.panel h2 { margin: 0 0 4px; font-size: 1.05rem; }
.panel__top h2 { margin: 0; }
.form { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; align-items: start; }
.form .wide { grid-column: span 2; }
.form label { display: flex; flex-direction: column; gap: 4px; font-size: .85rem; }
input, select { padding: 8px; border: 1px solid #b9c0ca; border-radius: 4px; font: inherit; }
input.invalid { border-color: var(--red); background: #fff6f5; }
.error { color: var(--red); }
.btn { cursor: pointer; border: 1px solid var(--ink); background: #fff; color: var(--ink);
       padding: 8px 14px; border-radius: 4px; font: inherit; }
.btn--solid { background: var(--accent); border-color: var(--accent); color: #fff; align-self: end; }
.btn--ghost:hover { background: var(--ink); color: #fff; }
.goal { display: flex; align-items: center; gap: 8px; font-size: .85rem; }
.goal input { width: 110px; }
.meter { height: 12px; background: #e3e7ec; margin: 6px 0; }
.meter__fill { height: 100%; transition: width .4s; }
.stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(170px, 1fr)); gap: 12px; margin-bottom: 20px; }
.break { display: grid; grid-template-columns: 90px 1fr 110px; gap: 10px; align-items: center; margin-top: 10px; font-size: .88rem; }
.break__track { height: 10px; background: #e3e7ec; }
.break__fill { height: 100%; background: var(--ink); transition: width .4s; }
.break__num { text-align: right; color: #5a6472; }
.list { list-style: none; margin: 0; padding: 0; border: 1px solid #dfe3e8; }
.filters { display: flex; gap: 8px; }
.search { width: 170px; }
.hint { color: #5a6472; margin: 8px 0 0; }
.toast { background: #dff3e9; color: #1f5b41; padding: 8px 12px; margin: 0 0 16px; }
@media (max-width: 760px) { .form { grid-template-columns: 1fr 1fr; } }
</style>