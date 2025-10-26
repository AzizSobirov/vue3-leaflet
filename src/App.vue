<template>
  <div id="map"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

let map
let regionsLayer
let districtsLayer
let regionMarkers = []

let lastClickedRegion = null
let selectedRegionLayer = null

onMounted(async () => {
  map = L.map('map').setView([41.3, 64.6], 6)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 18,
  }).addTo(map)

  await loadRegions()

  // 🔙 Dastlabki holatga qaytish uchun map click
  map.on('click', (e) => {
    if (map.getZoom() <= 6) return

    // Agar region ichida click bo‘lmagan bo‘lsa
    if (!e.originalEvent._clickedRegion) {
      console.log('Outside region clicked')

      lastClickedRegion = null
      resetToRegions()
    }
  })
})

// 🗺️ Viloyatlar yuklash
async function loadRegions() {
  const res = await fetch('/geojson/uz_regions.geojson')
  const data = await res.json()

  // Viloyatlar chizish
  regionsLayer = L.geoJSON(data, {
    style: {
      color: '#555',
      weight: 1,
      fillColor: '#ccc',
      fillOpacity: 0.7,
    },
    onEachFeature: (feature, layer) => {
      layer.on('click', (e) => {
        if (selectedRegionLayer) {
          regionsLayer.resetStyle(selectedRegionLayer)
        }

        selectedRegionLayer = e.target

        layer.setStyle({
          fillColor: 'white',
          fillOpacity: 0.9,
        })

        lastClickedRegion = feature
        onRegionClick(feature, layer)
        e.originalEvent._clickedRegion = true // flag
      })
    },
  }).addTo(map)

  // Har bir viloyatga “cluster-style” marker
  data.features.forEach((f) => {
    const center = L.geoJSON(f).getBounds().getCenter()
    const icon = L.divIcon({
      html: `
        <div style="
          background:#e7f1ff;
          border:2px solid #3a8dff;
          color:#3a8dff;
          border-radius:50%;
          width:45px;
          height:45px;
          display:flex;
          align-items:center;
          justify-content:center;
          font-weight:600;
          font-family:sans-serif;
        ">${Math.floor(Math.random() * 300)}</div>`,
      className: '',
      iconSize: [45, 45],
    })
    const marker = L.marker(center, { icon }).addTo(map)
    regionMarkers.push(marker)
  })
}

// Viloyat tanlandi
async function onRegionClick(feature, layer) {
  // Zoom viloyatga
  map.fitBounds(layer.getBounds())

  // Oldingi layerlarni tozalash
  clearMap()

  // Tumanlar yuklash
  const regionName = feature.properties.name
  const regionId = regionName.toLowerCase().split(' ')[0]
  const res = await fetch(`/geojson/districts/${regionId}.geojson`)
  const districtData = await res.json()

  setTimeout(() => {
    districtsLayer = L.geoJSON(districtData, {
      style: {
        color: '#ff6600',
        weight: 1,
        fillColor: '#ffbb3344',
      },
      onEachFeature: (f, l) => {
        l.bindTooltip(f.properties.name)
      },
    }).addTo(map)

    // Tumanga markerlar (fake)
    const markers = []
    for (let i = 0; i < 10; i++) {
      const [lat, lng] = randomInBounds(layer.getBounds())
      const marker = L.marker([lat, lng]).bindPopup(`<b>Loyiha ${i + 1}</b>`)
      markers.push(marker)
    }

    // const group = L.markerClusterGroup()
    // markers.forEach((m) => group.addLayer(m))
    // map.addLayer(group)
  }, 100)
}

// Dastlabki holatga qaytish
function resetToRegions() {
  clearMap()
  loadRegions()
  map.setView([41.3, 64.6], 6)
}

// 🧹 Layer va markerlarni tozalash
function clearMap() {
  // if (regionsLayer) map.removeLayer(regionsLayer)
  if (districtsLayer) map.removeLayer(districtsLayer)
  regionMarkers.forEach((m) => map.removeLayer(m))
  regionMarkers = []
}

// Random koordinata olish (tuman markazlari uchun)
function randomInBounds(bounds) {
  const lat = bounds.getSouth() + Math.random() * (bounds.getNorth() - bounds.getSouth())
  const lng = bounds.getWest() + Math.random() * (bounds.getEast() - bounds.getWest())
  return [lat, lng]
}
</script>

<style>
#map {
  width: 100%;
  height: 100vh;
}
</style>
