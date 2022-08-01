<template>
  <div id="app">
    <div class="row no-gutters" style="margin: 0">
      <div class="col-sm-3">
        <div class="toolbox">
          <div class="sticky-top bg-white shadow-sm p-2">
            <div class="form-group d-flex">
              <label for="cityName" class="mr-2 col-form-label text-right">縣市</label>
              <div class="flex-fill">
                <select id="cityName" class="form-control" v-model="select.city" @change="select.area = ''">
                  <option value="">-- Select One --</option>
                  <option :value="city.CityName" v-for="(city, index) in cityData" :key="index">{{ city.CityName }}</option>
                </select>
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right">地區</label>
              <div class="flex-fill">
                <select id="area" class="form-control" v-if="select.city.length"
                v-model="select.area" @change="updateSelect">
                  <option value="">-- Select One --</option>
                  <option :value="area.AreaName"
                  v-for="area in cityData.find((city) => city.CityName === select.city).AreaList"
                  :key="area.AreaName">
                  {{ area.AreaName }}
                  </option>
                </select>
              </div>
            </div>
            <p class="mb-0 small text-muted text-right">請先選擇區域查看（綠色表示還有口罩）</p>
            </div>
          <ul class="list-group">
            <template v-for="(item, key) in pharmacyData">
              <a class="list-group-item text-left" :key="key"
                v-if="item.properties.county === select.city
                && item.properties.town === select.area"
                :class="{ 'highlight': item.properties.mask_adult || item.properties.mask_child}"
                @click="panTo(item)">
                <h3>{{ item.properties.name }}</h3>
                <p class="mb-0">成人口罩：{{ item.properties.mask_adult}} | 兒童口罩： {{ item.properties.mask_child}}</p>
                <p class="mb-0">地址：<a :href="`https://www.google.com.tw/maps/place/${item.properties.address}`"
                target="_blank" title="Google Map">
                {{ item.properties.address }}</a></p>
              </a>
            </template>
          </ul>
        </div>
        </div>
      <div class="col-sm-9">
        <div id="map"></div>
      </div>
    </div>
  </div>
</template>

<script>
import cityData from '@/assets/CityCountyData.json'
import axios from 'axios'
import L from 'leaflet'

let osmMap = {}

const iconsConfig = {
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41]
}
const icons = {
  green: new L.Icon({
    iconUrl: 'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png',
    ...iconsConfig
  }),
  grey: new L.Icon({
    iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png',
    ...iconsConfig
  })
}

const osm = {
  addMapMarker (x, y, item) {
    const icon = item.mask_adult || item.mask_child ? icons.green : icons.grey
    L.marker([y, x], {
      icon
    }).addTo(osmMap).bindPopup(`<strong>${item.name}</strong> <br>
    口罩剩餘：<strong>成人 - ${item.mask_adult ? `${item.mask_adult} 個` : '未取得資料'}/ 兒童 - ${item.mask_child ? `${item.mask_child} 個` : '未取得資料'}</strong><br>
    地址: <a href="https://www.google.com.tw/maps/place/${item.address}" target="_blank">${item.address}</a><br>
    電話: ${item.phone}<br>
    <small>最後更新時間: ${item.updated}</small>`)
  },
  removeMapMarker () {
    osmMap.eachLayer((layer) => {
      if (layer instanceof L.Marker) {
        osmMap.removeLayer(layer)
      }
    })
  },
  panTo (x, y, item) {
    const icon = item.mask_adult || item.mask_child ? icons.green : icons.grey
    osmMap.panTo(new L.LatLng(y, x))
    L.marker([y, x], {
      icon
    }).addTo(osmMap).bindPopup(`<strong>${item.name}</strong> <br>
    口罩剩餘：<strong>成人 - ${item.mask_adult ? `${item.mask_adult} 個` : '未取得資料'}/ 兒童 - ${item.mask_child ? `${item.mask_child} 個` : '未取得資料'}</strong><br>
    地址: <a href="https://www.google.com.tw/maps/place/${item.address}" target="_blank">${item.address}</a><br>
    電話: ${item.phone}<br>
    <small>最後更新時間: ${item.updated}</small>`).openPopup()
  }
}

export default {
  name: 'App',
  data () {
    return {
      cityData,
      pharmacyData: [],
      select: {
        city: '臺北市',
        area: '大安區'
      }
    }
  },
  methods: {
    getData () {
      axios.get('https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json')
        .then(res => {
          // console.log(res.data)
          this.pharmacyData = res.data.features
          this.updateMarker()
        })
    },
    // 更新地图
    updateMarker () {
      const pharmacies = this.pharmacyData.filter((pharmacy) => {
        if (!this.select.area) {
          return pharmacy.properties.county === this.select.city
        }
        return pharmacy.properties.town === this.select.area
      })

      pharmacies.forEach((pharmacy) => {
        const { properties, geometry } = pharmacy
        osm.addMapMarker(
          geometry.coordinates[0],
          geometry.coordinates[1],
          properties
        )
      })
    },
    // 切換地區清除其他 mark
    removeMarker () {
      osm.removeMapMarker()
    },
    // 改變定位地點
    panTo (pharmacy) {
      const { properties, geometry } = pharmacy
      osm.panTo(geometry.coordinates[0], geometry.coordinates[1], properties)
    },
    updateSelect () {
      this.removeMarker()
      this.updateMarker()
      const pharmacy = this.pharmacyData.find(item => item.properties.town === this.select.area)
      const { geometry, properties } = pharmacy
      osm.panTo(geometry.coordinates[0], geometry.coordinates[1], properties)
    }
  },
  mounted () {
    this.getData()
    // 地图加载...
    osmMap = L.map('map', {
      // 中心目标定位经纬度📌
      center: [25.03, 121.55],
      zoom: 12
    })

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 18,
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
    }).addTo(osmMap)
  }
}
</script>

<style lang="scss">
@import 'bootstrap/scss/bootstrap';

#map {
  height: 100vh;
}

.highlight {
  background: #e9ffe3;
}

.toolbox {
  height: 100vh;
  overflow-y: auto;
  a {
    cursor: pointer;
  }
}
</style>
