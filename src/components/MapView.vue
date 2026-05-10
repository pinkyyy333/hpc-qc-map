<template>
  <div class="map-panel">
    <div class="map-toolbar">
      <div>
        <p class="toolbar-label">Map View</p>
        <strong>{{ centers.length }} centers</strong>
      </div>
      <div class="toolbar-actions">
        <button @click="fitToMarkers">Fit markers</button>
        <button @click="fitGlobal">Global</button>
      </div>
    </div>

    <div ref="mapContainer" class="map-container"></div>

    <div v-if="showLabels && labelMode === 'callout'" class="label-overlay">
      <svg class="label-lines" aria-hidden="true">
        <line
          v-for="line in labelLines"
          :key="line.id"
          :x1="line.x1"
          :y1="line.y1"
          :x2="line.x2"
          :y2="line.y2"
        />
      </svg>

      <div
        v-for="label in labelItems"
        :key="label.id"
        class="map-info-label"
        :class="label.className"
        :style="{ left: `${label.x}px`, top: `${label.y}px` }"
      >
        <strong>{{ label.institution }}</strong>
      </div>
    </div>

    <article v-if="selectedCenter" class="detail-card">
      <button class="close-button" @click="selectedCenter = null">?</button>
      <p class="eyebrow">{{ selectedCenter.area }}</p>
      <h2>{{ selectedCenter.institution }}</h2>
      <dl>
        <div>
          <dt>Country</dt>
          <dd>{{ selectedCenter.countryRaw }}</dd>
        </div>
        <div>
          <dt>Provider</dt>
          <dd>{{ selectedCenter.provider }}</dd>
        </div>
        <div>
          <dt>Address</dt>
          <dd>{{ selectedCenter.address }}</dd>
        </div>
      </dl>
    </article>
  </div>
</template>

<script setup>
import { nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import L from 'leaflet'
import 'leaflet.markercluster'

const props = defineProps({
  centers: {
    type: Array,
    required: true
  },
  allCenters: {
    type: Array,
    required: true
  },
  showLabels: {
    type: Boolean,
    default: false
  },
  labelMode: {
    type: String,
    default: 'callout'
  },
  useClusters: {
    type: Boolean,
    default: true
  },
  activeArea: {
    type: String,
    default: 'Global'
  }
})

const emit = defineEmits(['select-center'])

const mapContainer = ref(null)
const selectedCenter = ref(null)
const labelItems = ref([])
const labelLines = ref([])

let map = null
let markerLayer = null
let labelFrame = null

const LABEL_WIDTH = 150
const LABEL_HEIGHT = 34
const LABEL_GAP = 8
const LABEL_TOP_SAFE = 72

const areaViews = {
  Global: { center: [25, 20], zoom: 2 },
  Americas: { center: [28, -96], zoom: 4 },
  'EuroHPC JU': { center: [51, 13], zoom: 4 },
  'Asia-Pacific': { center: [23, 109], zoom: 3 }
}

const areaColors = {
  'EuroHPC JU': '#d7a441',
  Americas: '#22c55e',
  'Asia-Pacific': '#ec4899'
}

const areaLabelClasses = {
  'EuroHPC JU': 'marker-label--eurohpc',
  Americas: 'marker-label--americas',
  'Asia-Pacific': 'marker-label--asia-pacific'
}

onMounted(() => {
  map = L.map(mapContainer.value, {
    zoomControl: true,
    worldCopyJump: true
  }).setView(areaViews.Global.center, areaViews.Global.zoom)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 18,
    attribution: '穢 OpenStreetMap contributors'
  }).addTo(map)

  map.on('move zoom resize moveend zoomend viewreset', scheduleUpdateLabels)
  renderMarkers()
})

onBeforeUnmount(() => {
  if (map) {
    map.remove()
    map = null
  }
})

watch(
  () => [props.centers, props.showLabels, props.labelMode, props.useClusters],
  () => {
    renderMarkers()
  },
  { deep: true }
)

watch(
  () => props.activeArea,
  () => {
    nextTick(() => {
      if (props.activeArea === 'Global') {
        fitGlobal()
      } else {
        fitToMarkers()
      }
    })
  }
)

