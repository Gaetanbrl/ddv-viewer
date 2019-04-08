<template xmlns:>
  <div id="app" :class="[$options.name]">
    <!-- app map -->
    <vl-map v-if="mapVisible" class="map" ref="map" :load-tiles-while-animating="true" :load-tiles-while-interacting="true"
            @click="clickCoordinate = $event.coordinate" @postcompose="onMapPostCompose"
            data-projection="EPSG:4326" @mounted="onMapMounted">
      <!-- map view aka ol.View -->
      <vl-view ref="view" :center.sync="center" :zoom.sync="zoom" :rotation.sync="rotation"></vl-view>

      <!-- interactions -->
      <vl-interaction-select :features.sync="selectedFeatures" v-if="drawType == null">
        <template slot-scope="select">
          <!-- select styles -->
          <vl-style-box>
            <vl-style-stroke color="#423e9e" :width="7"></vl-style-stroke>
            <vl-style-fill :color="[254, 178, 76, 0.7]"></vl-style-fill>
            <vl-style-circle :radius="5">
              <vl-style-stroke color="#423e9e" :width="7"></vl-style-stroke>
              <vl-style-fill :color="[254, 178, 76, 0.7]"></vl-style-fill>
            </vl-style-circle>
          </vl-style-box>
          <vl-style-box :z-index="1">
            <vl-style-stroke color="#d43f45" :width="2"></vl-style-stroke>
            <vl-style-circle :radius="5">
              <vl-style-stroke color="#d43f45" :width="2"></vl-style-stroke>
            </vl-style-circle>
          </vl-style-box>
          <!--// select styles -->

          <!--// selected popup -->
        </template>
      </vl-interaction-select>
      <!--// interactions -->

      <!-- geolocation -->
      <vl-geoloc @update:position="onUpdatePosition">
        <template slot-scope="geoloc">
          <vl-feature v-if="geoloc.position" id="position-feature">
            <vl-geom-point :coordinates="geoloc.position"></vl-geom-point>
            <vl-style-box>
              <vl-style-icon src="./assets/marker.png" :scale="0.4" :anchor="[0.5, 1]"></vl-style-icon>
            </vl-style-box>
          </vl-feature>
        </template>
      </vl-geoloc>
      <!--// geolocation -->

      <!-- base layers -->
      <vl-layer-tile v-for="layer in baseLayers" :key="layer.name" :id="layer.name" :visible="layer.visible">
        <component :is="'vl-source-' + layer.name" v-bind="layer"></component>
      </vl-layer-tile>
      <!--// base layers -->
      
      <!-- ban result feature-->
      <!--vl-feature id="banMarker" ref="banMarker" >
        <template slot-scope="feature">
          <vl-geom-point :coordinates="banSelectedPosition"></vl-geom-point>
          <vl-style-box>
            <vl-style-icon src="./assets/place.png" :scale="0.5" :anchor="[0.1, 0.95]" ></vl-style-icon>
          </vl-style-box>
        </template>
      </vl-feature-->
      
      <!-- interaction layer -->      
      <vl-layer-vector id="featuresLayer">
        <vl-source-vector ident="featuresSource" :features.sync="featuresBan"></vl-source-vector>        
      </vl-layer-vector>

      <vl-interaction-modify source="featuresSource"></vl-interaction-modify>
      <!-- // interraction layer ->
      <!-- // ban result feature-->

      <!-- other layers from config -->
      <component v-for="layer in layers" :is="layer.cmp" v-if="layer.visible" :key="layer.id" v-bind="layer">
        <!-- add vl-source-* -->
        <component :is="layer.source.cmp" v-bind="layer.source">
          <!-- add static features to vl-source-vector if provided -->
          <vl-feature v-if="layer.source.staticFeatures && layer.source.staticFeatures.length"
                      v-for="feature in layer.source.staticFeatures" :key="feature.id"
                      :id="feature.id" :properties="feature.properties">
            <component :is="geometryTypeToCmpName(feature.geometry.type)" v-bind="feature.geometry"></component>
          </vl-feature>

          <!-- add inner source if provided (like vl-source-vector inside vl-source-cluster) -->
          <component v-if="layer.source.source" :is="layer.source.source.cmp" v-bind="layer.source.source">
            <!-- add static features to vl-source-vector if provided -->
            <vl-feature v-if="layer.source.source.staticFeatures && layer.source.source.staticFeatures.length"
                        v-for="feature in layer.source.source.staticFeatures" :key="feature.id"
                        :id="feature.id" :properties="feature.properties">
              <component :is="geometryTypeToCmpName(feature.geometry.type)" v-bind="feature.geometry"></component>
            </vl-feature>
          </component>
        </component>
        <!--// vl-source-* -->

        <!-- add style components if provided -->
        <!-- create vl-style-box or vl-style-func -->

        <!-- COMPRENDRE CA POUR LES STYLES -->

        <component v-if="layer.style" v-for="(style, i) in layer.style" :key="i" :is="style.cmp" v-bind="style">
          <!-- create inner style components: vl-style-circle, vl-style-icon, vl-style-fill, vl-style-stroke & etc -->
          <component v-if="style.styles" v-for="(st, cmp) in style.styles" :key="cmp" :is="cmp" v-bind="st">
            <!-- vl-style-fill, vl-style-stroke if provided -->
            <vl-style-fill v-if="st.fill" v-bind="st.fill"></vl-style-fill>
            <vl-style-stroke v-if="st.stroke" v-bind="st.stroke"></vl-style-stroke>
          </component>
        </component>
        <!--// style -->
      </component>
      <!--// other layers -->
      <!--// draw components -->
    </vl-map>
    <!--// app map -->

    <!-- map panel, controls -->
    <div class="map-panel">
      <b-collapse class="panel box is-paddingless" :open.sync="panelOpen">
        <div slot="trigger" class="panel-heading">
          <div class="columns">
            <div class="column is-11">
              <strong>Outils</strong>
            </div>
            <div class="column">
              <b-icon :icon="panelOpen ? 'chevron-up' : 'chevron-down'" size="is-small"></b-icon>
            </div>
          </div>
        </div>
        <!--Tools for drop downmap panel-->
        <p class="panel-tabs">          
          <a @click="showMapPanelTab('adresse')" :class="mapPanelTabClasses('adresse')">Adresse</a>          
          <a @click="showMapPanelTab('layers')" :class="mapPanelTabClasses('layers')">Layers</a>                   
        </p>

        <div class="panel-block" v-show="mapPanel.tab === 'adresse'">         
          <section class="sec-adresse">
            <b-field label="Recherche d'adresse:">
              <!--b-autocomplete v-model="name" :data="data" placeholder="15 Rue des Frères Tilly 22700" field="label" :loading="isFetching" @input="getAsyncData" @select="option => selected = option"-->
              <b-autocomplete v-model="name" :data="data" placeholder="15 Rue des Frères Tilly 22700" field="label" :loading="isFetching" @input="getAsyncData" @select="updateCoordinates">
                <template slot-scope="props">
                  {{ props.option.properties.label }}
                </template>
              </b-autocomplete>
            </b-field>
            <a class="button is-danger" @click="removeFeature">Supprimer</a>
          </section>       
        </div>

        <div class="panel-block" v-for="layer in layers" :key="layer.id" @click="showMapPanelLayer"
             :class="{ 'is-active': layer.visible }"
             v-show="mapPanel.tab === 'layers'">
          <b-switch :key="layer.id" v-model="layer.visible">
            {{ layer.title }}
          </b-switch>
        </div>

      </b-collapse>
    </div>
    <!--// map panel, controls -->

    <!-- base layers switch -->
    <div class="base-layers-panel">
      <div class="buttons has-addons">
        <button class="button is-light" v-for="layer in baseLayers"
                :key="layer.name" :class="{ 'is-info': layer.visible }"
                @click="showBaseLayer(layer.name)">
          {{ layer.title }}
        </button>
        <button class="button is-light" @click="mapVisible = !mapVisible">
          {{ mapVisible ? 'Hide map' : 'Show map' }}
        </button>
      </div>
    </div>    
    <!--logo-->
    <div class="logo-org">
      <a href="http://distillerie-vercors.com">
        <img src="http://distillerie-vercors.com/wp-content/uploads/2018/05/Distillerie_logo_brun_corail-e1527681979704.png"></img>
      </a>
    </div>
  </div>
