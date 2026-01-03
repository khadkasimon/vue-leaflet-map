<template>
  <div id="app">
    <div class="container mt-4">
      <div class="card">
        <div class="card-header">
          <h3 class="mb-0">कार्यालयको GPS लोकेसन व्यवस्थापन</h3>
        </div>
        <div class="card-body">
          <MapComponent
            v-model:mapData="officeMap"
            :tenant-name="tenantName"
            @coordinates-changed="handleCoordinatesChanged"
          />
        </div>
        <div class="card-footer">
          <div class="d-flex justify-content-between align-items-center">
            <div>
              <strong>स्थिति:</strong>
              <span v-if="officeMap" class="text-success ms-2"
                >GPS सेट भएको छ</span
              >
              <span v-else class="text-warning ms-2">GPS सेट गर्नुहोस्</span>
            </div>
            <button class="btn btn-success" @click="saveLocation">
              सेभ गर्नुहोस्
            </button>
          </div>
        </div>
      </div>

      <div class="mt-4">
        <div class="card">
          <div class="card-header">
            <h5 class="mb-0">GeoJSON डाटा</h5>
          </div>
          <div class="card-body">
            <pre v-if="officeMap" class="bg-light p-3 rounded">{{
              formatGeoJson(officeMap)
            }}</pre>
            <p v-else class="text-muted mb-0">
              GPS सेट गरेपछि GeoJSON डाटा देखाइनेछ
            </p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from "vue";
import MapComponent from "./components/MapComponent.vue";

export default {
  name: "App",
  components: {
    MapComponent,
  },
  setup() {
    const tenantName = ref("कार्यालयको नाम");
    const officeMap = ref("");

    // Example initial data (optional)
    officeMap.value = JSON.stringify({
      type: "Feature",
      geometry: {
        type: "Point",
        coordinates: [85.373568, 27.687178],
      },
      properties: {},
    });

    const handleCoordinatesChanged = (coordinates) => {
      console.log("नयाँ GPS निर्देशांक:", coordinates);
    };

    const saveLocation = () => {
      if (!officeMap.value) {
        alert("कृपया पहिले GPS लोकेसन सेट गर्नुहोस्");
        return;
      }

      // Here you would typically save to your backend
      console.log("GPS डाटा सेभ भयो:", officeMap.value);
      alert("GPS लोकेसन सफलतापूर्वक सेभ भयो!");
    };

    const formatGeoJson = (jsonString) => {
      try {
        const parsed = JSON.parse(jsonString);
        return JSON.stringify(parsed, null, 2);
      } catch (e) {
        return jsonString;
      }
    };

    return {
      tenantName,
      officeMap,
      handleCoordinatesChanged,
      saveLocation,
      formatGeoJson,
    };
  },
};
</script>

<style>
#app {
  min-height: 100vh;
  background-color: #f8f9fa;
}

.card {
  border: none;
  box-shadow: 0 2px 15px rgba(0, 0, 0, 0.08);
  border-radius: 10px;
}

.card-header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 10px 10px 0 0 !important;
  padding: 15px 20px;
}

.card-body {
  padding: 20px;
}

.card-footer {
  background-color: #f8f9fa;
  border-top: 1px solid #eee;
  padding: 15px 20px;
  border-radius: 0 0 10px 10px;
}

pre {
  font-size: 12px;
  max-height: 200px;
  overflow-y: auto;
  margin: 0;
}
</style>