<template>
  <aside class="list-panel">
    <div class="list-header">
      <div>
        <p class="toolbar-label">Center List</p>
        <h2>{{ centers.length }} centers</h2>
      </div>
    </div>

    <div class="center-list">
      <button
        v-for="center in centers"
        :key="center.id"
        class="center-card"
        :class="{ selected: selectedCenter?.id === center.id }"
        @click="$emit('select-center', center)"
      >
        <div class="card-topline">
          <span>{{ center.countryRaw }}</span>
          <small>{{ center.area }}</small>
        </div>
        <h3>{{ center.institution }}</h3>
        <p><strong>Provider:</strong> {{ center.provider }}</p>
        <p class="address">{{ center.address }}</p>
      </button>

      <div v-if="centers.length === 0" class="empty-state">
        No centers match the current filters.
      </div>
    </div>
  </aside>
</template>

<script setup>
defineProps({
  centers: {
    type: Array,
    required: true
  },
  selectedCenter: {
    type: Object,
    default: null
  }
})

defineEmits(['select-center'])
</script>
