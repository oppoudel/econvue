<template>
  <div id="app">
    <div id="query" class="leaflet-bar">
      <label>
        Project Types
        <select v-model="selectedType">
          <option v-for="type in projectTypes">{{type}}</option>
        </select>
      </label>
    </div>
  </div>
</template>

<script>
import L from 'leaflet'
import esri from 'esri-leaflet'
import axios from 'axios'
export default {
  name: 'app',
  data() {
    return {
      selectedType: '',
      econProjects: [],
      projectData: [],
      icons: {},
      map: null,
      projectTypes: ['All'],
      layers: {}
    }
  },
  mounted() {
    this.getSymbols()
    this.createMap()
  },
  methods: {
    createMap() {
      let self = this
      this.map = L.map('app').setView([39.3062, -76.6138], 12)
      const mbAttr = 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, ' +
        '<a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
        'Imagery © <a href="http://mapbox.com">Mapbox</a>'
      const mbUrl = 'https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token=pk.eyJ1Ijoib3Bwb3VkZWwiLCJhIjoiY2l4bHh1Zjk4MDBiNTJ3bGU2d2J4czhibCJ9.fPd6d_roJ2oaXBore0IwfQ';
      L.tileLayer(mbUrl, { id: 'mapbox.streets', attribution: mbAttr }).addTo(this.map);

      let template = '<strong>{Pro_Name}</strong>' + "<Center><img src='{Picture}' alt='{Pro_Name}' height='200' width='200'><Center><br />" +
        '<table class="attrTable">' +
        '<tr valign="top">' +
        '<td class="attrName">Location:</td>' +
        '<td class="attrValue">{Address}</td>' +
        '</tr>' +
        '<tr valign="top">' +
        '<td class="attrName">Project Description:</td>' +
        '<td class="attrValue">{Pro_Desc}</td>' +
        '</tr>' +
        '<tr valign="top">' +
        '<td class="attrName">Project Type/Use:</td>' +
        '<td class="attrValue">{Pro_type}</td>' +
        '</tr>' +
        '</table>'

      const promise = axios.get('https://maps.baltimorecity.gov/egis/rest/services/EconView/Econ_Projects/MapServer/0/query?where=1%3D1&outFields=*&f=geojson')
      promise.then(response => {
        this.econProjects = response.data.features
        this.projectTypes.map((project) => {
          this.layers[project] = L.geoJson(this.econProjects, {
            filter(feature) {
              if (feature.properties.Pro_type === project) {
                return true
              }
            },
            pointToLayer(feature, latlng) {
              let marker = L.marker(latlng, {
                icon: self.icons[feature.properties.Pro_type]
              })
              marker.bindPopup((layer) => (L.Util.template(template, layer.feature.properties)))
              return marker
            }
          })
          for (let prop in this.layers) {
            this.layers[prop].addTo(this.map)
          }
        })
      })
    },
    updateData(value) {
      if (value === 'All') {
        this.projectData = this.econProjects
      }
      this.projectData = this.econProjects.filter(function (item) {
        return item.properties.Pro_type === value
      })
    },
    getSymbols() {
      let econIcon = L.Icon.extend({
        options: {
          iconSize: [20, 24]
        }
      })
      axios.get('https://maps.baltimorecity.gov/egis/rest/services/EconView/Econ_Projects/MapServer/legend?f=pjson')
        .then(response => (response.data.layers[0].legend))
        .then(results => results.forEach((item) => {
          if (item.label === 'Park & Recreation') {
            item.label = 'Park & Open Space'
          }
          this.projectTypes.push(item.label)
          this.icons[item.label] = new econIcon({ iconUrl: 'https://gis.baltimorecity.gov/egis/rest/services/EconView/Econ_Projects/MapServer/0/images/' + item.url })
        }))
    }
  },
  watch: {
    selectedType(value) {
      for (let prop in this.layers) {
        this.map.removeLayer(this.layers[prop])
      }
      if (value === 'All') {
        for (let prop in this.layers) {
          this.map.addLayer(this.layers[prop])
        }
      } else {
        this.map.addLayer(this.layers[value])
        this.map.fitBounds(this.layers[value].getBounds(), {
          padding: [50, 50]
        })
      }
      this.updateData(value)
    }
  }
}
</script>

<style>
@import url('https://unpkg.com/leaflet@1.0.3/dist/leaflet.css');

body {
  margin: 0;
  padding: 0;
}

#app {
  position: absolute;
  top: 0;
  bottom: 0;
  right: 0;
  left: 0;
}

#query {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 1000;
  background: white;
  padding: 1em;
}

#query select {
  font-size: 16px;
}
</style>
