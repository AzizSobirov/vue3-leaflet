<template>
  <div class="w-screen h-screen">
    <div id="map"></div>
  </div>
</template>

<script setup>
import { onMounted } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

let map
let geoLayer
let clusterGroup

onMounted(async () => {
  map = L.map('map').setView([41.3, 64.6], 6)

  // Tile layer
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 18,
  }).addTo(map)

  // Viloyatlar GeoJSON yuklash
  const res = await fetch('/geojson/uz_regions.geojson')
  const regionsData = await res.json()

  geoLayer = L.geoJSON(regionsData, {
    style: {
      color: '#0077ff',
      weight: 2,
      fillColor: '#00aaff55',
    },
    onEachFeature: (feature, layer) => {
      layer.on({
        click: () => onRegionClick(feature, layer),
      })
      layer.bindTooltip(feature.properties.name)
    },
  }).addTo(map)
})

async function onRegionClick(feature, layer) {
  const regionName = feature.properties.name
  const regionId = regionName?.toLowerCase().split(' ')[0]

  // Zoom-in
  map.fitBounds(layer.getBounds())

  // 🧹 Oldingi layerlarni tozalash
  if (geoLayer) map.removeLayer(geoLayer)
  if (clusterGroup) map.removeLayer(clusterGroup)

  // Tumanlar GeoJSON yuklash (local)
  const res = await fetch(`/geojson/districts/${regionId}.geojson`)
  const districtData = await res.json()

  geoLayer = L.geoJSON(districtData, {
    style: {
      color: '#ff6600',
      weight: 2,
      fillColor: '#ffaa0044',
    },
    onEachFeature: (feature, layer) => {
      layer.on({
        click: () => showDistrictMarkers(feature, layer),
      })
      layer.bindTooltip(feature.properties.name)
    },
  }).addTo(map)
}

function showDistrictMarkers(feature, layer) {
  if (clusterGroup) map.removeLayer(clusterGroup)

  // Tumanga zoom
  map.fitBounds(layer.getBounds())

  // 📍 Loyihalar (demo)
  const projects = [
    { coords: [41.31, 69.24], name: 'Project A' },
    { coords: [41.315, 69.245], name: 'Project B' },
    { coords: [41.33, 69.25], name: 'Project C' },
  ]

  // 🔵 Cluster group
  clusterGroup = L.markerClusterGroup()

  projects.forEach((p) => {
    const marker = L.marker(p.coords).bindPopup(`<b>${p.name}</b>`)
    clusterGroup.addLayer(marker)
  })

  map.addLayer(clusterGroup)
}
</script>

<style>
#map {
  width: 100%;
  height: 100%;
}
</style>
