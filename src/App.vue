<template>
  <main class="app-shell">
    <aside class="sidebar">
      <div class="brand">
        <p class="eyebrow">HPC-QC Map</p>
        <h1>Global Quantum Computer Map</h1>
        <p class="subtitle">
          Browse HPC-QC centers by region, country, institution, and provider.
        </p>
      </div>

      <div class="stats-grid">
        <div class="stat-card">
          <span>Centers</span>
          <strong>{{ filteredCenters.length }}</strong>
        </div>
        <div class="stat-card">
          <span>Countries</span>
          <strong>{{ activeCountriesCount }}</strong>
        </div>
        <div class="stat-card">
          <span>Region</span>
          <strong>{{ activeAreaLabel }}</strong>
        </div>
      </div>

      <section class="panel">
        <label class="field-label" for="search">Search Institution / Provider / Country</label>
        <input
          id="search"
          v-model.trim="searchText"
          class="search-input"
          type="search"
          placeholder="Search IBM, Japan, NSCC..."
        />
      </section>

      <section class="panel">
        <div class="panel-title-row">
          <h2>Regions</h2>
          <button class="ghost-button" @click="resetView">Reset</button>
        </div>
        <div class="chip-row">
          <button
            v-for="area in areaOptions"
            :key="area.value"
            class="chip"
            :class="{ active: activeArea === area.value }"
            @click="setArea(area.value)"
          >
            {{ area.label }}
          </button>
        </div>
      </section>

      <section class="panel">
        <div class="panel-title-row">
          <h2>Countries</h2>
          <div class="inline-actions">
            <button class="text-button" @click="selectAllCountries">All</button>
            <button class="text-button" @click="clearCountries">Clear</button>
          </div>
        </div>
        <div class="country-list">
          <label v-for="country in countries" :key="country.name" class="checkbox-row">
            <input
              type="checkbox"
              :checked="selectedCountries.has(country.name)"
              @change="toggleCountry(country.name)"
            />
            <span>{{ country.name }}</span>
            <small>{{ country.count }}</small>
          </label>
        </div>
      </section>

      <section class="panel compact-options">
        <label class="switch-row">
          <input type="checkbox" v-model="showLabels" />
          <span>Show labels</span>
        </label>
        <label class="field-label" for="label-mode">Label mode</label>
        <select id="label-mode" v-model="labelMode" class="select-input">
          <option value="tooltip">Original labels</option>
          <option value="callout">Callout labels</option>
        </select>
        <label class="switch-row">
          <input type="checkbox" v-model="useClusters" />
          <span>Cluster markers</span>
        </label>
      </section>
    </aside>

    <section class="content">
      <MapView
        ref="mapRef"
        :centers="filteredCenters"
        :all-centers="centers"
        :show-labels="showLabels"
        :label-mode="labelMode"
        :use-clusters="useClusters"
        :active-area="activeArea"
        @select-center="selectedCenter = $event"
      />

      <CenterList
        :centers="filteredCenters"
        :selected-center="selectedCenter"
        @select-center="handleListSelect"
      />
    </section>
  </main>
</template>

<script setup>
import { computed, ref } from 'vue'
import centers from './data/hpc_qc_centers.json'
import MapView from './components/MapView.vue'
import CenterList from './components/CenterList.vue'

const mapRef = ref(null)
const searchText = ref('')
const activeArea = ref('Global')
const showLabels = ref(true)
const labelMode = ref('tooltip')
const useClusters = ref(false)
const selectedCenter = ref(null)

const allCountryNames = Array.from(new Set(centers.map((item) => item.country))).sort((a, b) =>
  a.localeCompare(b)
)
const selectedCountries = ref(new Set(allCountryNames))

const areaOptions = computed(() => {
  const areas = Array.from(new Set(centers.map((item) => item.area))).sort()
  return [{ label: 'Global', value: 'Global' }, ...areas.map((area) => ({ label: area, value: area }))]
})

const activeAreaLabel = computed(() => (activeArea.value === 'Global' ? 'Global' : activeArea.value))

const countries = computed(() => {
  const source = activeArea.value === 'Global'
    ? centers
    : centers.filter((item) => item.area === activeArea.value)

  const counts = source.reduce((acc, item) => {
    acc[item.country] = (acc[item.country] || 0) + 1
    return acc
  }, {})

  return Object.entries(counts)
    .map(([name, count]) => ({ name, count }))
    .sort((a, b) => a.name.localeCompare(b.name))
})

const filteredCenters = computed(() => {
  const keyword = searchText.value.toLowerCase()

  return centers.filter((item) => {
    const matchesArea = activeArea.value === 'Global' || item.area === activeArea.value
    const matchesCountry = selectedCountries.value.has(item.country)
    const text = `${item.countryRaw} ${item.country} ${item.institution} ${item.provider} ${item.address} ${item.area}`.toLowerCase()
    const matchesSearch = !keyword || text.includes(keyword)
    return matchesArea && matchesCountry && matchesSearch
  })
})

const activeCountriesCount = computed(() => {
  return new Set(filteredCenters.value.map((item) => item.country)).size
})

function toggleCountry(country) {
  const next = new Set(selectedCountries.value)
  if (next.has(country)) {
    next.delete(country)
  } else {
    next.add(country)
  }
  selectedCountries.value = next
}

function selectAllCountries() {
  selectedCountries.value = new Set(allCountryNames)
}

function clearCountries() {
  selectedCountries.value = new Set()
}

function setArea(area) {
  activeArea.value = area
  selectedCenter.value = null
}

function resetView() {
  activeArea.value = 'Global'
  searchText.value = ''
  selectedCenter.value = null
  selectAllCountries()
}

function handleListSelect(center) {
  selectedCenter.value = center
  mapRef.value?.focusCenter(center)
}
</script>
