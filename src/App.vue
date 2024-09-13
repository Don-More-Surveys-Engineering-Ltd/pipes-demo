<template>
    <div id="map">
        <div
			v-if="!currentlyTrackingUser"
			id="map-crosshair"
		>
			<div id="vert-line" />
			<div id="hor-line" />
			<div id="center-circle" />
		</div>
        <div id="controls">
            <button @click="addPipeVertex">Add vertex</button>
            <button v-if="selectedVertex" @click="moveSelectionToCursor">Move vertex to cursor</button>
        </div>
        <div id="systems">
            Selected system: {{ selectedSystem }}
            <div v-for="system in systems.values()" class="system-li">
                <span>{{ system.id }}</span>
                <span><input v-model="system.name" type="text" @blur="setPipeSources(system.id)"></span>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import maplibregl, { Color } from 'maplibre-gl';
import SelectionIMG from './assets/symbol_selection.png'
import 'maplibre-gl/dist/maplibre-gl.css';
import { computed, onMounted, Ref, ref, watch } from 'vue';

const mapRef = ref<maplibregl.Map>() as Ref<maplibregl.Map>
const currentlyTrackingUser = ref(false)

type PipeVertex = {
    nextIds: number[],
    lnglat: maplibregl.LngLat,
    systemId: number,
    id: number,
}

type System = {
    id: number,
    name: string,
}
let nextVertexId = ref(0)
function popPipeVertexID() {
    return nextVertexId.value++
}
let nextSystemId = ref(0)
function popSystemID() {
    return nextSystemId.value++
}
const systems = ref<Map<number, System>>(new Map())
const selectedVertex = ref<PipeVertex>()
const selectedSystem = computed(() => selectedVertex.value ? systems.value.get(selectedVertex.value.systemId) : undefined)
const pipeVertices = ref<Map<number, PipeVertex>>(new Map())

const LOG_TAG = "[App.vue]" as const;

function onVertexClick(event: maplibregl.MapLayerMouseEvent) {
    const feature = event.features?.[0]
    if (!feature)
        throw new Error("UH OH OH FUCK OH SHIT")
    console.debug(LOG_TAG, `Clicked on vertex ${feature}.`);
    const id = feature.id as number
    selectedVertex.value = pipeVertices.value.get(id)
}

function setPipeSources(systemId: number) {
    (mapRef.value.getSource(`pipe-verticies-${systemId}`) as maplibregl.GeoJSONSource).setData({
        type: 'FeatureCollection',
        features: [...pipeVertices.value.values()].map(vertex => ({
            'type': 'Feature',
            'geometry': {
                type: 'Point',
                coordinates: vertex.lnglat.toArray()
            },
            'properties': {},
            id: vertex.id
        }))
    });

    const lines: GeoJSON.LineString[] = []

    function addLine(from: PipeVertex, to?: PipeVertex) {
        if (!to) {
            for (const nextid of from.nextIds) {
                addLine(from, pipeVertices.value.get(nextid))
            }
            return
        }
        
        console.debug(LOG_TAG, `LINE ${from.id} -> ${to.id}`);
        
        lines.push({
            'type': 'LineString',
            'coordinates': [
                from.lnglat.toArray(),
                to.lnglat.toArray()
            ]
        })
        
        for (const nextid of to.nextIds) {
            addLine(to, pipeVertices.value.get(nextid))
        }
    }

    const lowestIdInSystem = Math.min(
        ...[...pipeVertices.value.values()]
            .filter(v => v.systemId == systemId)
            .map(v => v.id)
    )

    const origin = pipeVertices.value.get(lowestIdInSystem)
    if (origin) {
        addLine(origin, undefined);
        (mapRef.value.getSource(`pipe-lines-${systemId}`) as maplibregl.GeoJSONSource).setData({
            type: 'FeatureCollection',
            features: lines.map(line => ({
                'type': 'Feature',
                'geometry': line,
                'properties': {
                    systemName: systems.value.get(systemId)?.name
                },
            }))
        });
    }
    else
        console.error(LOG_TAG, "No origin point.", pipeVertices.value);
}

function setSelectionSource() {
    (mapRef.value.getSource("selection-point") as maplibregl.GeoJSONSource).setData({
        type: 'FeatureCollection',
        features: selectedVertex.value ? [{
            'type': 'Feature',
            'geometry': {
                type: 'Point',
                coordinates: selectedVertex.value.lnglat.toArray()
            },
            'properties': {},
            id: selectedVertex.value.id
        }] : []
    });
}

function createNewSystem() {
    const systemId = popSystemID()
    const system = {
        id: systemId,
        name: `System ${systemId + 1}`,
    }
    systems.value.set(systemId, system)

        mapRef.value.addSource(`pipe-verticies-${systemId}`, {
        type: "geojson",
        data: {
            type : 'FeatureCollection',
            features: []
        }
    })
    
    mapRef.value.addLayer({
        id: `pipe-verticies-${systemId}`,
        source: `pipe-verticies-${systemId}`,
        type: "circle",
        paint: {
            "circle-color": "#FFFFFF",
            "circle-stroke-color": "#000000",
            "circle-stroke-width": 2,
        },
    })

    mapRef.value.addSource(`pipe-lines-${systemId}`, {
        type: "geojson",
        data: {
            type : 'FeatureCollection',
            features: []
        }
    })
    
    mapRef.value.addLayer({
        id: `pipe-lines-${systemId}`,
        source: `pipe-lines-${systemId}`,
        type: "line",
        paint: {
            "line-color": "#000000",
            "line-width": 2
        },
    })

    mapRef.value.addLayer({
        id: `pipe-text-${systemId}`,
        source: `pipe-lines-${systemId}`,
        type: "symbol",
        layout: {
            "symbol-placement": "line",
            "text-field": ["get", "systemName"]
        },
        paint: {
            "text-color": "#ffffff",
            "text-halo-color": "#000000",
            "text-halo-width": 1,
        }
    })

    
    mapRef.value.on("click", `pipe-verticies-${systemId}`, onVertexClick)

    return system
}


