<template>
  <div class="map-container">
    <div id="map"></div>

    <div class="left-overlay">
      <h2>Search location</h2>

      <div class="controls">
        <input
          v-model="searchQuery"
          placeholder="Search city or place"
        />

        <button @click="searchPlace">
          Search
        </button>
      </div>

      <h2>Coordinates</h2>

      <div class="controls">
        <input
          v-model="latitude"
          placeholder="Latitude"
        />

        <input
          v-model="longitude"
          placeholder="Longitude"
        />

        <button @click="goToCoordinates">
          Go
        </button>
      </div>
    </div>

    <div
      v-if="selectedPlace"
      class="right-popup"
    >
      <h2>Location details</h2>

      <p>
        <strong>Name:</strong>
        {{ selectedPlace.properties.name }}
      </p>

      <p>
        <strong>City:</strong>
        {{ selectedPlace.properties.city }}
      </p>

      <p>
        <strong>Country:</strong>
        {{ selectedPlace.properties.country }}
      </p>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

import maplibregl, {
  type Map,
  type Marker,
} from 'maplibre-gl'

import 'maplibre-gl/dist/maplibre-gl.css'

interface PhotonFeature {
  geometry: {
    coordinates: [number, number]
  }

  properties: {
    name?: string
    city?: string
    country?: string
  }
}

export default Vue.extend({
  name: 'MapView',

  data() {
    return {
      map: null as Map | null,

      marker: null as Marker | null,

      selectedPlace: null as PhotonFeature | null,

      searchQuery: '',

      latitude: '',

      longitude: '',
    }
  },

  mounted() {
    this.map = new maplibregl.Map({
      container: 'map',

      style: 'https://tiles.openfreemap.org/styles/liberty',

      center: [21.23, 45.75],

      zoom: 13,
    })

    this.map.on('click', async (e) => {
      const { lng, lat } = e.lngLat

      await this.loadPlaceFromCoordinates(
        lat,
        lng
      )
    })
  },

  methods: {
    async loadPlaceFromCoordinates(
      lat: number,
      lng: number
    ) {
      this.updateMarker(lat, lng)

      this.map?.flyTo({
        center: [lng, lat],

        zoom: 13,
      })

      const response = await fetch(
        `https://photon.komoot.io/reverse?lat=${lat}&lon=${lng}`
      )

      const data = await response.json()

      this.selectedPlace =
        data.features[0] || null
    },

    updateMarker(
      lat: number,
      lng: number
    ) {
      this.marker?.remove()

      this.marker = new maplibregl.Marker()
        .setLngLat([lng, lat])
        .addTo(this.map!)
    },

    async searchPlace() {
      if (!this.searchQuery) {
        return
      }

      const response = await fetch(
        `https://photon.komoot.io/api/?q=${this.searchQuery}`
      )

      const data = await response.json()

      const feature = data.features[0]

      if (!feature) {
        return
      }

      this.selectedPlace = feature

      const [lng, lat] =
        feature.geometry.coordinates

      this.latitude = String(lat)

      this.longitude = String(lng)

      this.updateMarker(lat, lng)

      this.map?.flyTo({
        center: [lng, lat],

        zoom: 13,
      })
    },

    async goToCoordinates() {
      const lat = Number(this.latitude)

      const lng = Number(this.longitude)

      if (
        Number.isNaN(lat) ||
        Number.isNaN(lng)
      ) {
        return
      }

      await this.loadPlaceFromCoordinates(
        lat,
        lng
      )
    },
  },
})
</script>

<style scoped>
.map-container {
  position: relative;
  width: 100%;
  height: 100vh;
}

#map {
  width: 100%;
  height: 100%;
}

/* LEFT PANEL (desktop) */
.left-overlay {
  position: absolute;
  top: 20px;
  left: 20px;
  width: 320px;
  padding: 20px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
  z-index: 10;
}

/* RIGHT POPUP (desktop) */
.right-popup {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 320px;
  padding: 20px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
  z-index: 10;
}

/* controls */
.controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 24px;
}

input {
  padding: 12px;
  border: 1px solid #ccc;
  border-radius: 10px;
  font-size: 14px;
}

button {
  padding: 12px;
  border: none;
  border-radius: 10px;
  background: #222;
  color: white;
  cursor: pointer;
  font-size: 14px;
}

button:hover {
  opacity: 0.9;
}

/* ===================== */
/* MOBILE RESPONSIVE FIX */
/* ===================== */
@media (max-width: 768px) {
  .left-overlay {
    position: absolute;
    top: 10px;
    left: 10px;
    right: 10px;
    width: auto;
    max-height: 45vh;
    overflow-y: auto;
  }

  .right-popup {
    position: absolute;
    left: 10px;
    right: 10px;
    bottom: calc(20px + env(safe-area-inset-bottom));
    top: auto;
    width: auto;
    max-height: 40vh;
    overflow-y: auto;
    z-index: 10;
  }

  input,
  button {
    font-size: 16px;
  }

  button {
    min-height: 44px;
  }
}
</style>