function createMarker(center) {
  const color = areaColors[center.area] || '#0f766e'
  const icon = L.divIcon({
    className: '',
    html: `<span class="custom-marker" style="--marker-color:${color}"></span>`,
    iconSize: [18, 18],
    iconAnchor: [9, 9]
  })

  const marker = L.marker([center.latitude, center.longitude], { icon })

  marker.bindPopup(`
    <div class="popup-card">
      <strong>${escapeHtml(center.institution)}</strong>
      <p>${escapeHtml(center.countryRaw)}</p>
      <p><b>Provider:</b> ${escapeHtml(center.provider)}</p>
    </div>
  `)

  if (props.showLabels && props.labelMode === 'tooltip') {
    marker.bindTooltip(formatMarkerLabel(center), {
      permanent: true,
      direction: 'top',
      offset: [0, -10],
      className: `marker-label ${areaLabelClasses[center.area] || 'marker-label--default'}`
    })
  }

  marker.on('click', () => {
    selectedCenter.value = center
    emit('select-center', center)
  })

  return marker
}

function formatMarkerLabel(center) {
  return `
    <div class="marker-label-content">
      <div><b>Country:</b> ${escapeHtml(center.country)}</div>
      <div><b>Institution:</b> ${escapeHtml(center.institution)}</div>
      <div><b>Provider:</b> ${escapeHtml(center.provider)}</div>
    </div>
  `
}

function renderMarkers() {
  if (!map) return

  if (markerLayer) {
    map.removeLayer(markerLayer)
    markerLayer = null
  }

  const markers = props.centers.map(createMarker)

  if (props.useClusters) {
    markerLayer = L.markerClusterGroup({
      showCoverageOnHover: false,
      maxClusterRadius: 52,
      spiderfyOnMaxZoom: true
    })
    markerLayer.addLayers(markers)
  } else {
    markerLayer = L.layerGroup(markers)
  }

  markerLayer.addTo(map)

  if (props.centers.length > 0) {
    fitToMarkers()
  }

  scheduleUpdateLabels()
}

function fitToMarkers() {
  if (!map || !props.centers.length) return

  const bounds = L.latLngBounds(props.centers.map((item) => [item.latitude, item.longitude]))
  map.fitBounds(bounds, {
    padding: [60, 60],
    maxZoom: props.centers.length === 1 ? 7 : 5
  })
}

function fitGlobal() {
  if (!map) return
  map.setView(areaViews.Global.center, areaViews.Global.zoom)
}

function focusCenter(center) {
  if (!map) return
  map.setView([center.latitude, center.longitude], 7)
  selectedCenter.value = center
  emit('select-center', center)
}

function scheduleUpdateLabels() {
  if (labelFrame) {
    cancelAnimationFrame(labelFrame)
  }
  labelFrame = requestAnimationFrame(updateLabels)
}

function updateLabels() {
  labelFrame = null

  if (!map || !props.showLabels || props.labelMode !== 'callout') {
    labelItems.value = []
    labelLines.value = []
    return
  }

  const size = map.getSize()
  const boxes = props.centers
    .map((center, index) => {
      const point = map.latLngToContainerPoint([center.latitude, center.longitude])

      return {
        id: center.id,
        index,
        center,
        anchorX: point.x,
        anchorY: point.y,
        x: 0,
        y: 0,
        width: LABEL_WIDTH,
        height: LABEL_HEIGHT
      }
    })

  arrangeLabelGrid(boxes, size)

  labelItems.value = boxes
    .sort((a, b) => a.index - b.index)
    .map((box) => ({
      id: box.id,
      x: Math.round(box.x),
      y: Math.round(box.y),
      country: box.center.country,
      institution: box.center.institution,
      provider: box.center.provider,
      className: areaLabelClasses[box.center.area] || 'marker-label--default'
    }))

  labelLines.value = boxes.map((box) => {
    const target = nearestPointOnBox(box.anchorX, box.anchorY, box)
    return {
      id: box.id,
      x1: Math.round(box.anchorX),
      y1: Math.round(box.anchorY),
      x2: Math.round(target.x),
      y2: Math.round(target.y)
    }
  })
}

