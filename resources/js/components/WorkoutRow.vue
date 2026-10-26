<script setup>
import { computed } from 'vue'

// COMPONENT: props in, events out
const props = defineProps({
  workout: { type: Object, required: true },
  unit: { type: String, default: 'kg' },
  factor: { type: Number, default: 1 },
  isPb: { type: Boolean, default: false },
})
const emit = defineEmits(['remove', 'repeat'])

// COMPUTED inside a child: convert stored kg to the chosen unit
const weight = computed(() => Math.round(props.workout.weight * props.factor * 10) / 10)
const volume = computed(() =>
  Math.round(props.workout.sets * props.workout.reps * props.workout.weight * props.factor)
)

// EVENT METHOD that emits to the parent
function remove() {
  emit('remove', props.workout.id)
}
</script>

<template>
  <li class="row" :class="{ 'row--pb': isPb }">
    <div class="row__main">
      <strong>{{ workout.exercise }}</strong>
            <span class="row__sub">{{ workout.muscle }} · {{ workout.date }}</span>
      <!-- v-if: only show the note when one was entered -->
      <span v-if="workout.notes" class="row__note">“{{ workout.notes }}”</span>
    </div>

    <span class="row__cell">{{ workout.sets }} × {{ workout.reps }}</span>
    <span class="row__cell">{{ weight }} {{ unit }}</span>
    <span class="row__cell row__vol">{{ volume.toLocaleString() }} {{ unit }}</span>

    <!-- v-if / v-else -->
    <span v-if="isPb" class="badge badge--pb">New PB</span>
    <span v-else class="badge badge--normal">Logged</span>

    <button class="btn btn--ghost" @click="emit('repeat', workout.id)">Repeat</button>
    <button class="btn btn--ghost" @click="remove">Remove</button>
  </li>
</template>

<style scoped>
.row { display: grid; grid-template-columns: 1.6fr .7fr .8fr 1fr 80px auto auto; gap: 10px;
       align-items: center; padding: 10px 12px; border-bottom: 1px solid #dfe3e8; background: #fff; }
.row--pb { background: #fff8e1; box-shadow: inset 4px 0 0 var(--gold); }
.row__sub { display: block; font-size: .78rem; color: #5a6472; }
.row__note { display: block; font-size: .8rem; font-style: italic; color: #3d4753; margin-top: 2px; }
.row__cell { font-size: .92rem; }
.row__vol { font-weight: 700; }
.badge { font-size: .72rem; padding: 3px 8px; border-radius: 10px; text-align: center; }
.badge--pb { background: var(--gold); color: #3b2c00; font-weight: 700; }
.badge--normal { background: #e8ebef; color: #4a5461; }
.btn { cursor: pointer; border: 1px solid var(--ink); background: #fff; color: var(--ink);
       padding: 5px 10px; border-radius: 4px; font: inherit; font-size: .8rem; }
.btn:hover { background: var(--ink); color: #fff; }
@media (max-width: 760px) { .row { grid-template-columns: 1fr 1fr; } }
</style>