</template>

<script>
  import { kebabCase, range, random, camelCase, debounce } from 'lodash'
  import { createProj, addProj, findPointOnSurface, createStyle, createMultiPointGeom } from 'vuelayers/lib/ol-ext'
  import pacmanFeaturesCollection from './assets/pacman.geojson'
  import ScaleLine from 'ol/control/ScaleLine'
  import FullScreen from 'ol/control/FullScreen'
  import OverviewMap from 'ol/control/OverviewMap'
  import ZoomSlider from 'ol/control/ZoomSlider'
  // -- Add by Gaetan
  import KML from 'ol/format/KML'
  import axios from 'axios'
  import clientsJson from './assets/clients_4326.geojson'
  import faker from 'faker'

  // Custom projection for static Image layer
  let x = 1024 * 10000
  let y = 968 * 10000
  let imageExtent = [-x / 2, -y / 2, x / 2, y / 2]
  let customProj = createProj({
    code: 'xkcd-image',
    units: 'pixels',
    extent: imageExtent,
  })
  // add to the list of known projections
  // after that it can be used by code
  addProj(customProj)

  const methods = {
    removeFeature: function () {
      this.featuresBan = []
    },
    updateCoordinates: function (selected) {
      if (selected.geometry) {
        let geom = {
          type: 'Point',
          coordinates: selected.geometry.coordinates,
        }
        // create new feature if necessary
        this.featuresBan = [{
          id: faker.random.uuid(),
          type: 'Feature',
          geometry: geom,
        }]
      }
    },
    getAsyncData: debounce(function () {
      this.data = []
      this.isFetching = true
      axios.get(`https://api-adresse.data.gouv.fr/search/?q=${this.name}`)
        .then(({
          data,
        }) => {
          data.features.forEach((item) => this.data.push(item))
          this.isFetching = false
        })
        .catch((error) => {
          this.isFetching = false
          throw error
        })
    }, 500),
    camelCase,
    pointOnSurface: findPointOnSurface,
    geometryTypeToCmpName (type) {
      return 'vl-geom-' + kebabCase(type)
    },
    /**
     * Clients layer Style function factory
     * @return {ol.StyleFunction}
     */
    clientsStyleFunc () {
      return function __clientsStyleFunc () {
        return createStyle({
          strokeColor: '#fff',
          fillColor: '#3399cc',
          textFillColor: '#fff',
        })
      }
    },
    /**
     * Packman layer Style function factory
     * @return {ol.StyleFunction}
     */
    pacmanStyleFunc () {
      const pacman = [
        createStyle({
          strokeColor: '#de9147',
          strokeWidth: 3,
          fillColor: [222, 189, 36, 0.8],
        }),
      ]
      const path = [
        createStyle({
          strokeColor: 'blue',
          strokeWidth: 1,
        }),
        createStyle({
          imageRadius: 5,
          imageFillColor: 'orange',
          geom (feature) {
            // geometry is an LineString, convert it to MultiPoint to style vertex
            return createMultiPointGeom(feature.getGeometry().getCoordinates())
          },
        }),
      ]
      const eye = [
        createStyle({
          imageRadius: 6,
          imageFillColor: '#444444',
        }),
      ]

      return function __pacmanStyleFunc (feature) {
        switch (feature.getId()) {
          case 'pacman':
            return pacman
          case 'pacman-path':
            return path
          case 'pacman-eye':
            return eye
        }
      }
    },
    /**
     * Cluster layer style function factory
     * @return {ol.StyleFunction}
     */
    clusterStyleFunc () {
      const cache = {}

      return function __clusterStyleFunc (feature) {
        const size = feature.get('features').length
        let style = cache[size]

        if (!style) {
          style = createStyle({
            imageRadius: 10,
            strokeColor: '#fff',
            fillColor: '#3399cc',
            text: size.toString(),
            textFillColor: '#fff',
          })
          cache[size] = style
        }
        return [style]
      }
    },
    selectFilter (feature) {
      return ['position-feature'].indexOf(feature.getId()) === -1
    },
    onUpdatePosition (coordinate) {
      this.deviceCoordinate = coordinate
    },
    onMapPostCompose ({ vectorContext, frameState }) {
      if (!this.$refs.banMarker || !this.$refs.banMarker.$feature) return
      this.$refs.map.render()
    },
    onMapMounted () {
      // now ol.Map instance is ready and we can work with it directly
      this.$refs.map.$map.getControls().extend([
        new ScaleLine(),
        new FullScreen(),
        new OverviewMap({
          collapsed: false,
          collapsible: true,
        }),
        new ZoomSlider(),
      ])
    },
    // base layers
    showBaseLayer (name) {
      let layer = this.baseLayers.find(layer => layer.visible)
      if (layer != null) {
        layer.visible = false
      }

      layer = this.baseLayers.find(layer => layer.name === name)
      if (layer != null) {
        layer.visible = true
      }
    },
    // map panel
    mapPanelTabClasses (tab) {
      return {
        'is-active': this.mapPanel.tab === tab,
      }
    },
    showMapPanelLayer (layer) {
      layer.visible = !layer.visible
    },
    showMapPanelTab (tab) {
      this.mapPanel.tab = tab
      if (tab !== 'draw') {
        this.drawType = undefined
      }
    },
  }

  export default {
    name: 'vld-demo-app',
    methods,
    data () {
      return {
        // -- specific
        latLong: [0, 0],
        featuresBan: [],
        banSelectedPosition: [0, 0],
        data: [],
        name: '',
        coordinates: undefined,
        selected: null,
        isFetching: false,
        // -- origin
        center: [0, 0],
        zoom: 2,
        rotation: 0,
        clickCoordinate: undefined,
        selectedFeatures: [],
        deviceCoordinate: undefined,
        mapPanel: {
          tab: 'state',
        },
        panelOpen: true,
        mapVisible: true,
        drawType: undefined,
        drawnFeatures: [],
        // base layers
        baseLayers: [
          {
            name: 'osm',
            title: 'OpenStreetMap',
            visible: true,
          },
          {
            name: 'bingmaps',
            title: 'Bing Maps',
            apiKey: 'ArbsA9NX-AZmebC6VyXAnDqjXk6mo2wGCmeYM8EwyDaxKfQhUYyk0jtx6hX5fpMn',
            imagerySet: 'CanvasGray',
            visible: false,
          },
        ],
        // layers config
        layers: [
          {
            id: 'dataKml',
            title: 'Layer KML',
            cmp: 'vl-layer-vector',
            visible: false,
            renderMode: 'image',
            source: {
              cmp: 'vl-source-vector',
              url: './src/assets/clients_4326.kml',
              formatFactory: function () {
                return new KML()
              },
            },
          },
          {
            id: 'clientJson',
            title: 'Client GeoJson',
            cmp: 'vl-layer-vector',
            visible: false,
            source: {
              cmp: 'vl-source-vector',
              staticFeatures: clientsJson.features,
            },
            style: [
              {
                cmp: 'vl-style-box',
                styles: {
                  'vl-style-fill': {
                    color: [255, 255, 255, 0.5],
                  },
                  'vl-style-stroke': {
                    color: '#219e46',
                    width: 2,
                  },
                },
              },
            ],
          },
          {
            id: 'pacman',
            title: 'Pacman',
            cmp: 'vl-layer-vector',
            visible: false,
            renderMode: 'image',
            source: {
              cmp: 'vl-source-vector',
              staticFeatures: pacmanFeaturesCollection.features,
            },
            style: [
              {
                cmp: 'vl-style-func',
                factory: this.pacmanStyleFunc,
              },
            ],
          },
          // Vector layer with clustering
          {
            id: 'cluster',
            title: 'Cluster',
            cmp: 'vl-layer-vector',
            renderMode: 'image',
            visible: false,
            // Cluster source (vl-source-cluster) wraps vector source (vl-source-vector)
            source: {
              cmp: 'vl-source-cluster',
              distance: 50,
              source: {
                cmp: 'vl-source-vector',
                // features defined as array of GeoJSON encoded Features
                // to not overload Vue and DOM
                features: range(0, 10000).map(i => {
                  let coordinate = [
                    random(-50, 50),
                    random(-50, 50),
                  ]

                  return {
                    type: 'Feature',
                    id: 'random-' + i,
                    geometry: {
                      type: 'Point',
                      coordinates: coordinate,
                    },
                  }
                }),
              },
            },
            style: [
              {
                cmp: 'vl-style-func',
                factory: this.clusterStyleFunc,
              },
            ],
          },
        ],
      }
    },
  }
