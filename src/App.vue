<template>
  <div id='map'></div>
  <div class="filter-ctrl">
    <input id="filter-input" type="text" name="filter" placeholder="ค้นหาโดยชื่อ" v-model="keyword" @keyup="search"/>
  </div>
  <div class="province-ctrl">
    <select class="form-control" v-model="selectedProvinceId" @change="filterProvince">
      <option value="">ทุกจังหวัด</option>
      <option v-for="shop in shops" :value="shop.id" :key="shop.id">{{ shop.name }}</option>
    </select>
  </div>
  <div id="state-legend" class="legend">
    <h4>Category</h4>
    <div><span style="background-color: #478ba2"></span>ร้านอาหาร</div>
    <div><span style="background-color: #de5b6d"></span>ร้านค้า</div>
    <div><span style="background-color: #b9dcdb"></span>อื่น ๆ</div>
  </div>
</template>

<script>
import shop20 from './assets/geo_shop_20.json'
import shop50 from './assets/geo_shop_50.json'
import shop83 from './assets/geo_shop_83.json'
import mapboxgl from 'mapbox-gl'

export default {
  name: 'App',
  data () {
    return {
      shops: [
        { id: '20', name: 'ชลบุรี', data: shop20 }, 
        { id: '50', name: 'เชียงใหม่', data: shop50 },
        { id: '83', name: 'ภูเก็ต', data: shop83 }
      ],
      keyword: '',
      selectedProvinceId: '',
      features: [],
      isMapReady: false,
      mapStyle: 'mapbox://styles/mapbox/streets-v10',
      accessToken: 'pk.eyJ1Ijoia3JhYnJyIiwiYSI6ImNpb3p3b3k3NzAyZjJ1MG00NXQyN3oxMG4ifQ.6Joj8DEUGjj0hrlgdsojqQ'
    }
  },
  methods: {
    search () {
      if (!this.isMapReady) {
        return
      }
      const filtered = this.features.filter((feature) => {
        const name = feature.properties.name
        let nameEN = ''
        if (feature.properties.name_en) {
          nameEN = feature.properties.name_en.trim().toLowerCase()
        }
        return name.indexOf(this.keyword) > -1 || nameEN.indexOf(this.keyword) > -1
      })
      for (let i = 0; i < this.shops.length; i++) {
        const shop = this.shops[i]
        const id = shop.id
        if (filtered.length) {
          this.map.setFilter(id, [
            'match',
            ['get', 'name'],
            filtered.map(function (feature) {
              return feature.properties.name
            }),
            true,
            false
          ])
        } else {
          this.map.setFilter(id, ['==', ['get', 'name'], ''])
        }
      }
    },
    filterProvince () {
      if (!this.isMapReady) {
        return
      }
      for (let i = 0; i < this.shops.length; i++) {
        const shop = this.shops[i]
        const id = shop.id
        this.map.setLayoutProperty(id, 'visibility', !this.selectedProvinceId  || id == this.selectedProvinceId ? 'visible' : 'none')
      }
    },
    getUniqueFeatures(features, comparatorProperty) {
      let existingKeys = {}
      // Because features come from tiled vector data, feature geometries may be split
      // or duplicated across tile boundaries and, as a result, features may appear
      // multiple times in query results.
      const uniqueFeatures = features.filter((feature) => {
        if (existingKeys[feature.properties[comparatorProperty]]) {
          return false
        } else {
          existingKeys[feature.properties[comparatorProperty]] = true
          return true
        }
      })
      return uniqueFeatures
    },
    createMap () {
        mapboxgl.accessToken = this.accessToken
        let center = [100.763007, 13.636028]
        this.map = new mapboxgl.Map({
          container: 'map',
          style: this.mapStyle,
          minzoom: 1.3,
          center,
          zoom: 5
        })
        
        this.map.on('load', () => {
          let layers = []
          for (let i = 0; i < this.shops.length; i++) {
            const shop = this.shops[i]
            this.features = [ ...this.features, ...shop.data.features ]
            this.map.addSource(shop.id, {
              type: 'geojson',
              data: shop.data
            })
            this.map.addLayer({
              id: shop.id,
              type: 'circle',
              source: shop.id,
              paint: {
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
            layers.push(shop.id)
          }
          this.features = this.getUniqueFeatures(this.features, 'name')
          this.map.addControl(new mapboxgl.NavigationControl())
          this.map.on('click', (e) => {
            const features = this.map.queryRenderedFeatures(e.point, { layers })
            if (!features.length) {
              return
            }
            const feature = features[0]
            let img = ''
            if (feature.properties.image) {
              img = `<img src='https://search-merchant.xn--42caj4e6bk1f5b1j.com/img/${feature.properties.image}/small' width='100%'>`
            }
            new mapboxgl.Popup()
              .setLngLat(feature.geometry.coordinates)
              .setHTML('<div id=\'popup\' class=\'popup\' style=\'z-index: 10;\'>' +
              img +
              '<table class="table" style="margin-top: 2rem; padding: 1rem;">' +
              '<tbody>' +
              `<tr><td>ชื่อร้าน:</td><td><a href='https://search-merchant.xn--42caj4e6bk1f5b1j.com/shop/${feature.properties.id}' target='_blank'>${feature.properties.name}</a></td></tr>` +
              `<tr><td>ประเภท:</td><td>${feature.properties.category}</td></tr>` +
              `<tr><td>ประเภทย่อย:</td><td>${feature.properties.sub_category}</td></tr>` +
              '</tbody></table></div>')
              .addTo(this.map)
          })
          this.map.on('mousemove', (e) => {
            const features = this.map.queryRenderedFeatures(e.point, { layers })
            this.map.getCanvas().style.cursor = features.length ? 'pointer' : ''
          })
          this.map.resize()
          this.isMapReady = true
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

.filter-ctrl {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
}
  
.filter-ctrl input[type='text'] {
  font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
  width: 100%;
  border: 0;
  background-color: #fff;
  margin: 0;
  color: rgba(0, 0, 0, 0.5);
  padding: 10px;
  box-shadow: 0 0 0 2px rgba(0, 0, 0, 0.1);
  border-radius: 3px;
  width: 180px;
}

.province-ctrl {
  position: absolute;
  top: 60px;
  left: 10px;
  z-index: 1;
  width: 180px;
  font-size: 85%;
}

.province-ctrl select {
  font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
  height: 40px;
}
</style>
