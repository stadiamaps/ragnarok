<script setup>
import "leaflet/dist/leaflet.css";
import {LMap, LPolyline, LTileLayer, LTooltip} from "@vue-leaflet/vue-leaflet";
import {decode} from "@googlemaps/polyline-codec";
import {computed, ref} from "vue";

const props = defineProps(['data'])

const data = computed(function () {
  try {
    return JSON.parse(props.data)
  } catch (e) {
    return {}
  }
});

const bbox = computed(function () {
  if (data.value.trip) {
    const trip = data.value.trip;
    return [trip.summary.min_lon, trip.summary.min_lat, trip.summary.max_lon, trip.summary.max_lat]
  } else {
    return [-171.791110603, 18.91619, -66.96466, 71.3577635769];
  }
});

function getColorForPercentile(value, maxValue) {
  const colors = [
    '#006837', '#1a9850', '#66bd63', '#a6d96a', '#d9ef8b',
    '#fee08b', '#fdae61', '#f46d43', '#d73027', '#a50026'
  ];

  // Calculate the percentile
  const percentile = value / maxValue;

  // Determine the appropriate bucket
  const bucket = Math.min(Math.floor(percentile * colors.length), colors.length - 1);

  return colors[bucket];
}


const segments = computed(function () {
  const trip = data.value.trip;
  let result = [];
  let maxCost = 1;
  if (trip) {
    result = trip.legs.flatMap(function (leg) {
      const line = decode(leg.shape, 6);

      return leg.maneuvers.map(function (maneuver) {
        const {
          instruction,
          begin_shape_index,
          end_shape_index,
          ...rest
        } = maneuver;
        const shape = line.slice(begin_shape_index, end_shape_index + 1);
        rest.costPerMeter = rest.length ? rest.cost / (rest.length * 100) : 0;
        maxCost = Math.max(maxCost, rest.costPerMeter);

        return {
          "title": instruction,
          "shape": shape,
          "data": rest,
        };
      })
    })
  }

  return result.map(function (maneuver) {
    maneuver.color = getColorForPercentile(maneuver.data.costPerMeter, maxCost)

    return maneuver;
  });
});

const segmentColorOverrides = ref({})
</script>

<template>
  <l-map id="map" :center="[(bbox[1] + bbox[3]) / 2, (bbox[0] + bbox[2]) / 2]"
         :zoom="13" :use-global-leaflet="false">
    <l-tile-layer
        url="https://tiles.stadiamaps.com/tiles/outdoors/{z}/{x}/{y}{r}.png"
        layer-type="base"
        name="Stadia Maps Basemap"
        attribution='&copy; <a href="https://stadiamaps.com/">Stadia Maps</a>, &copy; <a href="https://openmaptiles.org/">OpenMapTiles</a> &copy; <a href="https://openstreetmap.org">OpenStreetMap</a> contributors'
    />
    <l-polyline v-for="(segment, idx) in segments" :lat-lngs="segment.shape"
                :weight="7"
                :color="segmentColorOverrides[idx] || segment.color"
                @mouseover="segmentColorOverrides[idx] = 'blue'"
                @mouseout="segmentColorOverrides[idx] = undefined">
      <l-tooltip>
        <div>{{ segment.title }}</div>
        <pre>{{ segment.data }}</pre>
      </l-tooltip>
    </l-polyline>
  </l-map>
</template>

<style scoped>
#map {
  /* Something more responsive; hack for now */
  min-height: 750px;
}
</style>