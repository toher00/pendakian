<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Asisten Pendaki Offline & Online</title>
  <!-- Leaflet CSS untuk Peta Interaktif -->
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: Arial, sans-serif; }
    body { background-color: #121212; color: #fff; display: flex; flex-direction: column; height: 100vh; }
    header { background-color: #1e293b; padding: 12px; text-align: center; font-weight: bold; font-size: 1.1rem; border-bottom: 1px solid #334155; }
    
    /* Panel Informasi Utama */
    .status-bar { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; padding: 10px; background-color: #1e293b; text-align: center; }
    .status-card { background: #0f172a; padding: 8px; border-radius: 8px; border: 1px solid #334155; }
    .status-card h4 { font-size: 0.75rem; color: #94a3b8; margin-bottom: 4px; }
    .status-card p { font-size: 0.95rem; font-weight: bold; color: #38bdf8; }

    /* Area Peta Navigasi */
    #map { flex: 1; width: 100%; background: #242f3e; }

    /* Control Panel di Bagian Bawah */
    .controls { padding: 10px; background: #1e293b; display: flex; gap: 8px; border-top: 1px solid #334155; }
    .btn { flex: 1; padding: 10px; border: none; border-radius: 6px; font-weight: bold; cursor: pointer; color: white; background: #2563eb; font-size: 0.85rem; }
    .btn-emerald { background: #059669; }
    .btn-rose { background: #e11d48; }

    /* Status Koneksi */
    .mode-tag { font-size: 0.7rem; padding: 2px 6px; border-radius: 4px; margin-left: 5px; }
    .online { background: #16a34a; }
    .offline { background: #dc2626; }
  </style>
</head>
<body>

  <header>
    ⛰️ Navigasi Pendaki 
    <span id="connection-status" class="mode-tag online">ONLINE</span>
  </header>

  <!-- Dashboard Informasi Realtime -->
  <div class="status-bar">
    <div class="status-card">
      <h4>KETINGGIAN</h4>
      <p id="altitude">-- mdpl</p>
    </div>
    <div class="status-card">
      <h4>CUACA / SUHU</h4>
      <p id="weather-info">--°C</p>
    </div>
    <div class="status-card">
      <h4>POS TERDEKAT</h4>
      <p id="nearest-pos">--</p>
    </div>
  </div>

  <!-- Peta Offline / Online -->
  <div id="map"></div>

  <!-- Tombol Aksi Kunci -->
  <div class="controls">
    <button class="btn" onclick="locateUser()">📍 Cek Posisi Saya</button>
    <button class="btn btn-emerald" onclick="estimateWeatherOffline()">🌡️ Estimasi Suhu</button>
    <button class="btn btn-rose" onclick="alertSOS()">🚨 SOS Emergency</button>
  </div>

  <!-- Leaflet JS -->
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script>
    let map, userMarker, watchId;
    let currentAlt = 0;

    // Database Jalur & Pos Pendakian (Bisa ditambah sesuai kebutuhan)
    const posList = [
      { name: "Basecamp", lat: -7.5800, lng: 112.5800, alt: 1300 },
      { name: "Pos 1 (Shetler 1)", lat: -7.5850, lng: 112.5830, alt: 1650 },
      { name: "Pos 2 (Mata Air)", lat: -7.5910, lng: 112.5860, alt: 2100 },
      { name: "Pos 3 (Camp Area)", lat: -7.5960, lng: 112.5890, alt: 2600 },
      { name: "Puncak", lat: -7.6010, lng: 112.5920, alt: 3150 }
    ];

    // Initialize Map
    function initMap() {
      // Inisialisasi Peta
      map = L.map('map').setView([-7.5800, 112.5800], 14);

      // Layer Peta Topografi (OpenTopoMap)
      L.tileLayer('https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png', {
        maxZoom: 17,
        attribution: 'Map data: © OpenStreetMap'
      }).addTo(map);

      // Plot Jalur dan Pos di Peta
      const latlngs = [];
      posList.forEach(pos => {
        L.marker([pos.lat, pos.lng]).addTo(map)
          .bindPopup(`<b>${pos.name}</b><br>Ketinggian: ${pos.alt} mdpl`);
        latlngs.push([pos.lat, pos.lng]);
      });

      // Garis Jalur Pendakian
      L.polyline(latlngs, {color: '#f59e0b', weight: 4, dashArray: '5, 10'}).addTo(map);

      // Deteksi Koneksi Internet
      window.addEventListener('online', updateOnlineStatus);
      window.addEventListener('offline', updateOnlineStatus);
      updateOnlineStatus();
      
      // Auto-start GPS Tracking
      startTracking();
    }

    // Monitoring Status Online/Offline
    function updateOnlineStatus() {
      const statusEl = document.getElementById('connection-status');
      if (navigator.onLine) {
        statusEl.textContent = 'ONLINE';
        statusEl.className = 'mode-tag online';
        getRealtimeWeather();
      } else {
        statusEl.textContent = 'OFFLINE';
        statusEl.className = 'mode-tag offline';
        estimateWeatherOffline();
      }
    }

    // Navigasi & GPS Realtime
    function startTracking() {
      if ('geolocation' in navigator) {
        watchId = navigator.geolocation.watchPosition(position => {
          const lat = position.coords.latitude;
          const lng = position.coords.longitude;
          currentAlt = Math.round(position.coords.altitude || 1300);

          document.getElementById('altitude').textContent = `${currentAlt} mdpl`;

          // Update/Buat Marker Posisi User
          if (!userMarker) {
            userMarker = L.circleMarker([lat, lng], { radius: 8, color: '#00ffff', fillColor: '#0284c7', fillOpacity: 1 }).addTo(map);
          } else {
            userMarker.setLatLng([lat, lng]);
          }

          findNearestPos(lat, lng);
        }, err => console.log(err), { enableHighAccuracy: true });
      }
    }

    function locateUser() {
      if (userMarker) {
        map.setView(userMarker.getLatLng(), 16);
      } else {
        alert("Mencari sinyal GPS HP Anda...");
      }
    }

    // Cari Pos Terdekat
    function findNearestPos(userLat, userLng) {
      let minDistance = Infinity;
      let nearest = null;

      posList.forEach(pos => {
        const dist = Math.hypot(pos.lat - userLat, pos.lng - userLng);
        if (dist < minDistance) {
          minDistance = dist;
          nearest = pos;
        }
      });

      if (nearest) {
        document.getElementById('nearest-pos').textContent = nearest.name;
      }
    }

    // Fitur Cuaca Realtime (Saat Online)
    function getRealtimeWeather() {
      // Menggunakan Open-Meteo API Gratis (Tanpa API Key)
      fetch('https://api.open-meteo.com/v1/forecast?latitude=-7.58&longitude=112.58&current_weather=true')
        .then(res => res.json())
        .then(data => {
          const temp = Math.round(data.current_weather.temperature);
          document.getElementById('weather-info').textContent = `${temp}°C (Live)`;
        })
        .catch(() => estimateWeatherOffline());
    }

    // Fitur Cuaca Offline (Logika Hukum Adiabatik Ketinggian)
    function estimateWeatherOffline() {
      // Setiap naik 100 mdpl, suhu turun ±0.65°C dari suhu basecamp (asumsi basecamp 27°C pada 0 mdpl)
      const baseTemp = 27; 
      const estimatedTemp = Math.round(baseTemp - ((currentAlt / 100) * 0.65));
      document.getElementById('weather-info').textContent = `~${estimatedTemp}°C (Est)`;
    }

    // Fitur Darurat / SOS
    function alertSOS() {
      alert(`🚨 PANGGILAN DARURAT 🚨\nKetinggian Anda saat ini: ${currentAlt} mdpl.\nTetap tenang, diam di tempat, dan hemat baterai HP Anda!`);
    }

    window.onload = initMap;
  </script>
</body>
</html>
