<script setup>
// COMPONENT (child): receives data through props
defineProps({
  label: { type: String, required: true },
  value: { type: [String, Number], required: true },
  tone: { type: String, default: 'neutral' },
})
</script>

<template>
  <!-- DYNAMIC CLASS BINDING: border color depends on the tone prop -->
  <div class="stat" :class="`stat--${tone}`">
    <span class="stat__value">{{ value }}</span>
    <span class="stat__label">{{ label }}</span>
  </div>
</template>

<style scoped>
.stat { padding: 14px 16px; border-left: 4px solid var(--ink); background: #fff; }
.stat--good { border-color: var(--green); }
.stat--bad  { border-color: var(--red); }
.stat--gold { border-color: var(--gold); }
.stat--hot  { border-color: var(--accent); }
.stat__value { display: block; font-size: 1.7rem; font-weight: 700; }
.stat__label { font-size: .85rem; color: #5a6472; }
</style>