function arrangeLabelGrid(boxes, size) {
  const columnCount = Math.max(1, Math.floor((size.x - LABEL_GAP) / (LABEL_WIDTH + LABEL_GAP)))
  const visibleRowCount = Math.max(
    1,
    Math.floor((size.y - LABEL_TOP_SAFE - LABEL_GAP + LABEL_GAP) / (LABEL_HEIGHT + LABEL_GAP))
  )
  const rowCount = Math.max(visibleRowCount, Math.ceil(boxes.length / columnCount))
  const usableWidth = columnCount * LABEL_WIDTH + (columnCount - 1) * LABEL_GAP
  const startX = Math.max(LABEL_GAP, (size.x - usableWidth) / 2)
  const cells = []

  for (let row = 0; row < rowCount; row += 1) {
    for (let column = 0; column < columnCount; column += 1) {
      cells.push({
        x: startX + column * (LABEL_WIDTH + LABEL_GAP),
        y: LABEL_TOP_SAFE + row * (LABEL_HEIGHT + LABEL_GAP),
        centerX: startX + column * (LABEL_WIDTH + LABEL_GAP) + LABEL_WIDTH / 2,
        centerY: LABEL_TOP_SAFE + row * (LABEL_HEIGHT + LABEL_GAP) + LABEL_HEIGHT / 2
      })
    }
  }

  boxes
    .slice()
    .sort((a, b) => distanceToNearestCell(b, cells) - distanceToNearestCell(a, cells))
    .forEach((box) => {
      const cellIndex = findBestCellIndex(box, cells)
      const [cell] = cells.splice(cellIndex, 1)
      box.x = cell.x
      box.y = cell.y
    })
}

function findBestCellIndex(box, cells) {
  return cells.reduce((bestIndex, cell, index) => {
    const bestCell = cells[bestIndex]
    return cellCost(box, cell) < cellCost(box, bestCell) ? index : bestIndex
  }, 0)
}

function distanceToNearestCell(box, cells) {
  return Math.min(...cells.map((cell) => cellCost(box, cell)))
}

function cellCost(box, cell) {
  const dx = box.anchorX - cell.centerX
  const dy = box.anchorY - cell.centerY
  return Math.sqrt(dx * dx + dy * dy)
}

function relaxLabelBoxes(boxes, maxX, maxY) {
  for (let iteration = 0; iteration < 140; iteration += 1) {
    for (let i = 0; i < boxes.length; i += 1) {
      for (let j = i + 1; j < boxes.length; j += 1) {
        separateBoxes(boxes[i], boxes[j])
      }
    }

    boxes.forEach((box) => {
      box.x += (box.idealX - box.x) * 0.012
      box.y += (box.idealY - box.y) * 0.012
      box.x = clamp(box.x, LABEL_GAP, maxX)
      box.y = clamp(box.y, LABEL_GAP, maxY)
    })
  }
}

function separateBoxes(a, b) {
  const overlapX = Math.min(a.x + a.width + LABEL_GAP, b.x + b.width + LABEL_GAP) - Math.max(a.x, b.x)
  const overlapY = Math.min(a.y + a.height + LABEL_GAP, b.y + b.height + LABEL_GAP) - Math.max(a.y, b.y)

  if (overlapX <= 0 || overlapY <= 0) return

  const aCenterX = a.x + a.width / 2
  const bCenterX = b.x + b.width / 2
  const aCenterY = a.y + a.height / 2
  const bCenterY = b.y + b.height / 2

  if (overlapX < overlapY) {
    const direction = aCenterX <= bCenterX ? -1 : 1
    const shift = overlapX / 2
    a.x += direction * shift
    b.x -= direction * shift
  } else {
    const direction = aCenterY <= bCenterY ? -1 : 1
    const shift = overlapY / 2
    a.y += direction * shift
    b.y -= direction * shift
  }
}

function nearestPointOnBox(x, y, box) {
  return {
    x: clamp(x, box.x, box.x + box.width),
    y: clamp(y, box.y, box.y + box.height)
  }
}

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max)
}

function escapeHtml(value) {
  return String(value || '')
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;')
    .replaceAll("'", '&#039;')
}

defineExpose({
  focusCenter,
  fitToMarkers,
  fitGlobal
})
</script>