</script>
<style lang="sass">
  @import ~bulma/sass/utilities/_all

  html, body, #app
    width: 100%
    height: 100%
    margin: 0
    padding: 0

  .vld-demo-app
    position: relative

    .map
      height: 100%
      width: 100%

    .map-panel
      padding: 0

      .panel-heading
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-content
        background: $white
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-block
        &.draw-panel
          .buttons
            .button
              display: block
              flex: 1 1 100%

      +tablet()
        position: absolute
        top: 0
        right: 0
        max-height: 500px
        width: 22em

    .base-layers-panel
      position: absolute
      left: 50%
      bottom: 20px
      transform: translateX(-50%)

    .feature-popup
      position: absolute
      left: -50px
      bottom: 12px
      width: 20em
      max-width: none

      &:after, &:before
        top: 100%
        border: solid transparent
        content: ' '
        height: 0
        width: 0
        position: absolute
        pointer-events: none
      &:after
        border-top-color: white
        border-width: 10px
        left: 48px
        margin-left: -10px
      &:before
        border-top-color: #cccccc
        border-width: 11px
        left: 48px
        margin-left: -11px

      .card-content
        max-height: 20em
        overflow: auto

      .content
        word-break: break-all
    .logo-org
      position: fixed
      bottom: 3%
      right: 10px      
    .logo-org img
      max-width: 80%
      float: right
      width: 50%
    .sec-adresse
      width: 100%
</style>
