<template>
    <div>
        <div style="margin-bottom: 1rem;">
            <label for="minIncome">Min Income: </label>
            <input id="minIncome" type="number" v-model="minIncome" style="width: 60px; margin-right: 1rem;" />

            <label for="regionId">Region ID: </label>
            <input id="regionId" type="text" v-model="regionId" style="width: 120px; margin-right: 1rem;" />

            <button @click="executeFilter">Execute</button>
        </div>

        <div ref="mapContainer" style="width: 100%; height: 500px;"></div>

        <div v-if="showSaveButton">
            <button @click="saveLayer">Save Layer</button>
        </div>
        <div v-if="gridFeatures.length > 0">
            <button @click="exportGridAsGeoJSON">Export Grid as GeoJSON</button>
        </div>

    </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import maplibregl, { Map, Popup } from 'maplibre-gl';
import axios from 'axios';
import * as turf from '@turf/turf';

const mapContainer = ref<HTMLDivElement | null>(null);
let map: Map;

const minIncome = ref<number>(0);
const regionId = ref<string>('');

let familyCardPoints: any[] = [];
let welfarePoints: any[] = [];
let gridFeatures: any[] = [];

const showSaveButton = ref<boolean>(false);

const fetchDataAndPlot = async () => {
    try {
        const cqlFamilyCard = `region_id = '${regionId.value}'`;
        const cqlWelfare = `avg_monthly_income >= ${minIncome.value} AND region_id = '${regionId.value}'`;

        // Mengambil data family_card dan welfare secara bersamaan
        const [familyCardData, welfareData] = await Promise.all([
            axios.get(`https://desa-proaktif.app.barraslogi.id/geoserver/Desa-Proaktif/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=Desa-Proaktif:family_cards&outputFormat=application/json&CQL_FILTER=${cqlFamilyCard}`),
            axios.get(`https://desa-proaktif.app.barraslogi.id/geoserver/Desa-Proaktif/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=Desa-Proaktif:welfares&outputFormat=application/json&CQL_FILTER=${cqlWelfare}`)
        ]);
        console.log('Family Card Data:', familyCardData.data);
        console.log('Welfare Data:', welfareData.data);

        // Memeriksa apakah data ada dan formatnya benar
        if (!familyCardData || !familyCardData.data || !Array.isArray(familyCardData.data.features)) {
            console.error('Invalid family card data', familyCardData);
            alert('Family card data is invalid or empty.');
            return;
        }

        if (!welfareData || !welfareData.data || !Array.isArray(welfareData.data.features)) {
            console.error('Invalid welfare data', welfareData);
            alert('Welfare data is invalid or empty.');
            return;
        }

        // Set the filtered points for Family Cards and Welfares
        familyCardPoints = familyCardData.data.features;
        welfarePoints = welfareData.data.features;

        // Check if the results are empty
        if (familyCardPoints.length === 0 || welfarePoints.length === 0) {
            alert('No data found based on the filters.');
            return;
        }

        // Plot Family Cards and Welfares first
        plotFamilyCardLayer(familyCardPoints);
        plotWelfareLayer(welfarePoints);

        // Proceed to Grid Analysis after plotting
        performGridAnalysis();
    } catch (error) {
        console.error('Error fetching or processing data:', error);
        alert('An error occurred while fetching data. Please check the console for details.');
    }
};

const plotFamilyCardLayer = (points: any[]) => {
    if (map.getSource('familyCardLayer')) {
        map.getSource('familyCardLayer')!.setData({
            type: 'FeatureCollection',
            features: points,
        });
    } else {
        map.addSource('familyCardLayer', {
            type: 'geojson',
            data: {
                type: 'FeatureCollection',
                features: points,
            },
        });

        map.addLayer({
            id: 'familyCardLayer',
            type: 'circle',
            source: 'familyCardLayer',
            paint: {
                'circle-radius': 6,
                'circle-color': '#1f77b4',
                'circle-opacity': 0.7,
            },
        });
    }
};

const plotWelfareLayer = (points: any[]) => {
    if (map.getSource('welfareLayer')) {
        map.getSource('welfareLayer')!.setData({
            type: 'FeatureCollection',
            features: points,
        });
    } else {
        map.addSource('welfareLayer', {
            type: 'geojson',
            data: {
                type: 'FeatureCollection',
                features: points,
            },
        });

        map.addLayer({
            id: 'welfareLayer',
            type: 'circle',
            source: 'welfareLayer',
            paint: {
                'circle-radius': 6,
                'circle-color': '#ff7f0e',
                'circle-opacity': 0.7,
            },
        });
    }
};

