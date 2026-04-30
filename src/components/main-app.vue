<script setup>
import { ref, onMounted } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

// Fix untuk Ikon Marker Leaflet
import markerIcon from 'leaflet/dist/images/marker-icon.png';
import markerShadow from 'leaflet/dist/images/marker-shadow.png';
let DefaultIcon = L.icon({
  iconUrl: markerIcon,
  shadowUrl: markerShadow,
  iconSize: [25, 41],
  iconAnchor: [12, 41]
});
L.Marker.prototype.options.icon = DefaultIcon;

const apiKey = import.meta.env.VITE_OPENWEATHER_API_KEY;

// State
const weather = ref(null);
const forecast = ref([]);
const searchQuery = ref('');
const loading = ref(false);
const isLocating = ref(false);
let map = null;
let currentMarker = null;

// Fungsi Mengambil Data Cuaca & Forecast
const fetchData = async (lat, lon) => {
  try {
    const weatherRes = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&appid=${apiKey}&units=metric&lang=id`
    );
    weather.value = await weatherRes.json();

    const forecastRes = await fetch(
      `https://api.openweathermap.org/data/2.5/forecast?lat=${lat}&lon=${lon}&appid=${apiKey}&units=metric&lang=id`
    );
    const forecastData = await forecastRes.json();
    forecast.value = forecastData.list.filter(item => item.dt_txt.includes("12:00:00"));
  } catch (error) {
    console.error("Gagal memuat data:", error);
  }
};

// Fungsi Update Peta & Marker
const updateMap = (lat, lon, name) => {
  const coords = [lat, lon];
  if (map) {
    map.setView(coords, 12);
    if (currentMarker) map.removeLayer(currentMarker);
    currentMarker = L.marker(coords).addTo(map).bindPopup(name).openPopup();
    fetchData(lat, lon);
  }
};

// Fungsi Cari Lokasi via Nominatim
const searchLocation = async () => {
  if (!searchQuery.value) return;
  loading.value = true;
  try {
    const response = await fetch(
      `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(searchQuery.value)}`
    );
    const data = await response.json();
    if (data.length > 0) {
      updateMap(data[0].lat, data[0].lon, data[0].display_name);
    } else {
      alert("Lokasi tidak ditemukan");
    }
  } catch (error) {
    console.error("Error search:", error);
  } finally {
    loading.value = false;
  }
};

// Fungsi Meminta Lokasi User (GPS)
const getUserLocation = () => {
  isLocating.value = true;
  if (!navigator.geolocation) {
    alert("Geolocation tidak didukung oleh browser Anda");
    initDefaultMap();
    return;
  }

  navigator.geolocation.getCurrentPosition(
    (position) => {
      const { latitude, longitude } = position.coords;
      updateMap(latitude, longitude, "Lokasi Anda Sekarang");
      isLocating.value = false;
    },
    (error) => {
      console.warn("Akses lokasi ditolak:", error.message);
      initDefaultMap(); // Fallback ke default jika ditolak
      isLocating.value = false;
    },
    { enableHighAccuracy: true, timeout: 5000, maximumAge: 0 }
  );
};

// Inisialisasi Peta Default (Jakarta)
const initDefaultMap = () => {
  const defaultCoords = [-6.2000, 106.8166];
  map.setView(defaultCoords, 10);
  updateMap(defaultCoords[0], defaultCoords[1], "Jakarta (Default)");
};

// Helper Format Hari
const formatDate = (dateStr) => {
  const days = ['Minggu', 'Senin', 'Selasa', 'Rabu', 'Kamis', 'Jumat', 'Sabtu'];
  const d = new Date(dateStr);
  return days[d.getDay()];
};

onMounted(() => {
  // Setup dasar peta
  map = L.map('map-container').setView([-6.2000, 106.8166], 10);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors'
  }).addTo(map);

  map.on('click', (e) => {
    updateMap(e.latlng.lat, e.latlng.lng, "Lokasi Terpilih");
  });

  // Jalankan deteksi lokasi otomatis
  getUserLocation();
});
</script>

<template>
 <div class="app-wrapper">
    <header class="main-header">
      <div class="brand">
        <h1>Pro Weather App</h1>
        <p v-if="isLocating" class="locating-text">Mendeteksi lokasi Anda...</p>
      </div>
      
      <div class="search-wrapper"> <div class="search-container">
          <input 
            v-model="searchQuery" 
            @keyup.enter="searchLocation"
            type="text" 
            placeholder="Cari Kota..." 
          />
          <button @click="searchLocation" :disabled="loading" class="btn-search">
            <span v-if="!loading">Cari</span>
            <span v-else>...</span>
          </button>
        </div>
        <button @click="getUserLocation" class="btn-gps" title="Gunakan Lokasi Saya">
          📍
        </button>
      </div>
    </header>

    <main class="content">
      <!-- Info Cuaca Utama -->
      <section v-if="weather" class="current-weather">
        <div class="weather-top">
          <div class="location">
            <h2>{{ weather.name }}</h2>
            <p>{{ formatDate(new Date()) }}, {{ new Date().toLocaleDateString('id-ID') }}</p>
          </div>
          <div class="main-stats">
            <img :src="`http://openweathermap.org/img/wn/${weather.weather[0].icon}@4x.png`" alt="Cuaca" />
            <div class="temp-big">{{ Math.round(weather.main.temp) }}°C</div>
          </div>
          <p class="description">{{ weather.weather[0].description }}</p>
        </div>

        <div class="weather-grid">
          <div class="grid-item">
            <span class="label">Kelembapan</span>
            <span class="value">{{ weather.main.humidity }}%</span>
          </div>
          <div class="grid-item">
            <span class="label">Angin</span>
            <span class="value">{{ weather.wind.speed }} m/s</span>
          </div>
          <div class="grid-item">
            <span class="label">Terasa</span>
            <span class="value">{{ Math.round(weather.main.feels_like) }}°C</span>
          </div>
          <div class="grid-item">
            <span class="label">Tekanan</span>
            <span class="value">{{ weather.main.pressure }} hPa</span>
          </div>
        </div>
      </section>

      <!-- Prakiraan 5 Hari -->
      <section v-if="forecast.length" class="forecast-section">
        <h3>Prakiraan 5 Hari (12:00)</h3>
        <div class="forecast-list">
          <div v-for="day in forecast" :key="day.dt" class="forecast-card">
            <span class="f-day">{{ formatDate(day.dt_txt) }}</span>
            <img :src="`http://openweathermap.org/img/wn/${day.weather[0].icon}.png`" alt="Icon" />
            <span class="f-temp">{{ Math.round(day.main.temp) }}°C</span>
          </div>
        </div>
      </section>

      <!-- Peta -->
      <div id="map-container"></div>
    </main>
  </div>
</template>

<style>
@import url('/src/components/main-app.css');
</style>