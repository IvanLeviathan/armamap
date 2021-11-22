<template>
  <div>
    <div style="height: 100vh; width: 100vw;" id="mapContainer"></div>
    <div class="popups">
      <div class="popup" id="player-popup" ref="playerPopup">
        Custom Player popup
      </div>
    </div>
  </div>
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
import LG from 'leaflet-graticule';

import mapObj from '../../../back/public/maps/pulau/pulau.json'

import {onMounted, onBeforeUnmount, ref} from 'vue'
import axios from 'axios'

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
  setup(props,ctx){
    const url = `http://localhost:3000/maps/${mapObj.worldName}/{z}/{x}/{y}.png`,
          playerPopup = ref(null)

    let map = null,
        markersOnMap = {}


    const MGRS_CRS = (factorx, factory, tileWidth) => {
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

    const clearMarkers = () => {
      for(let id in markersOnMap){
        let curMarker = markersOnMap[id];
        curMarker.removeFrom(map);
        delete(markersOnMap[id]);
      }
    }

    const updateMarkers = async () => {
      console.log('update');
      const res = await axios.get('https://discdev.ru/kapayji')
      if(!res.data)
        return;
      let markersData = [];

      res.data.forEach(group => {
        group.forEach((element) => {
          element.marker = false;
          element.bot = false;

          if(!!element.marker_text)
            element.marker = true;

          if(!!element.identify)
            element.bot = true;

          markersData.push(element);
        })
      });


      
      markersData.forEach((curMarker) => {

        let id = curMarker.id;
        let name = curMarker.name;

        if(curMarker.bot){
          id = curMarker.identify
          name = curMarker.identify
        }

        let url = 'http://localhost:3000/images/markers/iconMan_ca.png';
        if(!!curMarker.bot)
          url = 'http://localhost:3000/images/markers/iconMan_ca_bot.png';
        const icon = L.icon({
          iconUrl: url,

          iconSize:     [32, 32], // size of the icon
          iconAnchor:   [32, 32], // point of the icon which will correspond to marker's location
          popupAnchor:  [-15, -40] // point from which the popup should open relative to the iconAnchor
      });

        if(!!markersOnMap[id]){
          const curMarkerOnMap = markersOnMap[id];
          curMarkerOnMap.setLatLng([curMarker.coord_y, curMarker.coord_x]);
          if(curMarker.direction)
              curMarkerOnMap._icon.style.transform = curMarkerOnMap._icon.style.transform + `rotate(${curMarker.direction}deg)`;
        }else{
          const newMarker = L.marker(
            [curMarker.coord_y, curMarker.coord_x],
            {
              icon: icon
            }
          ).addTo(map).bindPopup(playerPopup.value);

          if(!curMarker.marker){
            newMarker._icon.classList.add("player");
            newMarker._icon.classList.add("player-side-" + curMarker.side);
            if(!!curMarker.bot)
              newMarker._icon.classList.add("bot");
            if(curMarker.direction)
              newMarker._icon.style.transform = newMarker._icon.style.transform + `rotate(${curMarker.direction}deg)`;
          }
          markersOnMap[id] = newMarker;
        }

      })

    }

    
    //TODO вынести как то в утилиты или тп
    L.LatLngGraticule = L.Layer.extend({
        includes: L.Evented.prototype,

        options: {
            opacity: 1,
            weight: 0.8,
            color: '#444',
            font: '12px Verdana',
            zoomInterval: [
                { start: 0, end: 5, interval: 1000 }
            ]
        },

        initialize: function (options) {
            L.setOptions(this, options);

            var defaultFontName = 'Verdana';
            var _ff = this.options.font.split(' ');
            if (_ff.length < 2) {
                this.options.font += ' ' + defaultFontName;
            }

            if (!this.options.fontColor) {
                this.options.fontColor = this.options.color;
            }

            if (this.options.zoomInterval) {
                if (this.options.zoomInterval.latitude) {
                    this.options.latInterval = this.options.zoomInterval.latitude;
                    if (!this.options.zoomInterval.longitude) {
                        this.options.lngInterval = this.options.zoomInterval.latitude;
                    }
                }
                if (this.options.zoomInterval.longitude) {
                    this.options.lngInterval = this.options.zoomInterval.longitude;
                    if (!this.options.zoomInterval.latitude) {
                        this.options.latInterval = this.options.zoomInterval.longitude;
                    }
                }
                if (!this.options.latInterval) {
                    this.options.latInterval = this.options.zoomInterval;
                }
                if (!this.options.lngInterval) {
                    this.options.lngInterval = this.options.zoomInterval;
                }
            }
        },

        onAdd: function (map) {
            this._map = map;

            if (!this._container) {
                this._initCanvas();
            }

            map._panes.overlayPane.appendChild(this._container);

            map.on('viewreset', this._reset, this);
            map.on('move', this._reset, this);
            map.on('moveend', this._reset, this);

            this._reset();
        },

        onRemove: function (map) {
            map.getPanes().overlayPane.removeChild(this._container);

            map.off('viewreset', this._reset, this);
            map.off('move', this._reset, this);
            map.off('moveend', this._reset, this);
        },

        addTo: function (map) {
            map.addLayer(this);
            return this;
        },

        setOpacity: function (opacity) {
            this.options.opacity = opacity;
            this._updateOpacity();
            return this;
        },

        bringToFront: function () {
            if (this._canvas) {
                this._map._panes.overlayPane.appendChild(this._canvas);
            }
            return this;
        },

        bringToBack: function () {
            var pane = this._map._panes.overlayPane;
            if (this._canvas) {
                pane.insertBefore(this._canvas, pane.firstChild);
            }
            return this;
        },

        getAttribution: function () {
            return this.options.attribution;
        },

        _initCanvas: function () {
            this._container = L.DomUtil.create('div', 'leaflet-image-layer');

            this._canvas = L.DomUtil.create('canvas', '');

            if (this._map.options.zoomAnimation && L.Browser.any3d) {
                L.DomUtil.addClass(this._canvas, 'leaflet-zoom-animated');
            } else {
                L.DomUtil.addClass(this._canvas, 'leaflet-zoom-hide');
            }

            this._updateOpacity();

            this._container.appendChild(this._canvas);

            L.extend(this._canvas, {
                onselectstart: L.Util.falseFn,
                onmousemove: L.Util.falseFn,
                onload: L.bind(this._onCanvasLoad, this)
            });
        },

        _reset: function () {
            var container = this._container,
                canvas = this._canvas,
                size = this._map.getSize(),
                lt = this._map.containerPointToLayerPoint([0, 0]);

            L.DomUtil.setPosition(container, lt);

            container.style.width = size.x + 'px';
            container.style.height = size.y + 'px';

            canvas.width = size.x;
            canvas.height = size.y;
            canvas.style.width = size.x + 'px';
            canvas.style.height = size.y + 'px';

            this.__calcInterval();

            this.__draw(true);
        },

        _onCanvasLoad: function () {
            this.fire('load');
        },

        _updateOpacity: function () {
            L.DomUtil.setOpacity(this._canvas, this.options.opacity);
        },

        __format_lat: function (lat) {
            var str = "00" + (lat / 1000).toFixed();
            return str.substring(str.length - 2);
        },

        __format_lng: function (lng) {
            var str = "00" + (lng / 1000).toFixed();
            return str.substring(str.length - 2);
        },

        __calcInterval: function () {
            var zoom = this._map.getZoom();
            if (this._currZoom != zoom) {
                this._currLngInterval = 0;
                this._currLatInterval = 0;
                this._currZoom = zoom;
            }

            var interv;

            if (!this._currLngInterval) {
                try {
                    for (var idx in this.options.lngInterval) {
                        var dict = this.options.lngInterval[idx];
                        if (dict.start <= zoom) {
                            if (dict.end && dict.end >= zoom) {
                                this._currLngInterval = dict.interval;
                                break;
                            }
                        }
                    }
                }
                catch (e) {
                    this._currLngInterval = 0;
                }
            }

            if (!this._currLatInterval) {
                try {
                    for (var idx in this.options.latInterval) {
                        var dict = this.options.latInterval[idx];
                        if (dict.start <= zoom) {
                            if (dict.end && dict.end >= zoom) {
                                this._currLatInterval = dict.interval;
                                break;
                            }
                        }
                    }
                }
                catch (e) {
                    this._currLatInterval = 0;
                }
            }
        },

        __draw: function (label) {
            function _parse_px_to_int(txt) {
                if (txt.length > 2) {
                    if (txt.charAt(txt.length - 2) == 'p') {
                        txt = txt.substr(0, txt.length - 2);
                    }
                }
                try {
                    return parseInt(txt, 10);
                }
                catch (e) { }
                return 0;
            };

            var self = this,
                canvas = this._canvas,
                map = this._map

            if (L.Browser.canvas && map) {
                if (!this._currLngInterval || !this._currLatInterval) {
                    this.__calcInterval();
                }

                var latInterval = this._currLatInterval,
                    lngInterval = this._currLngInterval;

                var ctx = canvas.getContext('2d');
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                ctx.lineWidth = this.options.weight;
                ctx.strokeStyle = this.options.color;
                ctx.fillStyle = this.options.fontColor;

                if (this.options.font) {
                    ctx.font = this.options.font;
                }
                var txtWidth = ctx.measureText('0').width;
                var txtHeight = 12;
                try {
                    var _font_size = ctx.font.split(' ')[0];
                    txtHeight = _parse_px_to_int(_font_size);
                }
                catch (e) { }

                var ww = canvas.width,
                    hh = canvas.height;

                var lt = map.containerPointToLatLng(L.point(0, 0));
                var rt = map.containerPointToLatLng(L.point(ww, 0));
                var rb = map.containerPointToLatLng(L.point(ww, hh));

                var _lat_b = rb.lat,
                    _lat_t = lt.lat;
                var _lon_l = lt.lng,
                    _lon_r = rt.lng;

                var _point_per_lat = (_lat_t - _lat_b) / (hh * 0.2);
                if (isNaN(_point_per_lat)) {
                    return;
                }

                if (_point_per_lat < 1) { _point_per_lat = 1; }
                _lat_b = parseInt(_lat_b - _point_per_lat, 10);
                _lat_t = parseInt(_lat_t + _point_per_lat, 10);
                var _point_per_lon = (_lon_r - _lon_l) / (ww * 0.2);
                if (_point_per_lon < 1) { _point_per_lon = 1; }
                _lon_r = parseInt(_lon_r + _point_per_lon, 10);
                _lon_l = parseInt(_lon_l - _point_per_lon, 10);

                var ll, latstr, lngstr;

                function __draw_lat_line(self, lat_tick) {
                    ll = self._latLngToCanvasPoint(L.latLng(lat_tick, _lon_l));
                    latstr = self.__format_lat(lat_tick);
                    txtWidth = ctx.measureText(latstr).width;

                    var __lon_right = _lon_r;
                    var rr = self._latLngToCanvasPoint(L.latLng(lat_tick, __lon_right));

                    /*ctx.beginPath();
                    ctx.moveTo(ll.x + 1, ll.y);
                    ctx.lineTo(rr.x - 1, rr.y);
                    ctx.stroke();*/

                    if (label) {
                        var _yy = ll.y + (txtHeight / 2) - 2;
                        ctx.fillText(latstr, 0, _yy);
                        ctx.fillText(latstr, ww - txtWidth, _yy);
                    }

                };

                if (latInterval > 0) {
                    for (var i = 0; i <= _lat_t; i += latInterval) {
                        if (i >= _lat_b) {
                            __draw_lat_line(this, i);
                        }
                    }
                }

                function __draw_lon_line(self, lon_tick) {
                    lngstr = self.__format_lng(lon_tick);
                    txtWidth = ctx.measureText(lngstr).width;
                    var bb = self._latLngToCanvasPoint(L.latLng(_lat_b, lon_tick));


                    var __lat_top = _lat_t;
                    var tt = self._latLngToCanvasPoint(L.latLng(__lat_top, lon_tick));

                    /*ctx.beginPath();
                    ctx.moveTo(tt.x, tt.y + 1);
                    ctx.lineTo(bb.x, bb.y - 1);
                    ctx.stroke();*/

                    if (label) {
                        ctx.fillText(lngstr, tt.x - (txtWidth / 2), txtHeight + 1);
                        ctx.fillText(lngstr, bb.x - (txtWidth / 2), hh - 3);
                    }

                };

                if (lngInterval > 0) {
                    for (var i = 0; i <= _lon_r; i += lngInterval) {
                        if (i >= _lon_l) {
                            __draw_lon_line(this, i);
                        }
                    }
                }
            }
        },

        _latLngToCanvasPoint: function (latlng) {
            map = this._map;
            var projectedPoint = map.project(L.latLng(latlng));
            projectedPoint._subtract(map.getPixelOrigin());
            return L.point(projectedPoint).add(map._getMapPanePos());
        }

    });

    L.latlngGraticule = function (options) {
        return new L.LatLngGraticule(options);
    };


    L.Control.GridMousePosition = L.Control.extend({
      options: {
        position: 'topright',
      precision: 4
      },

      onAdd: function (map) {
        this._container = L.DomUtil.create('div', 'leaflet-grid-mouseposition');
        L.DomEvent.disableClickPropagation(this._container);
        map.on('mousemove', this._onMouseMove, this);
      var placeHolder = '0'.repeat(this.options.precision);
        this._container.innerHTML = placeHolder + ' - ' + placeHolder;
        return this._container;
      },

      onRemove: function (map) {
        map.off('mousemove', this._onMouseMove)
      },

      _onMouseMove: function (e) {
        this._container.innerHTML = toGrid(e.latlng, this.options.precision);
      }

    });

    L.control.gridMousePosition = function (options) {
        return new L.Control.GridMousePosition(options);
    };
    


    const toGrid = (latlng, precision)  => {
      return toCoord(latlng.lng, precision) + " - " + toCoord(latlng.lat, precision);
    }

    const toCoord = (num, precision) => {
      if (precision === undefined || precision > 5) {
        precision = 4;
      }
      if (num <= 0) {
        return '0'.repeat(precision);
      }
      var numText = "00000" + num.toFixed(0);
      return numText.substr(numText.length - 5, precision);
    }
    // /TODO вынести как то в утилиты или тп

    

    onMounted(() => {
      map = L.map("mapContainer", {
        minZoom: mapObj.minZoom,
        maxZoom: mapObj.maxZoom,
        crs: MGRS_CRS(mapObj.crs[0], mapObj.crs[1], mapObj.crs[2])
      });

      L.tileLayer(url, {
          attribution: mapObj.attribution,
          tileSize: mapObj.tileSize
      }).addTo(map);

      L.latlngGraticule().addTo(map);

      map.setView(mapObj.center, mapObj.defaultZoom);

      L.control.scale({ maxWidth: 200, imperial: false }).addTo(map);

		  L.control.gridMousePosition().addTo(map);

      setInterval(() => {
        updateMarkers();
      }, 500)
      setInterval(() => {
        clearMarkers();
      }, 60000)
    })
    
    onBeforeUnmount(() => {
      if (map) {
        map.remove();
      }
    })
    return {
      playerPopup
    }
  }
};
</script>