const performGridAnalysis = () => {
    // Define the bounding box for the grid based on welfare and family card data
    const bbox = turf.bbox({
        type: 'FeatureCollection',
        features: [...familyCardPoints, ...welfarePoints],
    });

    console.log('Bounding Box:', bbox); // Debugging bounding box

    const grid = turf.squareGrid(bbox, 1, { units: 'kilometers' });

    console.log('Generated Grid:', grid); // Debugging grid features

    // Filter the grid based on points from family cards and welfares
    gridFeatures = grid.features.filter((cell) => {
        const familyCardCount = familyCardPoints.filter((point: any) =>
            turf.booleanPointInPolygon(point.geometry, cell)
        ).length;

        const welfareCount = welfarePoints.filter((point: any) =>
            turf.booleanPointInPolygon(point.geometry, cell)
        ).length;

        if (familyCardCount > 0 && welfareCount > 0) {
            // Add aggregated properties
            cell.properties = {
                family_card_count: familyCardCount,
                welfare_count: welfareCount,
            };
            return true;
        }
        return false;
    });

    console.log('Filtered Grid Features:', gridFeatures); // Debugging filtered grid features

    // Plot the grid analysis results
    updateMapLayer(gridFeatures);
};


const updateMapLayer = (grid: any) => {
    if (map.getSource('grid')) {
        console.log('Updating grid layer...');
        map.getSource('grid')!.setData({
            type: 'FeatureCollection',
            features: grid,
        });
    } else {
        console.log('Adding new grid layer...');
        map.addSource('grid', {
            type: 'geojson',
            data: {
                type: 'FeatureCollection',
                features: grid,
            },
        });

        map.addLayer({
            id: 'grid-layer',
            type: 'fill',
            source: 'grid',
            paint: {
                'fill-color': [
                    'interpolate',
                    ['linear'],
                    ['get', 'family_card_count'],
                    0,
                    '#ffffcc',
                    10,
                    '#ff4444',
                ],
                'fill-opacity': 1,
            },
            layerOptions: {
                'layer-z-index': 10,  // Pastikan ini berada di atas layer lain
            },
        });


        map.on('click', 'grid-layer', (e: any) => {
            const coordinates = e.lngLat;
            const gridFeature = e.features[0];

            const popupContent = `
                <div><strong>Grid Info:</strong><br>Family Card Count: ${gridFeature.properties.family_card_count}</div>
                <div><strong>Welfare Count:</strong> ${gridFeature.properties.welfare_count}</div>
            `;

            new Popup()
                .setLngLat(coordinates)
                .setHTML(popupContent)
                .addTo(map);
        });
    }
};


const executeFilter = async () => {
    showSaveButton.value = true;
    await fetchDataAndPlot();
};

const saveLayer = () => {
    const fileData = new Blob([JSON.stringify(gridFeatures)], {
        type: 'application/json',
    });

    const downloadLink = document.createElement('a');
    downloadLink.href = URL.createObjectURL(fileData);
    downloadLink.download = 'grid_analysis_results.json';
    downloadLink.click();
};

const exportGridAsGeoJSON = () => {
    // Memastikan gridFeatures berisi data
    if (gridFeatures.length > 0) {
        const geojson = {
            type: 'FeatureCollection',
            features: gridFeatures.map((gridFeature) => ({
                type: 'Feature',
                geometry: gridFeature.geometry,
                properties: gridFeature.properties,
            })),
        };

        // Menyimpan GeoJSON ke file atau menampilkan hasil
        const blob = new Blob([JSON.stringify(geojson, null, 2)], { type: 'application/json' });
        const link = document.createElement('a');
        link.href = URL.createObjectURL(blob);
        link.download = 'grid-analysis.geojson';  // Nama file yang akan di-download
        link.click();  // Memulai unduhan
    } else {
        alert('No grid data to export.');
    }
};


onMounted(() => {
    map = new maplibregl.Map({
        container: mapContainer.value! as HTMLElement,
        style: 'https://basemaps.cartocdn.com/gl/positron-nolabels-gl-style/style.json',
        center: [117.0536, -0.6022],
        zoom: 8,
        attributionControl: false,
    });
});
</script>
