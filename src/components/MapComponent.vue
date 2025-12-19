<template>
  <div class="map-wrapper">
    <!-- Loading -->
    <div v-if="loading" class="loading-spinner">
      {{ statusMessage }}
    </div>

    <!-- Error -->
    <div v-if="error" class="error-toast">
      {{ error }}
      <button @click="loadData">{{ t.retry }}</button>
    </div>

    <!-- Map -->
    <div ref="mapContainer" class="map-container"></div>

    <!-- Language Toggle -->
    <button class="lang-toggle" @click="toggleLanguage" :title="currentLang === 'en' ? 'Switch to Georgian' : 'Switch to English'">
      {{ currentLang === 'en' ? 'ქარ' : 'ENG' }}
    </button>

    <!-- Hover Card -->
    <div 
      class="hover-card" 
      :class="{ active: hover.visible }"
      :style="{ top: hover.y + 'px', left: hover.x + 'px' }"
    >
      <div class="img-wrapper">
        <img 
          :src="hover.picture_url || placeholderImg" 
          @error="handleImgError"
          alt="Region Flag" 
          class="card-img"
        >
        <span v-if="hover.is_capital" class="badge-capital">Capital</span>
        
        <!-- Timer Circle -->
        <div v-if="!isLocked && hover.visible" class="timer-circle">
           <svg width="28" height="28" viewBox="0 0 28 28">
             <circle cx="14" cy="14" r="12" fill="none" stroke="rgba(255,255,255,0.3)" stroke-width="2.5" />
             <circle 
               cx="14" cy="14" r="12" fill="none" 
               stroke="rgba(231, 76, 60, 0.9)" stroke-width="2.5"
               stroke-dasharray="75.4"
               :stroke-dashoffset="75.4 - (75.4 * lockProgress / 100)"
               transform="rotate(-90 14 14)"
             />
           </svg>
        </div>
        <div v-if="isLocked" class="locked-icon">
          <svg width="28" height="28" viewBox="0 0 28 28">
            <circle cx="14" cy="14" r="10" fill="rgba(46, 204, 113, 0.15)" stroke="rgba(46, 204, 113, 0.8)" stroke-width="2"/>
            <path d="M 10 14 L 12.5 16.5 L 18 11" stroke="rgba(46, 204, 113, 0.9)" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </div>
      </div>
      <div class="card-content">
        <h3 class="card-title">{{ hover.name_geo }}</h3>
        <h4 class="card-subtitle">{{ hover.name_eng }}</h4>
        <button class="card-btn" @click.stop="openModal">{{ t.viewDetails }}</button>
      </div>
    </div>

    <!-- Modal -->
    <div 
      class="modal-overlay" 
      :class="{ active: modal.visible }" 
      @click.self="closeModal"
    >
      <div class="modal-box">
        <span class="close-btn" @click="closeModal">&times;</span>

        <div class="modal-header">
          <img 
            :src="modal.data.picture_url || placeholderImg" 
            class="modal-flag"
            @error="handleImgError"
            alt="Flag"
          >
          <div>
            <h2 class="modal-title">{{ modal.data.name_geo }}</h2>
            <span class="modal-subtitle">{{ modal.data.name_eng }}</span>
          </div>
        </div>

        <div class="modal-body">
          <p class="desc-geo" v-if="currentLang === 'ka'"><strong>აღწერა:</strong> {{ modal.data.description_geo || 'ინფორმაცია ბაზაში არ მოიძებნა.' }}</p>
          <p class="desc-eng" v-if="currentLang === 'en'"><strong>Description:</strong> {{ modal.data.description_eng || 'No description available.' }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';
import { supabase } from '@/supabase'; // Adjust path

// CONFIG
const placeholderImg = "https://upload.wikimedia.org/wikipedia/commons/thumb/0/0f/Flag_of_Georgia.svg/320px-Flag_of_Georgia.svg.png";
const TABLE_NAME = 'georgianMap'; 

// LANGUAGE STATE
const currentLang = ref('en'); // 'en' or 'ka'

const translations = {
  en: {
    viewDetails: 'View Details',
    retry: 'Retry',
    noData: 'No Data',
    loading: 'Loading...',
    fetchingData: 'Fetching region data...',
    loadingMap: 'Loading map...',
    noDescription: 'No detailed information found in the database.'
  },
  ka: {
    viewDetails: 'დეტალები',
    retry: 'თავიდან',
    noData: 'მონაცემი არ არის',
    loading: 'იტვირთება...',
    fetchingData: 'რეგიონის მონაცემების ჩატვირთვა...',
    loadingMap: 'რუკის ჩატვირთვა...',
    noDescription: 'დეტალური ინფორმაცია ბაზაში არ მოიძებნა.'
  }
};

const t = computed(() => translations[currentLang.value]);

const toggleLanguage = () => {
  currentLang.value = currentLang.value === 'en' ? 'ka' : 'en';
}; 

// STATE
const mapContainer = ref(null);
const loading = ref(true);
const error = ref(null);
const statusMessage = ref("Initializing...");
let map = null;
let geojsonLayer = null;

const regionLookup = {};

const hover = reactive({
  visible: false,
  x: 0,
  y: 0,
  name_eng: '',
  name_geo: '',
  picture_url: '',
  is_capital: false,
  rawData: null
});

const modal = reactive({
  visible: false,
  data: {}
});

// HELPERS
const normalizeName = name => name ? name.replace(/Municipality|City|Munitcipality/gi, "").trim() : "";

// HOVER LOCK STATE
// HOVER LOCK STATE
const isLocked = ref(false);
const lockProgress = ref(0);
let lockFrame = null;
let lockStartTime = 0;
const LOCK_DURATION = 1500; // ms

// DATA FETCHING
const loadData = async () => {
  loading.value = true;
  error.value = null;
  statusMessage.value = t.value.fetchingData;

  try {
    const { data: dbData, error: dbError } = await supabase.from(TABLE_NAME).select('*');
    if (dbError) throw dbError;
    console.log("Supabase Rows Loaded:", dbData?.length);
    if (dbData?.length > 0) console.log("First Row:", dbData[0]);

    dbData.forEach(row => {
      regionLookup[normalizeName(row.name_eng)] = row;
      regionLookup[row.name_eng] = row;
    });

    statusMessage.value = t.value.loadingMap;
    await initMap();
  } catch (err) {

    console.error(err);
    error.value = "Failed to load map data.";
    loading.value = false;
  }
};

// MAP
const initMap = async () => {
  if (map) return;

  // Georgia Bounds
  const southWest = L.latLng(40.8, 38.5);
  const northEast = L.latLng(43.8, 47.0);
  const bounds = L.latLngBounds(southWest, northEast);

  map = L.map(mapContainer.value, {
    center: [42.1, 43.5],
    zoom: 7.5,
    minZoom: 7,
    maxZoom: 10,
    maxBounds: bounds,
    tap: true,  // Enable tap for mobile
    tapTolerance: 15,  // Increase tap tolerance for touch
    maxBoundsViscosity: 1.0,
    zoomControl: false,
    attributionControl: false,
    scrollWheelZoom: 'center',
    dragging: !L.Browser.mobile ? true : L.Browser.mobile  // Better mobile dragging
  });

  // Close card when clicking map background
  map.on('click', () => {
    isLocked.value = false;
    hover.visible = false;
    if (geojsonLayer) geojsonLayer.resetStyle();
  });

  try {
    const response = await fetch("https://raw.githubusercontent.com/bumbeishvili/geojson-georgian-regions/master/SubRegions_low_quality.json");
    if (!response.ok) throw new Error("GeoJSON not found");
    const geoData = await response.json();

    loading.value = false;

    geojsonLayer = L.geoJson(geoData, {
      // 2) Modern Look: Give polygons a subtle fill so they stand out
      style: () => ({ 
        fillColor: '#2c3e50', 
        weight: 1.5, 
        color: '#ffffff', 
        opacity: 0.6,
        fillOpacity: 0.05 
      }),
      onEachFeature: (feature, layer) => {
        layer.on({
          mouseover: e => highlightFeature(e, feature),
          mouseout: resetHighlight,
          click: L.DomEvent.stopPropagation
        });
      }
    }).addTo(map);

    map.fitBounds(geojsonLayer.getBounds());
  } catch (err) {
    console.error(err);
    error.value = "Failed to load map geometry.";
    loading.value = false;
  }
};

// HOVER LOGIC
const highlightFeature = (e, feature) => {
  // 1) Clean up any previous highlights (fixes "stuck" areas)
  if (geojsonLayer) geojsonLayer.resetStyle();

  // 2) Reset Lock
  cancelAnimationFrame(lockFrame);
  isLocked.value = false;
  lockProgress.value = 0;
  lockStartTime = performance.now();

  const animateLock = (time) => {
    // If we mouse out or it's improperly visible, stop
    if (!hover.visible) return; 

    const elapsed = time - lockStartTime;
    const progress = Math.min((elapsed / LOCK_DURATION) * 100, 100);
    lockProgress.value = progress;

    if (progress < 100) {
      lockFrame = requestAnimationFrame(animateLock);
    } else {
      isLocked.value = true; // Lock engaged
    }
  };
  
  lockFrame = requestAnimationFrame(animateLock);

  const layer = e.target;
  layer.setStyle({ weight: 3, color: '#e74c3c', fillOpacity: 0.1, fillColor: '#e74c3c' });
  layer.bringToFront();

  // ... rest of data fetching
  const geoName = feature.properties.NAME_2 || feature.properties.name || "";
  const dbRecord = regionLookup[normalizeName(geoName)];

  if (dbRecord) {
    Object.assign(hover, { 
      name_eng: dbRecord.name_eng, 
      name_geo: dbRecord.name_geo, 
      picture_url: dbRecord.picture_url, 
      is_capital: dbRecord.is_capital,
      rawData: dbRecord
    });
  } else {
    Object.assign(hover, { 
      name_eng: geoName, 
      name_geo: t.value.noData, 
      picture_url: null, 
      is_capital: false,
      rawData: null
    });
  }

  hover.visible = true;
};

const resetHighlight = e => {
  if (isLocked.value) return; // Verify lock
  cancelAnimationFrame(lockFrame);
  lockProgress.value = 0;
  // Note: we don't resetStyle here for EVERYTHING, just target, to be efficient.
  // But highlightFeature handles the global cleanup now.
  geojsonLayer.resetStyle(e.target);
  hover.visible = false;
};

const updateMousePosition = e => {
  if (!hover.visible || isLocked.value) return;

  const cardW = 260, cardH = 250;
  let x = e.clientX + 15, y = e.clientY + 15;

  if (x + cardW > window.innerWidth) x -= cardW + 30;
  if (y + cardH > window.innerHeight) y -= cardH + 30;

  hover.x = x;
  hover.y = y;
};

// IMAGE
const handleImgError = e => e.target.src = placeholderImg;

// MODAL
const openModal = () => {
  console.log("Open Modal Clicked. Data:", hover.rawData);
  
  // Always open modal (fallback if no data)
  modal.data = hover.rawData || {
    name_eng: hover.name_eng || "Unknown Region",
    name_geo: hover.name_geo || "...",
    description_eng: t.value.noDescription,
    description_geo: "ინფორმაცია ბაზაში არ მოიძებნა.",
    picture_url: null
  };

  hover.visible = false;
  isLocked.value = false;
  modal.visible = true;
};

const closeModal = () => modal.visible = false;

// LIFECYCLE
onMounted(() => {
  document.addEventListener('mousemove', updateMousePosition);
  loadData();
});

onBeforeUnmount(() => {
  document.removeEventListener('mousemove', updateMousePosition);
  if (map) map.remove();
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;600&display=swap');

.map-wrapper { 
  position: relative; 
  width: 100vw; height: 100vh; 
  font-family: 'Poppins', sans-serif; 
  /* 3) Nice Background: Subtle gradient */
  background: linear-gradient(135deg, #fdfbfb 0%, #ebedee 100%); 
  overflow: hidden; 
}
.map-container { width: 100%; height: 100%; z-index: 1; cursor: crosshair; }

/* Language Toggle Button */
.lang-toggle {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 2500;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(0, 0, 0, 0.1);
  padding: 10px 18px;
  border-radius: 12px;
  font-weight: 600;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  color: #2c3e50;
}
.lang-toggle:hover {
  background: rgba(255, 255, 255, 1);
  transform: translateY(-2px);
  box-shadow: 0 6px 16px rgba(0, 0, 0, 0.15);
}

/* Loading & Error */
.loading-spinner { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); padding: 15px 25px; border-radius: 30px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); background: white; color: #555; font-weight: 600; }
.error-toast { position: absolute; top: 20px; left: 50%; transform: translateX(-50%); background: #ff4d4d; color: white; padding: 10px 20px; border-radius: 8px; z-index: 5000; display: flex; gap: 10px; align-items: center; }
.error-toast button { background: white; color: #ff4d4d; border: none; padding: 5px 10px; border-radius: 4px; cursor: pointer; }

/* Hover Card */
/* 2) Modern Look: Glassmorphism */
.hover-card { 
  position: fixed; z-index: 2000; width: 280px; 
  border-radius: 16px; overflow: hidden; 
  opacity: 0; transition: opacity 0.2s cubic-bezier(0.25, 0.8, 0.25, 1);
  background: rgba(255, 255, 255, 0.85); 
  backdrop-filter: blur(12px) saturate(180%);
  -webkit-backdrop-filter: blur(12px) saturate(180%);
  box-shadow: 0 12px 30px rgba(0,0,0,0.1), 0 1px 3px rgba(0,0,0,0.05);
  border: 1px solid rgba(255,255,255,0.3);
  pointer-events: auto; 
}
.hover-card.active { opacity: 1; }
.img-wrapper { position: relative; width: 100%; height: 140px; background: #f0f0f0; }
.card-img { width: 100%; height: 100%; object-fit: cover; }
.badge-capital { position: absolute; top: 10px; right: 10px; background: #e74c3c; color: white; font-size: 10px; padding: 4px 8px; border-radius: 20px; text-transform: uppercase; font-weight: bold; box-shadow: 0 2px 5px rgba(0,0,0,0.2); }
.timer-circle { position: absolute; top: 8px; left: 8px; pointer-events: none; }
.locked-icon { position: absolute; top: 8px; left: 8px; pointer-events: none; display: flex; align-items: center; justify-content: center; }
.card-content { padding: 15px; }
.card-title { margin: 0; font-weight: 600; font-size: 16px; color: #2c3e50; }
.card-subtitle { margin: 0 0 10px 0; font-weight: 400; font-size: 13px; color: #7f8c8d; }
.card-btn { 
  width: 100%; padding: 10px; 
  background: linear-gradient(135deg, #ff6b6b 0%, #ee5253 100%); 
  color: white; border: none; border-radius: 10px; 
  font-weight: 600; letter-spacing: 0.5px;
  cursor: pointer; transition: transform 0.1s, box-shadow 0.2s; 
  box-shadow: 0 4px 6px rgba(238, 82, 83, 0.2);
}
.card-btn:hover { transform: translateY(-1px); box-shadow: 0 6px 12px rgba(238, 82, 83, 0.3); }

/* Modal */
.modal-overlay { 
  position: fixed; top: 0; left: 0; width: 100%; height: 100%; 
  background: rgba(44, 62, 80, 0.3); /* Darker, modern overlay */
  backdrop-filter: blur(4px);
  z-index: 3000; display: flex; justify-content: center; align-items: center; 
  opacity: 0; pointer-events: none; transition: opacity 0.3s ease; 
}
.modal-box { 
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(20px);
  width: 90%; max-width: 600px; 
  padding: 40px; border-radius: 24px; 
  box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25); 
  border: 1px solid rgba(255,255,255,0.5);
  position: relative; transform: translateY(20px); transition: transform 0.3s ease; 
}
.modal-overlay.active { opacity: 1; pointer-events: auto; }
.modal-overlay.active .modal-box { transform: translateY(0); }
.close-btn { position: absolute; top: 15px; right: 20px; font-size: 28px; cursor: pointer; color: #999; line-height: 1; }
.modal-header { display: flex; align-items: center; gap: 20px; margin-bottom: 20px; border-bottom: 1px solid #eee; padding-bottom: 15px; }
.modal-flag { width: 80px; height: 50px; object-fit: cover; border-radius: 6px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.modal-title { margin: 0; color: #e74c3c; font-size: 24px; }
.modal-subtitle { color: #888; font-size: 16px; }
.desc-eng { margin-bottom: 10px; color: #444; line-height: 1.6; }
.desc-geo { color: #666; font-style: italic; line-height: 1.6; }

/* Mobile Responsive Styles */
@media (max-width: 768px) {
  .hover-card {
    width: calc(100vw - 40px);
    max-width: 320px;
    left: 50% !important;
    transform: translateX(-50%);
    bottom: 20px;
    top: auto !important;
  }
  
  .timer-circle, .locked-icon {
    width: 32px;
    height: 32px;
  }
  
  .timer-circle svg {
    width: 32px;
    height: 32px;
  }
  
  .card-title { font-size: 15px; }
  .card-subtitle { font-size: 12px; }
  .card-btn { padding: 12px; font-size: 14px; }
  
  .modal-box {
    width: 95%;
    padding: 25px;
    max-height: 90vh;
    overflow-y: auto;
  }
  
  .modal-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 15px;
  }
  
  .modal-flag {
    width: 100%;
    height: auto;
    max-height: 120px;
  }
  
  .modal-title { font-size: 20px; }
  .modal-subtitle { font-size: 14px; }
  
  .close-btn {
    font-size: 32px;
    top: 10px;
    right: 15px;
  }
  
  .loading-spinner {
    padding: 12px 20px;
    font-size: 14px;
  }
}

@media (max-width: 480px) {
  .hover-card {
    width: calc(100vw - 30px);
  }
  
  .modal-box {
    padding: 20px;
    border-radius: 16px;
  }
  
  .card-title { font-size: 14px; }
  .modal-title { font-size: 18px; }
}
</style>