<style lang="css">
  body{
    margin: 0;
  }
  .leaflet-marker-pane > * {
    -webkit-transition: transform 1s linear;
    -moz-transition: transform 1s linear;
    -o-transition: transform 1s linear;
    -ms-transition: transform 1s linear;
    transition: transform 1s linear;
    transform-origin: center center;
  }
  .player-side-WEST{
    filter: brightness(89%) contrast(70%) sepia(33%) invert(93%) saturate(2300%) hue-rotate(10deg);
  }
  .player-side-EAST{
    filter: brightness(89%) contrast(70%) sepia(33%) invert(93%) saturate(2300%) hue-rotate(105deg);
  }
  .player-side-GUER{
    filter: brightness(89%) contrast(70%) sepia(33%) invert(93%) saturate(2300%) hue-rotate(-80deg);
  }
  .player-side-CIV{
    filter: brightness(89%) contrast(70%) sepia(33%) invert(93%) saturate(2300%) hue-rotate(55deg);
  }
  .player.bot{

  }
  .leaflet-grid-mouseposition {
    border: 2px solid #777;
    border-top: none;
    line-height: 1.1;
    padding: 2px 5px 1px;
    font-size: 11px;
    white-space: nowrap;
    overflow: hidden;
    box-sizing: border-box;
    background: #fff;
    background: rgba(255, 255, 255, 0.5);
  }
  .popups{
    display: none;
  }
</style>