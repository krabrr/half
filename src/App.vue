<template>
  <div id='map'></div>
  <div id="state-legend" class="legend">
    <h4>Category</h4>
    <div><span style="background-color: #478ba2"></span>ร้านอาหาร</div>
    <div><span style="background-color: #de5b6d"></span>ร้านค้า</div>
    <div><span style="background-color: #b9dcdb"></span>อื่น ๆ</div>
  </div>
</template>

<script>
import json from './assets/geo_json.json'
import mapboxgl from 'mapbox-gl'

export default {
  name: 'App',
  data () {
    return {
      geoJson: json,
      mapStyle: 'mapbox://styles/mapbox/streets-v10',
      accessToken: 'pk.eyJ1Ijoia3JhYnJyIiwiYSI6ImNpb3p3b3k3NzAyZjJ1MG00NXQyN3oxMG4ifQ.6Joj8DEUGjj0hrlgdsojqQ'
    }
  },
  methods: {
    createMap () {
        mapboxgl.accessToken = this.accessToken
        let center = [98.986013, 18.788926]
        this.map = new mapboxgl.Map({
          container: 'map',
          style: this.mapStyle,
          minzoom: 1.3,
          center,
          zoom: 10
        })
        
        this.map.on('load', () => {
          this.map.addSource('shops', {
            'type': 'geojson',
            'data': this.geoJson
          })
          this.map.addLayer({
            'id': 'shops',
            'type': 'circle',
            'source': 'shops',
            'paint': {
              'circle-radius': {
                'base': 6,
                'stops': [[12, 6], [22, 180]]
              },
              'circle-color': [
                'match',
                ['get', 'category'],
                'ร้านอาหาร', '#478ba2',
                'ร้านอาหาร/เครื่องดื่ม', '#478ba2',
                'ร้านค้าธงฟ้า', '#de5b6d',
                'ซุปเปอร์มาร์เก็ต/ขายของใช้/ขายของชำ/มินิมาร์ท/ตลาดสด', '#de5b6d',
                'ร้านค้าท้องถิ่น/ผลิตภัณฑ์ชุมชน/OTOP', '#de5b6d',
                'ร้านค้าทั่วไป', '#de5b6d',
                'แฟชั่น', '#de5b6d',
                'กิจการ OTOP', '#de5b6d',
                /* other */ '#b9dcdb'
              ]
            }
          })
          // this.map.scrollZoom.disable()
          this.map.addControl(new mapboxgl.NavigationControl())
          this.map.on('click', (e) => {
            const features = this.map.queryRenderedFeatures(e.point, { layers: ['shops'] })
            // if the features have no info, return nothing
            if (!features.length) {
              return
            }

            const feature = features[0]
            new mapboxgl.Popup()
              .setLngLat(feature.geometry.coordinates)
              .setHTML('<div id=\'popup\' class=\'popup\' style=\'z-index: 10;\'>' +
              '<table class="table" style="margin-top: 2rem; padding: 1rem;">' +
              '<tbody>' +
              '<tr><td>ชื่อร้าน:</td><td>' + feature.properties.name + '</td></tr>' +
              '<tr><td>ประเภท:</td><td>' + feature.properties.category + '</td></tr>' +
              '<tr><td>ประเภทย่อย:</td><td>' + feature.properties.sub_category + '</td></tr>' +
              '</tbody></table></div>')
              .addTo(this.map)
          })
          this.map.on('mousemove', (e) => {
            let features = this.map.queryRenderedFeatures(e.point, { layers: ['shops'] })
            this.map.getCanvas().style.cursor = features.length ? 'pointer' : ''
          })
          this.map.resize()
        })
      }
  },
  mounted() {
    this.createMap()
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}

.mapboxgl-canvas-container {
    height: 100vh;
}

.legend {
  background-color: #fff;
  border-radius: 3px;
  bottom: 30px;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
  padding: 10px;
  position: absolute;
  right: 10px;
  z-index: 1;
}
 
.legend h4 {
  margin: 0 0 10px;
}
 
.legend div span {
  border-radius: 50%;
  display: inline-block;
  height: 10px;
  margin-right: 5px;
  width: 10px;
}
</style>
