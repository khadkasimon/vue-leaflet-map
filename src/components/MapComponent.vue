<template>
  <div class="map-container">
    <div class="header-section">
      <p>GPS लोकेसन सेट गर्नुहोस् ।</p>
      <div class="leaflet-control search">
        <div class="input-group">
          <input
            type="text"
            v-model="coordinatesInput"
            class="form-control"
            placeholder="Latitude, Longitude"
            @keyup.enter="loadCoordinates"
          />
          <button @click="loadCoordinates" class="btn btn-primary">
            लोड गर्नुहोस
          </button>
        </div>
      </div>
    </div>

    <div id="map" ref="mapContainer" style="height: 380px"></div>

    <div class="mt-2">
      <label>{{ tenantName }}को GPS लोकेसन (GeoJSON):</label>
      <input
        type="text"
        v-model="geoJsonData"
        readonly
        class="form-control"
        :name="inputName"
        required
      />
    </div>
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, watch, nextTick } from "vue";

export default {
  name: "MapComponent",
  props: {
    mapData: {
      type: String,
      default: "",
    },
    tenantName: {
      type: String,
      default: "कार्यालय",
    },
    inputName: {
      type: String,
      default: "office_map",
    },
    center: {
      type: Array,
      default: () => [27.687178, 85.373568],
    },
    zoom: {
      type: Number,
      default: 7,
    },
  },
  emits: ["update:mapData", "coordinatesChanged"],

  setup(props, { emit }) {
    const mapContainer = ref(null);
    const coordinatesInput = ref("");
    const geoJsonData = ref(props.mapData || "");

    let map = null;
    let marker = null;
    let drawnItems = null;
    let drawControl = null;

    // Parse GeoJSON data
    const parseGeoJsonData = () => {
      if (!geoJsonData.value) return null;
      try {
        return JSON.parse(geoJsonData.value);
      } catch (error) {
        console.error("Error parsing GeoJSON:", error);
        return null;
      }
    };

    // Initialize map
    const initializeMap = async () => {
      // Dynamically import Leaflet and plugins
      const L = await import("leaflet");
      await import("leaflet-draw");
      await import("leaflet.locatecontrol");

      if (!mapContainer.value) return;

      // Create map with attribution control disabled
      map = L.map(mapContainer.value, { attributionControl: false }).setView(
        props.center,
        props.zoom
      );

      // Add NEB MAP attribution only
      L.control
        .attribution({
          prefix: '<span class="AttributionClass">SURESH MAP</span>',
        })
        .addTo(map);

      // Add OpenStreetMap tile layer WITHOUT attribution
      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution: "", // Empty attribution
      }).addTo(map);

      // Initialize marker
      initializeMarker(L);

      // Initialize draw controls
      initializeDrawControls(L);

      // Add locate control
      L.control.locate().addTo(map);

      // Add layer control with multiple base maps
      setupBaseLayers(L);

      // Update GeoJSON data when map changes
      updateGeoJsonFromMarker(L);
    };

    // Setup base layers
    const setupBaseLayers = (L) => {
      const osm = L.tileLayer(
        "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
        {
          attribution: "", // Empty attribution
        }
      );

      const googleStreets = L.tileLayer(
        "http://{s}.google.com/vt/lyrs=m&x={x}&y={y}&z={z}",
        {
          maxZoom: 20,
          subdomains: ["mt0", "mt1", "mt2", "mt3"],
          attribution: "", // Empty attribution
        }
      );

      const googleSat = L.tileLayer(
        "http://{s}.google.com/vt/lyrs=s&x={x}&y={y}&z={z}",
        {
          maxZoom: 20,
          subdomains: ["mt0", "mt1", "mt2", "mt3"],
          attribution: "", // Empty attribution
        }
      );

      const dark = L.tileLayer(
        "https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png",
        {
          subdomains: "abcd",
          maxZoom: 19,
          attribution: "", // Empty attribution
        }
      );

      // Add default layer
      osm.addTo(map);

      // Create layer control
      const baseMaps = {
        "ओपन स्ट्रिट म्याप": osm,
        "गुगल स्ट्रिट म्याप": googleStreets,
        "गुगल स्याटलाइट म्याप": googleSat,
        "डार्क म्याप": dark,
      };

      L.control.layers(baseMaps, null, { collapsed: false }).addTo(map);
    };

    // Initialize marker
    const initializeMarker = (L) => {
      const geoJson = parseGeoJsonData();

      if (geoJson && geoJson.geometry && geoJson.geometry.coordinates) {
        const coords = geoJson.geometry.coordinates.reverse();
        marker = L.marker(coords, { draggable: true }).addTo(map);

        marker
          .bindPopup(
            `<b>${props.tenantName}</b><br>` +
              `Latitude: ${coords[0]}<br>` +
              `Longitude: ${coords[1]}<br>` +
              `<a href="https://www.google.com/maps/search/?api=1&query=${coords[0]},${coords[1]}" target="_blank">Google Maps मा खोल्नुहोस्</a>`
          )
          .openPopup();

        map.setView(coords, 16);
      } else {
        marker = L.marker(props.center, { draggable: true })
          .addTo(map)
          .bindPopup("<b>कृपया कार्यालयको GPS लोकेसन सेट गर्नुहोस् ।</b>")
          .openPopup();
      }
    };

    // Initialize draw controls
    const initializeDrawControls = (L) => {
      drawnItems = new L.FeatureGroup();
      map.addLayer(drawnItems);

      drawControl = new L.Control.Draw({
        position: "topright",
        draw: {
          marker: true,
          polygon: false,
          polyline: false,
          rectangle: false,
          circle: true,
          circlemarker: false,
          circle: {
            shapeOptions: {
              color: "red",
            },
          },
        },
        edit: {
          featureGroup: drawnItems,
          remove: true,
        },
      });

      map.addControl(drawControl);

      // Handle draw events
      map.on("draw:created", (e) => {
        const layer = e.layer;
        drawnItems.addLayer(layer);
        updateGeoJsonFromLayer(layer, L);
      });

      map.on("draw:edited", (e) => {
        e.layers.eachLayer((layer) => {
          updateGeoJsonFromLayer(layer, L);
        });
      });
    };

    // Update GeoJSON from marker
    const updateGeoJsonFromMarker = (L) => {
      if (marker) {
        marker.on("dragend", () => {
          const geoJson = marker.toGeoJSON();
          geoJsonData.value = JSON.stringify(geoJson);
          emit("update:mapData", geoJsonData.value);
          emit("coordinatesChanged", geoJson.geometry.coordinates);
        });
      }
    };

    // Update GeoJSON from layer
    const updateGeoJsonFromLayer = (layer, L) => {
      const geoJson = layer.toGeoJSON();
      geoJsonData.value = JSON.stringify(geoJson);
      emit("update:mapData", geoJsonData.value);
      emit("coordinatesChanged", geoJson.geometry.coordinates);

      // Remove existing marker and use the drawn one
      if (marker) {
        map.removeLayer(marker);
      }

      if (layer instanceof L.Marker) {
        marker = layer;
        marker.dragging.enable();
        updateGeoJsonFromMarker(L);
      }
    };

    // Load coordinates from input
    const loadCoordinates = async () => {
      const L = await import("leaflet");

      if (!coordinatesInput.value.trim()) {
        map.setView(props.center, props.zoom);
        if (marker) {
          marker.setLatLng(props.center);
          updateGeoJsonFromMarker(L);
        }
        return;
      }

      const coordsArray = coordinatesInput.value
        .split(",")
        .map((coord) => parseFloat(coord.trim()));

      if (
        coordsArray.length === 2 &&
        !isNaN(coordsArray[0]) &&
        !isNaN(coordsArray[1])
      ) {
        const [lat, lng] = coordsArray;

        map.setView([lat, lng], 16);

        if (marker) {
          marker.setLatLng([lat, lng]);

          const geoJson = marker.toGeoJSON();
          geoJsonData.value = JSON.stringify(geoJson);
          emit("update:mapData", geoJsonData.value);
          emit("coordinatesChanged", [lng, lat]);
        }

        coordinatesInput.value = "";
      } else {
        alert("Invalid coordinates format. Please use: Latitude, Longitude");
      }
    };

    // Watch for mapData changes
    watch(
      () => props.mapData,
      (newVal) => {
        if (newVal !== geoJsonData.value) {
          geoJsonData.value = newVal;
          if (map && marker) {
            // Re-initialize marker with new data
            import("leaflet").then((L) => {
              map.removeLayer(marker);
              initializeMarker(L.default);
              updateGeoJsonFromMarker(L.default);
            });
          }
        }
      }
    );

    // Lifecycle hooks
    onMounted(() => {
      nextTick(() => {
        initializeMap();
      });
    });

    onUnmounted(() => {
      if (map) {
        map.remove();
        map = null;
      }
    });

    return {
      mapContainer,
      coordinatesInput,
      geoJsonData,
      loadCoordinates,
    };
  },
};
</script>

<style scoped>
.map-container {
  background: white;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.header-section {
  background: #f8f9fa;
  padding: 15px;
  border-radius: 6px;
  margin-bottom: 15px;
}

.btn-primary {
  white-space: nowrap;
}

#map {
  min-height: 380px;
}

.form-control[readonly] {
  background-color: #f8f9fa;
  font-family: monospace;
  font-size: 14px;
}
</style>