<script setup>
import "leaflet/dist/leaflet.css";
import {
  LMap,
  LMarker,
  LPolyline,
  LTileLayer,
  LTooltip
} from "@vue-leaflet/vue-leaflet";
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
    // Standard /route
    const trip = data.value.trip;
    return [trip.summary.min_lon, trip.summary.min_lat, trip.summary.max_lon, trip.summary.max_lat]
  } else if (data.value.properties) {
    // WIP: Expansion
    // const features = data.value.features;
    // const firstCoord = features[0].geometry.coordinates[0];
    // let minLon = firstCoord[0], minLat = firstCoord[1], maxLon = firstCoord[0], maxLat = firstCoord[1];
    // features.forEach(function (feature) {
    //   feature.geometry.coordinates.forEach(function (coord) {
    //     minLon = Math.min(minLon, coord[0]);
    //     maxLon = Math.max(maxLon, coord[0]);
    //     minLat = Math.min(minLat, coord[1]);
    //     maxLat = Math.max(maxLat, coord[1]);
    //   });
    // });
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

function parseTrip(trip) {
  let maxCost = 1;

  let result = trip.legs.flatMap(function (leg) {
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

  return result.map(function (maneuver) {
    maneuver.color = getColorForPercentile(maneuver.data.costPerMeter, maxCost)

    return maneuver;
  });
}

function parseExpansion(features) {
  let maxCost = 1;
  let result = features.map(function (feature) {
    feature.properties.costPerSecond = feature.properties.cost / feature.properties.duration
    maxCost = Math.max(maxCost, feature.properties.costPerSecond);
    return {
      "shape": feature.coordinates,
      "data": feature.properties
    }
  })

  return result.map(function (feature) {
    feature.color = getColorForPercentile(feature.data.costPerSecond, maxCost)

    return feature;
  });
}

const segments = computed(function () {
  if (data.value.trip) {
    return parseTrip(data.value.trip);
  } else if (data.value.properties) {
    return parseExpansion(data.value.features);
  }
});

const locations = computed(function () {
  if (data.value.trip) {
    return data.value.trip.locations.map(function (location) {
      return [location.lat, location.lon];
    })
  }
});

const segmentColorOverrides = ref({})
</script>

<template>
  <l-map id="map" :center="[(bbox[1] + bbox[3]) / 2, (bbox[0] + bbox[2]) / 2]"
         :zoom="12" :use-global-leaflet="false">
    <l-tile-layer
        url="https://tiles.stadiamaps.com/tiles/stamen_terrain/{z}/{x}/{y}{r}.png"
        layer-type="base"
        name="Stadia Maps Basemap"
        attribution='&copy; <a href="https://stadiamaps.com/" target="_blank">Stadia Maps</a> &copy; <a href="https://stamen.com/" target="_blank">Stamen Design</a> &copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> &copy; <a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>''
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
    <l-marker v-for="location in locations" :lat-lng="location"></l-marker>
  </l-map>
</template>

<style scoped>
#map {
  /* Something more responsive; hack for now */
  min-height: 750px;
}
</style>
