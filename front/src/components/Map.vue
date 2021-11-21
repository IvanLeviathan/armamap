<template>
  <div style="height: 100vh; width: 100vw;" id="mapContainer"></div>
</template>
<script>
import {
  LMap,
  LIcon,
  LTileLayer,
  LMarker,
  LControlLayers,
  LTooltip,
  LPopup,
  LPolyline,
  LPolygon,
  LRectangle,
} from "@vue-leaflet/vue-leaflet";
import "leaflet/dist/leaflet.css";

import L from "leaflet";

import mapObj from '../../../back/public/maps/pulau/pulau.json'

export default {
  components: {
    LMap,
    LIcon,
    LTileLayer,
    LMarker,
    LControlLayers,
    LTooltip,
    LPopup,
    LPolyline,
    LPolygon,
    LRectangle,
  },
  data() {
    return {
      zoom: mapObj.defaultZoom,
      iconWidth: 25,
      iconHeight: 40,
      center: mapObj.center,
      maxZoom: mapObj.maxZoom,
      minZoom: mapObj.minZoom,
      tileSize: mapObj.tileSize,
      map: null,
      url: `http://localhost:3000/maps/${mapObj.worldName}/{z}/{x}/{y}.png`,
      markers: {
        1331: {
          x: 2198.49,
          y: 3122.01,
          type: 'player',
          text: 'Игрок 1'
        },
        3333: {
          x: 4000.49,
          y: 3222.01,
          type: 'object',
          text: 'Маркер 2'
        }
      },
      markersOnMap: {}
    };
  },
  computed: {
    iconUrl() {
      return `https://placekitten.com/${this.iconWidth}/${this.iconHeight}`;
    },
    iconSize() {
      return [this.iconWidth, this.iconHeight];
    },
  },
  mounted() {
      this.map = L.map("mapContainer", {
        minZoom: mapObj.minZoom,
        maxZoom: mapObj.maxZoom,
        crs: this.MGRS_CRS(mapObj.crs[0], mapObj.crs[1], mapObj.crs[2])
      });

      L.tileLayer(this.url, {
          attribution: mapObj.attribution,
          tileSize: mapObj.tileSize
      }).addTo(this.map);
      
    this.map.setView(mapObj.center, mapObj.defaultZoom);

    setInterval(() => {
      this.updateMarkers();
    }, 2000)
  },
  beforeDestroy() {
    if (this.map) {
      this.map.remove();
    }
  },
  methods: {
    log(a) {
      // console.log(MGRS_CRS(mapObj.crs));
    },
    updateMarkers() {
      console.log('update');

      var playerIcon = L.icon({
          iconUrl: `https://placekitten.com/${this.iconWidth}/${this.iconHeight}`,

          iconSize:     [38, 95], // size of the icon
          shadowSize:   [50, 64], // size of the shadow
          iconAnchor:   [22, 94], // point of the icon which will correspond to marker's location
          shadowAnchor: [4, 62],  // the same for the shadow
          popupAnchor:  [-3, -76] // point from which the popup should open relative to the iconAnchor
      });

      for(let markerId in this.markers){
        const curMarker = this.markers[markerId];

        if(!!this.markersOnMap[markerId]){
          const curMarkerOnMap = this.markersOnMap[markerId];
          curMarkerOnMap.setLatLng([5000, 3000]);
        }else{
          console.log('here');
          const newMarker = L.marker(
            [curMarker.y, curMarker.x],
            {
              icon: playerIcon
            }
          ).addTo(this.map).bindPopup(curMarker.text);

          this.markersOnMap[markerId] = newMarker;
        }
      }

    },
    MGRS_CRS(factorx, factory, tileWidth) {
      return L.extend({}, L.CRS.Simple, {
          projection: L.Projection.LonLat,
          transformation: new L.Transformation(factorx, 0, -factory, tileWidth),
          scale: function (zoom) {
              return Math.pow(2, zoom);
          },
          zoom: function (scale) {
              return Math.log(scale) / Math.LN2;
          },
          distance: function (latlng1, latlng2) {
              var dx = latlng2.lng - latlng1.lng,
                  dy = latlng2.lat - latlng1.lat;
              return Math.sqrt(dx * dx + dy * dy);
          },
          infinite: true
      });
    }
  },
};
</script>


<style lang="css">
  body{
    margin: 0;
  }
  .leaflet-marker-pane > * {
    -webkit-transition: transform .3s linear;
    -moz-transition: transform .3s linear;
    -o-transition: transform .3s linear;
    -ms-transition: transform .3s linear;
    transition: transform .3s linear;
  }
</style>