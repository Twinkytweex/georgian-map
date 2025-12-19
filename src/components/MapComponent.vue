<template>
  <div class="map-wrapper">
    <div v-if="loading" class="loading-spinner">Loading Map Data...</div>

    <div ref="mapContainer" class="map-container"></div>

    <div 
      class="hover-card" 
      :class="{ active: hoverCard.visible }"
      :style="{ top: hoverCard.y + 'px', left: hoverCard.x + 'px' }"
    >
      <img :src="hoverCard.img" alt="Region" class="card-img">
      <div class="card-content">
        <h3 class="card-title">{{ hoverCard.title }}</h3>
        <button class="card-btn" @click="openModal">View Flag Details</button>
      </div>
    </div>

    <div 
      class="modal-overlay" 
      :class="{ active: modal.visible }" 
      @click.self="closeModal"
    >
      <div class="modal-box">
        <span class="close-btn" @click="closeModal">&times;</span>
        <h2 class="modal-title">{{ modal.title }}</h2>
        <p class="modal-desc">{{ modal.desc }}</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css'; // Import Leaflet CSS

// --- STATE MANAGEMENT ---
const mapContainer = ref(null);
const loading = ref(true);
let map = null;
let geojsonLayer = null;

// Reactive state for the Hover Card
const hoverCard = reactive({
  visible: false,
  x: 0,
  y: 0,
  title: '',
  img: '',
  regionKey: '' // Store the key to look up data for the modal
});

// Reactive state for the Modal
const modal = reactive({
  visible: false,
  title: '',
  desc: ''
});

// --- DATABASE ---
const database = {
  "Tbilisi": { 
    img: "https://upload.wikimedia.org/wikipedia/commons/thumb/d/d4/Tbilisi_View_from_Narikala.jpg/320px-Tbilisi_View_from_Narikala.jpg", 
    desc: "Tbilisi is the capital and the largest city of Georgia, lying on the banks of the Kura River. Its flag features a red cross on a white background with a central emblem depicting a pheasant and a falcon." 
  },
  "Batumi": { 
    img: "https://upload.wikimedia.org/wikipedia/commons/thumb/0/07/Batumi_from_alphabet_tower.jpg/320px-Batumi_from_alphabet_tower.jpg", 
    desc: "Batumi is the capital of the Autonomous Republic of Adjara. The flag of Adjara displays the national Five Cross Flag of Georgia in the canton, with dark blue stripes symbolizing the Black Sea." 
  },
  "Kutaisi": { 
    img: "https://upload.wikimedia.org/wikipedia/commons/thumb/c/c6/Kutaisi_Bagrati_Cathedral.jpg/320px-Kutaisi_Bagrati_Cathedral.jpg", 
    desc: "Kutaisi is one of the oldest continuously inhabited cities in the world. It served as the legislative capital from 2012 to 2018." 
  },
  "Mestia": { 
    img: "https://upload.wikimedia.org/wikipedia/commons/thumb/9/91/Mestia_Svaneti_Georgia.jpg/320px-Mestia_Svaneti_Georgia.jpg", 
    desc: "Mestia is a highland townlet in northwest Georgia, famous for its medieval stone defensive towers." 
  },
  "default": {
    img: "https://upload.wikimedia.org/wikipedia/commons/thumb/0/0f/Flag_of_Georgia.svg/320px-Flag_of_Georgia.svg.png",
    desc: "This is a municipality of Georgia. Specific detailed flag information for this region is not yet in this demo database."
  }
};

// --- LOGIC ---

// 1. Mouse Tracking for the Card
const updateMousePosition = (e) => {
  if (!hoverCard.visible) return;

  const x = e.clientX + 15;
  const y = e.clientY + 15;
  
  // Viewport boundary checks to keep card on screen
  const cardWidth = 250;
  const cardHeight = 220; // Approx height
  
  const finalX = (x + cardWidth > window.innerWidth) ? x - cardWidth - 20 : x;
  const finalY = (y + cardHeight > window.innerHeight) ? y - cardHeight - 20 : y;

  hoverCard.x = finalX;
  hoverCard.y = finalY;
};

// 2. Leaflet Style Function
const styleFeature = () => {
  return {
    fillColor: 'transparent', 
    weight: 1.5,
    opacity: 1,
    color: '#555',
    fillOpacity: 0
  };
};

// 3. Hover Highlight Logic
const highlightFeature = (e) => {
  const layer = e.target;
  layer.setStyle({
    weight: 3,
    color: '#e74c3c',
    fillOpacity: 0.1,
    fillColor: '#e74c3c'
  });
  layer.bringToFront();

  // Get Data
  const regionName = layer.feature.properties.name || layer.feature.properties.NAME_2 || "Unknown Region";
  const data = database[regionName] || database["default"];

  // Update Reactive State
  hoverCard.title = regionName;
  hoverCard.img = data.img;
  hoverCard.regionKey = regionName;
  hoverCard.visible = true;
};