function addPipeVertex() {
    const pos = mapRef.value.getCenter()
    const id = popPipeVertexID()

    console.debug(LOG_TAG, `Add vert ${pos} (${id})`);

    let systemId = selectedVertex.value?.systemId
    if (!selectedVertex.value) {
        // create system
       const newSystem = createNewSystem()
       systemId = newSystem.id
    }

    const vertex: PipeVertex = {
        nextIds: [],
        lnglat: pos,
        systemId: systemId!,
        id,
    }
    
    selectedVertex.value?.nextIds.push(id)
    selectedVertex.value = vertex
    pipeVertices.value.set(id, vertex);
    
    setPipeSources(systemId!)
}

function moveSelectionToCursor() {
    if (!selectedVertex.value)
        return
    const pos = mapRef.value.getCenter()

    console.debug(LOG_TAG, `Move vert to ${pos} (${selectedVertex.value.id})`);
    selectedVertex.value.lnglat = pos

    setPipeSources(selectedVertex.value.systemId)
    setSelectionSource()
}

watch(selectedVertex, () => {
    setSelectionSource()
})

onMounted(async () => {
    mapRef.value= new maplibregl.Map({
        container: 'map', // container id
		center: [0, 0], // starting position [lng, lat]
		zoom: 13, // starting zoom
		hash: true,	
		style: {
			version: 8,
			glyphs: "https://demotiles.maplibre.org/font/{fontstack}/{range}.pbf",
			sources: {},
			layers: []
		},
    });

    await mapRef.value.once("load")

    console.debug(LOG_TAG, "Map loaded.");
    // Icons
    const iconImage = new Image(256, 256);
    iconImage.src = SelectionIMG;
    iconImage.crossOrigin = "Anonymous";
    iconImage.onload = (() => {
        mapRef.value?.addImage?.("Point Selection", iconImage)
    })

    // layers
    mapRef.value.addSource("osm-source", {
		type: 'raster',
		tiles: ["https://tile.openstreetmap.org/{z}/{x}/{y}.png"],
		tileSize: 256,
		attribution: 'Map tiles by <a target="_top" rel="noopener" href="https://tile.openstreetmap.org/">OpenStreetMap tile servers</a>, under the <a target="_top" rel="noopener" href="https://operations.osmfoundation.org/policies/tiles/">tile usage policy</a>. Data by <a target="_top" rel="noopener" href="http://openstreetmap.org">OpenStreetMap</a>'
	})

	mapRef.value.addLayer({
		id: "OSM",
		type: 'raster',
		source: 'osm-source',
		layout: {
			visibility: 'visible'
		}
	})
    
    mapRef.value.addSource("selection-point", {
        type: "geojson",
        data: {
            type : 'FeatureCollection',
            features: []
        }
    })

    mapRef.value.addLayer({
        id: "selection-point",
        source: "selection-point",
        type: 'symbol',
        layout: {
            "icon-image": "Point Selection",
            "icon-overlap": "always",
            "icon-allow-overlap": true,
            "icon-size": .2,
        }
    })

    // controls
    const geolocateControl = new maplibregl.GeolocateControl({
		'trackUserLocation': true
	})
    mapRef.value.addControl(geolocateControl, "top-left")
	geolocateControl.on('trackuserlocationstart', () => currentlyTrackingUser.value = true)
	geolocateControl.on('trackuserlocationend', () => currentlyTrackingUser.value = false)

    // events

    mapRef.value.on("click", (event) => {
        const features = mapRef.value.queryRenderedFeatures(event.point)
        console.debug(LOG_TAG, "Plain click", features);
        
        if (features.length > 0) {
            return 
        }
        selectedVertex.value = undefined
    })
})
</script>

<style scoped lang="scss">

#systems {
    background-color: white;
    color: black;
    padding: 1rem;
    bottom: 0;
    right: 0;
    position: absolute;
    z-index: 1000;

    display: flex;
    flex-direction: column;
    gap: .5rem;

    border: 4px black solid;

    .system-li {
        display: flex;
        gap: .5rem;
        align-items: center;
    }
}

#controls {
    background-color: white;
    padding: 1rem;
    bottom: 0;
    left: 0;
    position: absolute;
    z-index: 1000;

    display: flex;
    flex-direction: row;
    gap: .5rem;

    border: 4px black solid;
}

#map {
    width: 100%;
    height: 100%;
}

#map-crosshair {
	width: 100%;
	height: 100%;
	position: absolute;
	top: 0;
	left: 0;
	z-index: 1000;

	touch-action: none;
	pointer-events: none;

	opacity: .6;

	& > *  {
		margin: auto;
		position: absolute;
		top: 0; left: 0; bottom: 0; right: 0;
	}


	#vert-line {
		height: 50px;
		width: 3px;
		background-color: black;

	} 
	
	#hor-line {
		width: 50px;
		height: 3px;
		background-color: black;
	}

	
	#center-circle {
		width: 35px;
		height: 35px;
		border: 3px black solid;
		border-radius: 100%;
		box-sizing: border-box;

	}
}
</style>
