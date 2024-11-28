<template>
    <div>
        <!-- UI untuk range filter dan tombol Execute -->
        <div style="margin-bottom: 1rem;">
            <label for="minMag">Min Magnitude: </label>
            <input id="minMag" type="number" v-model="minMag" :min="0" :max="10" step="0.1"
                style="width: 60px; margin-right: 1rem;" />

            <label for="maxMag">Max Magnitude: </label>
            <input id="maxMag" type="number" v-model="maxMag" :min="0" :max="10" step="0.1"
                style="width: 60px; margin-right: 1rem;" />

            <button @click="executeFilter">Execute</button>
        </div>

        <!-- Container peta -->
        <div ref="mapContainer" style="width: 100%; height: 500px;"></div>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import maplibregl, { Map, Popup } from 'maplibre-gl';
import axios from 'axios';
import * as turf from '@turf/turf';

// Referensi elemen peta
const mapContainer = ref<HTMLDivElement | null>(null);
let map: Map;

// Variabel untuk filter range magnitude
const minMag = ref<number>(1); // Default nilai minimum
const maxMag = ref<number>(2); // Default nilai maksimum

// Titik gempa yang akan ditampilkan
let earthquakePoints: any[] = [];

// Fungsi utama untuk memperbarui data peta
const updateMapLayer = (grid: any) => {
    // Periksa apakah source dan layer 'grid' sudah ada
    if (map.getSource('grid')) {
        map.getSource('grid')!.setData(grid); // Perbarui data
    } else {
        // Jika belum ada, tambahkan source dan layer baru
        map.addSource('grid', {
            type: 'geojson',
            data: grid,
        });

        map.addLayer({
            id: 'grid-layer',
            type: 'fill',
            source: 'grid',
            paint: {
                'fill-color': [
                    'interpolate',
                    ['linear'],
                    ['get', 'value'],
                    0,
                    '#ffffcc',
                    10,
                    '#ff4444',
                ],
                'fill-opacity': 0.5,
            },
        });

        // Tambahkan interaksi klik untuk menampilkan popup pada grid
        // Fungsi untuk menangani klik pada grid
        map.on('click', 'grid-layer', (e: any) => {
            const coordinates = e.lngLat;
            const value = e.features[0].properties.value;

            // Analisis titik gempa yang ada di dalam grid yang dipilih
            const gridPolygon = e.features[0].geometry;
            const pointsInGrid = earthquakePoints.filter((point: any) =>
                turf.booleanPointInPolygon(point.geometry, gridPolygon)
            );

            // Menjumlahkan magnitudo dari titik gempa yang ada dalam grid
            const totalMagnitude = pointsInGrid.reduce((sum, point: any) => {
                return sum + point.properties.mag; // Menjumlahkan magnitudo
            }, 0);

            const popupContent = `
        <div><strong>Grid Info:</strong><br>Value: ${value}</div>
        <div><strong>Total Magnitude of Earthquakes in this grid:</strong> ${totalMagnitude}</div>
    `;

            // Tampilkan popup dengan jumlah magnitudo
            new Popup()
                .setLngLat(coordinates)
                .setHTML(popupContent)
                .addTo(map);
        });

    }
};

// Fungsi utama untuk mengambil data dan memplot grid dan titik gempa
const fetchDataAndPlot = async () => {
    try {
        const response = await axios.get(
            'https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_week.geojson'
        );
        const earthquakeData = response.data;

        // Filter: hanya gempa dengan magnitude dalam range [minMag, maxMag]
        const filteredFeatures = earthquakeData.features.filter(
            (feature: any) =>
                feature.properties.mag >= minMag.value &&
                feature.properties.mag <= maxMag.value
        );
        console.log('Filtered features:', filteredFeatures);

        if (filteredFeatures.length === 0) {
            console.warn('No data available for the selected magnitude range.');
            alert('No earthquakes found in the selected magnitude range.');
            return;
        }

        const filteredGeoJSON = {
            type: 'FeatureCollection',
            features: filteredFeatures,
        };

        // Titik gempa untuk ditampilkan di peta
        earthquakePoints = filteredFeatures.map((feature: any) => ({
            type: 'Feature',
            geometry: feature.geometry,
            properties: feature.properties,
        }));

        // Plot titik gempa
        if (map.getSource('earthquake-points')) {
            map.getSource('earthquake-points')!.setData({
                type: 'FeatureCollection',
                features: earthquakePoints,
            });
        } else {
            map.addSource('earthquake-points', {
                type: 'geojson',
                data: {
                    type: 'FeatureCollection',
                    features: earthquakePoints,
                },
            });

            map.addLayer({
                id: 'earthquake-points-layer',
                type: 'circle',
                source: 'earthquake-points',
                paint: {
                    'circle-radius': 6,
                    'circle-color': '#ff6666',
                    'circle-opacity': 0.8,
                },
            });
        }

        // Tentukan batas (bounding box) untuk area peta
        const bbox = turf.bbox(filteredGeoJSON);
        const grid = turf.squareGrid(bbox, 100, { units: 'kilometers' });

        // Filter grid hanya untuk yang memenuhi kriteria
        grid.features = grid.features.filter((cell) => {
            const count = filteredFeatures.filter((feature: any) =>
                turf.booleanPointInPolygon(feature.geometry.coordinates, cell)
            ).length;

            cell.properties.value = count; // Tambahkan properti value
            return count > 0; // Hanya grid dengan value > 0
        });

        if (grid.features.length === 0) {
            console.warn('No grid cells matched the criteria.');
            alert('No grid cells matched the criteria.');
            return;
        }

        // Perbarui layer di peta untuk grid
        updateMapLayer(grid);
    } catch (error) {
        console.error('Error fetching or processing data:', error);
    }
};

// Fungsi untuk memperbarui data dan memplot peta
const executeFilter = async () => {
    await fetchDataAndPlot();
};

// Inisialisasi peta
onMounted(() => {
    map = new maplibregl.Map({
        container: mapContainer.value as HTMLElement,
        style: 'https://basemaps.cartocdn.com/gl/positron-nolabels-gl-style/style.json',
        center: [117.25, -2.5], // Fokus pada wilayah Indonesia
        zoom: 4,
    });

    // Pastikan peta siap sebelum memuat data awal
    map.on('load', fetchDataAndPlot);
});
</script>

<style>
/* Gaya tambahan */
button {
    padding: 0.5rem 1rem;
    background-color: #007cbf;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

button:hover {
    background-color: #005f8f;
}
</style>