// 4. Reset Highlight Logic
const resetHighlight = (e) => {
  geojsonLayer.resetStyle(e.target);
  hoverCard.visible = false;
};

// 5. Open Modal
const openModal = () => {
  if (!hoverCard.regionKey) return;
  
  const regionName = hoverCard.regionKey;
  const data = database[regionName] || database["default"];

  modal.title = `${regionName} Details`;
  modal.desc = data.desc;
  
  // Hide hover card immediately so it doesn't overlap modal
  hoverCard.visible = false;
  modal.visible = true;
};

// 6. Close Modal
const closeModal = () => {
  modal.visible = false;
};

// 7. Initialize Map
onMounted(() => {
  // Global mouse listener for smooth card tracking
  document.addEventListener('mousemove', updateMousePosition);

  // Init Leaflet
  map = L.map(mapContainer.value, {
    center: [42.1, 43.5], 
    zoom: 7.5,            
    zoomControl: false,
    attributionControl: false,
    doubleClickZoom: false,
    scrollWheelZoom: 'center'
  });

  // Fetch Data
  const geoJsonUrl = "https://raw.githubusercontent.com/bumbeishvili/geojson-georgian-regions/master/SubRegions_low_quality.json";

  fetch(geoJsonUrl)
    .then(res => res.json())
    .then(data => {
      loading.value = false;
      
      geojsonLayer = L.geoJson(data, {
        style: styleFeature,
        onEachFeature: (feature, layer) => {
          layer.on({
            mouseover: highlightFeature,
            mouseout: resetHighlight,
            click: L.DomEvent.stopPropagation // Prevent map zoom on click
          });
        }
      }).addTo(map);

      map.fitBounds(geojsonLayer.getBounds());
    })
    .catch(err => {
      loading.value = false;
      console.error("Error loading GeoJSON:", err);
    });
});

// Cleanup listeners to prevent memory leaks
onBeforeUnmount(() => {
  document.removeEventListener('mousemove', updateMousePosition);
  if (map) {
    map.remove();
  }
});
</script>

<style scoped>
/* Import Google Font */
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap');

.map-wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  font-family: 'Poppins', sans-serif;
  overflow: hidden;
  background: white;
}

.map-container {
  width: 100%;
  height: 100%;
  z-index: 1;
  cursor: crosshair;
}

/* --- HOVER CARD --- */
.hover-card {
  position: fixed;
  z-index: 2000;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(5px);
  width: 250px;
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.15);
  border: 1px solid rgba(0,0,0,0.05);
  overflow: hidden;
  pointer-events: auto;
  transition: opacity 0.2s ease, transform 0.2s ease;
  opacity: 0;
  transform: translateY(10px);
}

.hover-card.active {
  opacity: 1;
  transform: translateY(0);
}

.card-img {
  width: 100%;
  height: 140px;
  object-fit: cover;
  display: block;
  background: #ddd;
}

.card-content {
  padding: 15px;
}

.card-title {
  margin: 0 0 10px 0;
  font-weight: 600;
  font-size: 16px;
  color: #333;
}

.card-btn {
  width: 100%;
  padding: 8px;
  background-color: #e74c3c;
  color: white;
  border: none;
  border-radius: 6px;
  font-family: inherit;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.2s;
}

.card-btn:hover {
  background-color: #c0392b;
}

/* --- MODAL --- */
.modal-overlay {
  position: fixed;
  top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0,0,0,0.6);
  z-index: 3000;
  display: flex;
  justify-content: center;
  align-items: center;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.3s ease;
}

.modal-overlay.active {
  opacity: 1;
  pointer-events: auto;
}

.modal-box {
  background: white;
  width: 90%;
  max-width: 500px;
  padding: 30px;
  border-radius: 15px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.3);
  position: relative;
  transform: scale(0.9);
  transition: transform 0.3s ease;
}

.modal-overlay.active .modal-box {
  transform: scale(1);
}

.close-btn {
  position: absolute; top: 15px; right: 20px;
  font-size: 24px; cursor: pointer; color: #888;
}

.close-btn:hover {
  color: #333;
}

.modal-title {
  margin-top: 0;
  color: #e74c3c;
}

.modal-desc {
  line-height: 1.6;
  color: #555;
}

/* --- SPINNER --- */
.loading-spinner {
  position: absolute;
  top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  font-size: 18px;
  color: #555;
  z-index: 2000;
  background: rgba(255,255,255,0.9);
  padding: 20px;
  border-radius: 10px;
}
</style>