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
            <button :disabled="!selectedVertex || !selectedSystem" @click="onAddVertex">Add vertex</button>
            <button v-if="selectedVertex" @click="onMoveSelectedToCursor">Move vertex to cursor</button>
            <button v-if="selectedVertex" @click="onJoinToSystem">Join to system</button>
            <button @click="dbgPrnt">Dbg</button>
        </div>
        <div id="systems">
            <button @click="onAddSystemWithVertex">New System</button>
            Selected system: {{ selectedSystem?.name }}
            <div v-for="system in systems.values()" class="system-li">
                <span>{{ system.id }}</span>
                <span><input v-model="system.name" type="text" @blur="setPipeSources(system.id)"></span>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import maplibregl from 'maplibre-gl';
import SelectionIMG from './assets/symbol_selection.png'
import 'maplibre-gl/dist/maplibre-gl.css';
import { computed, onMounted, Ref, ref, watch } from 'vue';

function dbgPrnt() {
    console.debug(LOG_TAG, systems.value);
    console.debug(LOG_TAG, pipeVertices.value);
    console.debug(LOG_TAG, pipeSegments.value);
}

const mapRef = ref<maplibregl.Map>() as Ref<maplibregl.Map>
const currentlyTrackingUser = ref(false)

type PipeVertex = {
    lnglat: maplibregl.LngLat,
    systemId: number,
    id: number,
}
type PipeSegment = {
    id: number,
    systemId: number,
    point1Id: number,
    point2Id: number,
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
let nextSegmentId = ref(0)
function popSegmentID() {
    return nextSegmentId.value++
}
const systems = ref<Map<number, System>>(new Map())
const pipeVertices = ref<Map<number, PipeVertex>>(new Map())
const pipeSegments = ref<Map<number, PipeSegment>>(new Map())
const pipeSegmentsOriginMap = computed(() => {
    const map = new Map<number, PipeSegment[]>()
    for (const segment of pipeSegments.value.values()) {
        if (!map.has(segment.point1Id)) {
            map.set(segment.point1Id, [])
        }
        map.get(segment.point1Id)!.push(segment)
    }

    return map
})
const selectedVertex = ref<PipeVertex>()
const selectedSystem = computed(() => selectedVertex.value ? systems.value.get(selectedVertex.value.systemId) : undefined)

const LOG_TAG = "[App.vue]" as const;

function setPipeSources(systemId: number) {
    (mapRef.value.getSource(`pipe-vertices-${systemId}`) as maplibregl.GeoJSONSource).setData({
        type: 'FeatureCollection',
        features: [...pipeVertices.value.values()].map(vertex => ({
            'type': 'Feature',
            'geometry': {
                type: 'Point',
                coordinates: vertex.lnglat.toArray()
            },
            'properties': {
                "terminating": !pipeSegmentsOriginMap.value.get(vertex.id)?.length,
                "id": vertex.id
            },
            id: vertex.id
        }))
    });

    const lines: GeoJSON.LineString[] = []
    const pipeSegemntsInSystem = [...pipeSegments.value.values()].filter(seg => seg.systemId == systemId);

    // Generate line geojson for each line segment in the system.
    for (const segment of pipeSegemntsInSystem) {
        const point1 = pipeVertices.value.get(segment.point1Id)!
        const point2 = pipeVertices.value.get(segment.point2Id)!
        
        console.debug(LOG_TAG, `LINE SEGMENT ${segment.id} ${point1.id} -> ${point2.id}`);
        
        lines.push({
            'type': 'LineString',
            'coordinates': [
                point1.lnglat.toArray(),
                point2.lnglat.toArray()
            ]
        })
    }

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

        mapRef.value.addSource(`pipe-vertices-${systemId}`, {
        type: "geojson",
        data: {
            type : 'FeatureCollection',
            features: []
        }
    })
    
    mapRef.value.addLayer({
        id: `pipe-vertices-${systemId}`,
        source: `pipe-vertices-${systemId}`,
        type: "circle",
        paint: {
            "circle-color": [
                "case",
                ["get", "terminating"],
                "#000000",
                "#ffffff",
            ],
            "circle-stroke-color": "#000000",
            "circle-stroke-width": 2,
        },
    })

    mapRef.value.addLayer({
        id: `pipe-vertex-text-${systemId}`,
        source: `pipe-vertices-${systemId}`,
        type: "symbol",
        paint: {
            "text-color": "red"
        },
        layout: {
            "text-field": ["get", "id"],
            "text-size": 24,
            "text-offset": [0, 2]
        }
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

    
    mapRef.value.on("click", `pipe-vertices-${systemId}`, onVertexClick)

    return system
}

function deleteSystem(systemId: number) {
    mapRef.value.removeLayer(`pipe-text-${systemId}`)
    mapRef.value.removeLayer(`pipe-lines-${systemId}`)
    mapRef.value.removeLayer(`pipe-vertices-${systemId}`)
    mapRef.value.removeLayer(`pipe-vertex-text-${systemId}`)
    mapRef.value.removeSource(`pipe-lines-${systemId}`)
    mapRef.value.removeSource(`pipe-vertices-${systemId}`)
    systems.value.delete(systemId)
}

function createNewPipeSegment(systemId: number, point1Id: number, point2Id: number) {
    const id = popSegmentID()
    const segment: PipeSegment = {
        id,
        systemId,
        point1Id,
        point2Id
    }

    pipeSegments.value.set(id, segment)

    return segment
}

function createNewVertex(systemId: number) {
    const pos = mapRef.value.getCenter()
    const id = popPipeVertexID()

    console.debug(LOG_TAG, `Add vert ${pos} (${id})`);

    const vertex: PipeVertex = {
        lnglat: pos,
        systemId: systemId,
        id,
    }
    
    if (selectedVertex.value && selectedSystem.value)
        createNewPipeSegment(selectedSystem.value.id, selectedVertex.value.id, vertex.id)

    pipeVertices.value.set(id, vertex);
    
    setPipeSources(systemId)

    return vertex
}

/**
 * Create a new vertex at cursor if an origninating 
 * point (and system by extension) is selected
 */
function onAddVertex() {
    // An originating vertex is required
    if (!selectedVertex.value)
        return
    if (!selectedSystem.value)
        return
    createNewVertex(selectedSystem.value.id)
}

/**
 * Create a new system, and add a pipe vertex at the cursor.
 */
function onAddSystemWithVertex() {
    selectedVertex.value = undefined

    const newSystem = createNewSystem()
    const newVertex = createNewVertex(newSystem.id)
    selectedVertex.value = newVertex
}

/**
 * Move selected point to cursor
 */
function onMoveSelectedToCursor() {
    if (!selectedVertex.value)
        return
    const pos = mapRef.value.getCenter()

    console.debug(LOG_TAG, `Move vert to ${pos} (${selectedVertex.value.id})`);
    selectedVertex.value.lnglat = pos

    setPipeSources(selectedVertex.value.systemId)
    setSelectionSource()
}

/**
 * Select vertex when clicked
 */
function onVertexClick(event: maplibregl.MapLayerMouseEvent) {
    const feature = event.features?.[0]
    if (!feature)
        throw new Error("UH OH OH FUCK OH SHIT")
    console.debug(LOG_TAG, `Clicked on vertex.`, feature);
    const id = feature.id as number
    selectedVertex.value = pipeVertices.value.get(id)
}

/**
 * Join point P2 of system S2 to system S1 by replacing point P1 of S1.
 */
async function onJoinToSystem() {
    if (!selectedVertex.value || !selectedSystem.value)
        return
    const S2 = selectedSystem.value
    const P2 = selectedVertex.value
    alert("Tap on a point from another system.")
    
    const data = await new Promise<maplibregl.MapMouseEvent>(
        (resolve) => mapRef.value.once("click", resolve)
    )

    const features = mapRef.value.queryRenderedFeatures(data.point)
    const P1Id = features.find(
        feature => feature.source.includes("pipe-vertices")
    )?.id
    
    if (P1Id == undefined) {
        console.debug(LOG_TAG, "Canclled join.", features);
        return
    }

    const P1 = pipeVertices.value.get(P1Id as number)
    if (!P1) 
        throw new Error(`No vertex with id ${P1Id} found.`)
    const S1 = systems.value.get(P1.systemId)
    if (!S1) 
        throw new Error(`No system with id ${P1.systemId} found.`)
    console.debug(LOG_TAG, `Found point.`, P1, S1);

    if (!!pipeSegmentsOriginMap.value.get(P1.id)?.length) {
        alert("Cannot join to a pipe point that has pipes branching from it.")
        return
    }
    
    // All pipes terminating with P1 need to have their terminator set to P2
    const S1Segments = [...pipeSegments.value.values()].filter(seg => seg.systemId == S1.id)
    for (const segment of S1Segments) {
        if (segment.point1Id == P1.id) {
            segment.point1Id = P2.id
        }
        if (segment.point2Id == P1.id) {
            segment.point2Id = P2.id
        }
        console.debug(LOG_TAG, `Found point.`, P1, S1);
    }
    
    // Set S2 vertices to S1
    const S2Verts = [...pipeVertices.value.values()].filter(vert => vert.systemId == S2.id)
    for (const vert of S2Verts) {
        vert.systemId = S1.id
    }
    const S2Segments = [...pipeSegments.value.values()].filter(seg => seg.systemId == S2.id)
    for (const seg of S2Segments) {
        seg.systemId = S1.id
    }

    console.debug(LOG_TAG, `P1 is`, P1);
    console.debug(LOG_TAG, `S1 is`, S1);
    console.debug(LOG_TAG, `P2 (active) is`, P2);
    console.debug(LOG_TAG, `S2 (active) is`, S2);
    

    // Remove point 1
    pipeVertices.value.delete(P1.id)
    // Remove system 2
    deleteSystem(S2.id)

    // rerender S1
    setPipeSources(S1.id)
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
    top: 0;
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
    width: 100vw;
    height: 100vh;
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
