<template>
    <div>
        <div style="margin-bottom: 1rem;">
            <label for="minMag">Min Magnitude: </label>
            <input id="minMag" type="number" v-model="minMag" :min="0" :max="10" step="0.1"
                style="width: 60px; margin-right: 1rem;" />

            <label for="maxMag">Max Magnitude: </label>
            <input id="maxMag" type="number" v-model="maxMag" :min="0" :max="10" step="0.1"
                style="width: 60px; margin-right: 1rem;" />

            <button @click="executeFilter">Execute</button>
        </div>

        <div ref="mapContainer" style="width: 100%; height: 500px;"></div>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import maplibregl, { Map, Popup } from 'maplibre-gl';
import axios from 'axios';
import * as turf from '@turf/turf';

const mapContainer = ref<HTMLDivElement | null>(null);
let map: Map;

const minMag = ref<number>(1);
const maxMag = ref<number>(2);


let earthquakePoints: any[] = [];

const updateMapLayer = (grid: any) => {
    if (map.getSource('grid')) {
        map.getSource('grid')!.setData(grid);
    } else {
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

        map.on('click', 'grid-layer', (e: any) => {
            const coordinates = e.lngLat;
            const value = e.features[0].properties.value;

            const gridPolygon = e.features[0].geometry;
            const pointsInGrid = earthquakePoints.filter((point: any) =>
                turf.booleanPointInPolygon(point.geometry, gridPolygon)
            );

            const totalMagnitude = pointsInGrid.reduce((sum, point: any) => {
                return sum + point.properties.mag;
            }, 0);

            const popupContent = `
        <div><strong>Grid Info:</strong><br>Value: ${value}</div>
        <div><strong>Total Magnitude of Earthquakes in this grid:</strong> ${totalMagnitude}</div>
    `;

            new Popup()
                .setLngLat(coordinates)
                .setHTML(popupContent)
                .addTo(map);
        });

    }
};

const fetchDataAndPlot = async () => {
    try {
        const response = await axios.get(
            'https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_week.geojson'
        );
        const earthquakeData = response.data;

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
        earthquakePoints = filteredFeatures.map((feature: any) => ({
            type: 'Feature',
            geometry: feature.geometry,
            properties: feature.properties,
        }));

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

        const bbox = turf.bbox(filteredGeoJSON);
        const grid = turf.squareGrid(bbox, 100, { units: 'kilometers' });
        grid.features = grid.features.filter((cell) => {
            const count = filteredFeatures.filter((feature: any) =>
                turf.booleanPointInPolygon(feature.geometry.coordinates, cell)
            ).length;

            cell.properties.value = count;
            return count > 0;
        });

        if (grid.features.length === 0) {
            console.warn('No grid cells matched the criteria.');
            alert('No grid cells matched the criteria.');
            return;
        }

        updateMapLayer(grid);
    } catch (error) {
        console.error('Error fetching or processing data:', error);
    }
};

const executeFilter = async () => {
    await fetchDataAndPlot();
};

onMounted(() => {
    map = new maplibregl.Map({
        container: mapContainer.value as HTMLElement,
        style: 'https://basemaps.cartocdn.com/gl/positron-nolabels-gl-style/style.json',
        center: [117.25, -2.5],
        zoom: 4,
    });

    map.on('load', fetchDataAndPlot);
});
</script>

<style>
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
