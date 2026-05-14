<template>
  <div class="tech-chart">
    <div class="panel-title-row">
      <h2>Technology Types</h2>
      <small>{{ total }} centers</small>
    </div>

    <div class="bar-list">
      <div v-for="item in chartItems" :key="item.name" class="bar-row">
        <div class="bar-meta">
          <span>{{ item.name }}</span>
          <strong>{{ item.count }}</strong>
        </div>
        <div class="bar-track" :aria-label="`${item.name}: ${item.count}`">
          <div class="bar-fill" :style="{ width: `${item.percent}%` }"></div>
        </div>

        <div class="bar-tooltip" role="tooltip">
          <strong>{{ item.name }}</strong>
          <span>{{ item.providers.length }} providers</span>
          <p>{{ item.providers.join(', ') }}</p>
        </div>
      </div>

      <div v-if="chartItems.length === 0" class="chart-empty">
        No technology data for the current filters.
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  items: {
    type: Array,
    required: true
  }
})

const maxCount = computed(() => Math.max(...props.items.map((item) => item.count), 0))
const total = computed(() => props.items.reduce((sum, item) => sum + item.count, 0))

const chartItems = computed(() => {
  return props.items.map((item) => ({
    ...item,
    percent: maxCount.value ? Math.max((item.count / maxCount.value) * 100, 4) : 0
  }))
})
</script>
