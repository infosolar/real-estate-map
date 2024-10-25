<script setup lang="ts">
import ObjectsFilters from "@/components/filters/ObjectsFilters.vue"
import MarkerPopup from "@/components/map/parts/MarkerPopup.vue"
import MapLegend from "@/components/map/parts/MapLegend.vue"
import AppButton from "@/components/common/AppButton.vue"
import {config, LngLatBounds, Map, Popup} from '@maptiler/sdk'

import * as turf from '@turf/turf'
import {onMounted, onUnmounted, ref, shallowRef, markRaw, watch, createVNode, render} from 'vue'
import {type LocationQueryValue, useRoute, onBeforeRouteLeave} from "vue-router"
import {useCommonStore, useUserStore, useBuildingsStore} from "@/stores"
import {useI18n} from "vue-i18n"
import Tr from "@/i18n/translation"
import {getMapStyleUrl, loadImageAsync, updateLanguage} from "@/composables"
import type {IObjectAddress} from '@/types/interfaces/objects'

const route = useRoute()
const {t} = useI18n()
const userStore = useUserStore()
const commonStore = useCommonStore()
const buildingsStore = useBuildingsStore()

const currentRegion = ref<LocationQueryValue | LocationQueryValue[]>(route.query.region)
const currentAddress = ref<LocationQueryValue | LocationQueryValue[]>(route.query.name)
const map = shallowRef<Map | null>(null)
const mapContainer = shallowRef<HTMLDivElement>()
const isRulerActive = ref<boolean>(false)
const initialState = {
  lng: 30.5238,
  lat: 50.4501,
  zoom: 5
}
let popup: Popup | null = null
let rulerBtn: HTMLButtonElement | null = null

// GeoJSON object to hold our measurement features
interface IGeoJson {
  type: string
  features: any[]
}

const geojson: IGeoJson = {
  'type': 'FeatureCollection',
  'features': []
}

// Used to draw a line between points
interface ILinestring {
  type: string
  geometry: {
    type: string
    coordinates: number[]
  }
}

const linestring: ILinestring = {
  'type': 'Feature',
  'geometry': {
    'type': 'LineString',
    'coordinates': []
  }
}

function addDataToMap(initialLoad: boolean = false) {
  // group markers in clusters
  if (map.value) {
    const markersSource = map.value.getSource('markers')

    if (markersSource) return

    map.value.addSource('markers', {
      type: 'geojson',
      data: {
        type: 'FeatureCollection',
        features: buildingsStore.addresses.map((marker: IObjectAddress) => ({
          type: 'Feature',
          properties: {
            // add info about address for modal
            inFavorites: userStore.isObjectInFavorites(marker.id),
            media: marker.image,
            name: marker.name,
            region: marker.region,
            area: marker.area,
            estateStatus: marker.estate_status,
            fillPercent: marker.fill_percent ? Math.round(marker.fill_percent) : 0,
            id: marker.id,
            objectStatus: marker.object_status ? marker.object_status : null,
            price: marker.price ? Math.round(marker.price) : 0
          },
          geometry: {
            type: 'Point',
            coordinates: [
              marker.coordinates[1],
              marker.coordinates[0]
            ],
          },
        })),
      },
      cluster: true,
      clusterMaxZoom: 14,
      clusterRadius: 50
    })

    // Fit all markers to screen size
    fitMarkersToScreen(buildingsStore.addresses)

    map.value.loadImage('/images/map-images/icon-group.png', (error, image) => {
      if (error) throw error

      if (initialLoad) {
        if (map.value && image) map.value.addImage('icon-group', image)
      }

      if (map.value) {
        map.value.addLayer({
          id: 'clusters',
          type: 'symbol',
          source: 'markers',
          filter: ['has', 'point_count'],
          layout: {
            'icon-image': 'icon-group',
            'icon-size': 0.9,
            'icon-allow-overlap': true,
            'symbol-sort-key': ['get', 'point_count_abbreviated'],
            'icon-offset': [0, 0],

            'text-field': '{point_count_abbreviated}',
            'text-font': ['Metropolis Semi Bold'],
            'text-size': 12,
            'text-offset': [0, 1.8],
            'text-justify': 'center',
            'text-allow-overlap': true
          },
          'paint': {
            'text-color': '#FFFFFF',
            'text-halo-color': '#3a76bf',
            'text-halo-width': 2
          }
        })
      }
    })

    // set single marker view
    Promise.all([
      loadImageAsync(map.value, '/images/map-images/icon-red-empty.png'),
      loadImageAsync(map.value, '/images/map-images/icon-red.png'),
      loadImageAsync(map.value, '/images/map-images/icon-yellow.png'),
      loadImageAsync(map.value, '/images/map-images/icon-green.png'),
      loadImageAsync(map.value, '/images/map-images/icon-blue.png'),
      loadImageAsync(map.value, '/images/map-images/icon-red-empty-favorite.png'),
      loadImageAsync(map.value, '/images/map-images/icon-red-favorite.png'),
      loadImageAsync(map.value, '/images/map-images/icon-yellow-favorite.png'),
      loadImageAsync(map.value, '/images/map-images/icon-green-favorite.png'),
      loadImageAsync(map.value, '/images/map-images/icon-blue-favorite.png')
    ])
      .then(images => {
        if (initialLoad) {
          images.forEach((image: any, index) => {
            if (map.value) map.value.addImage(`marker-${index}`, image)
          })
        }

        if (map.value) {
          map.value.addLayer({
            id: 'unclustered-point',
            type: 'symbol',
            source: 'markers',
            filter: ['!', ['has', 'point_count']],
            layout: {
              'icon-image': [
                'match',
                ['get', 'estateStatus'],
                'YELLOW', [
                  'case',
                  ['get', 'inFavorites'],
                  'marker-7',
                  'marker-2'
                ],
                'GREEN', [
                  'case',
                  ['get', 'inFavorites'],
                  'marker-8',
                  'marker-3'
                ],
                'BLUE', [
                  'case',
                  ['get', 'inFavorites'],
                  'marker-9',
                  'marker-4'
                ],
                'GREY', [
                  'case',
                  ['>=', ['get', 'fillPercent'], 1],
                  [
                    'case',
                    ['get', 'inFavorites'],
                    'marker-6',
                    'marker-1'
                  ],
                  [
                    'case',
                    ['get', 'inFavorites'],
                    'marker-5',
                    'marker-0'
                  ]
                ],
                'marker-0'
              ],
              'icon-size': 0.7,
              'icon-offset': [
                'case',
                ['get', 'inFavorites'],
                ['literal', [8, -47]],
                ['literal', [0, -40]]
              ],
              'text-field': [
                'match',
                ['get', 'estateStatus'],
                'BLUE', '',
                ['case',
                  ['==', ['get', 'fillPercent'], 0],
                  '',
                  ['concat', ['to-string', ['get', 'fillPercent']], '%']
                ]
              ],
              'icon-allow-overlap': true,
              'symbol-sort-key': ['get', 'fillPercent'],
              'text-font': ['Metropolis Semi Bold'],
              'text-size': 11,
              'text-offset': [
                'case',
                ['get', 'inFavorites'],
                ['literal', [0, -1.6]],
                ['literal', [0, -1.6]]
              ],
              'text-justify': 'center',
              'text-allow-overlap': true
            },
            paint: {
              'text-color': '#ffffff'
            }
          })
        }
      })
      .catch(error => {
        console.error('Error loading images:', error)
      })
  }
}

onMounted(() => {
  config.apiKey = import.meta.env.VITE_MAPTILER_API_KEY

  function isWebglSupported() {
    if (window.WebGLRenderingContext) {
      const canvas = document.createElement('canvas')
      try {
        const context = canvas.getContext('webgl2') || canvas.getContext('webgl')
        if (context && typeof context.getParameter == 'function') {
          return true
        }
      } catch (e) {
        console.log('WebGL is supported, but disabled')
      }
      return false
    }
    console.log('WebGL not supported')
    return false
  }

  if (!isWebglSupported()) {
    alert('Your browser does not support MapLibre GL')
    return
  }

  if (mapContainer.value) {
    map.value = markRaw(new Map({
      container: mapContainer.value,
      style: getMapStyleUrl(commonStore.isDarkMode),
      center: [initialState.lng, initialState.lat],
      zoom: initialState.zoom,
      maxZoom: 20,
      minZoom: 5,
      // borders of Ukraine
      maxBounds: [
        [22.137955, 44.385997],
        [40.227802, 52.379189]
      ],
      pitch: 45,
      antialias: true,
      geolocateControl: "bottom-right",
      navigationControl: "bottom-right",
      scaleControl: "bottom-left"
    }))
  }

  if (map.value) {
    // set language
    map.value.on('style.load', async () => {
      updateLanguage(map.value, Tr.currentLocale)
    })

    class RulerControl {
      private map?: Map
      private container?: HTMLDivElement
      private button?: HTMLButtonElement
      private span?: HTMLSpanElement

      onAdd(map: Map): HTMLDivElement {
        this.map = map
        this.container = document.createElement('div')
        this.container.classList.add('maplibregl-ctrl', 'maplibregl-ctrl-group')
        this.button = document.createElement('button')
        this.button.classList.add('maplibregl-ctrl-ruler')
        this.button.type = 'button'
        this.button.setAttribute('aria-pressed', 'false')
        this.span = document.createElement('span')
        this.span.classList.add('maplibregl-ctrl-icon')
        this.span.setAttribute('aria-hidden', 'true')

        this.button.appendChild(this.span)
        this.container.appendChild(this.button)

        return this.container
      }

      onRemove() {
        if (this.container && this.container.parentNode) {
          this.container.parentNode.removeChild(this.container)
        }
        this.map = undefined
      }
    }

    // @ts-ignore
    // map.value.addControl(new RulerControl(), 'bottom-right')

    rulerBtn = document.querySelector('.maplibregl-ctrl-ruler')

    if (rulerBtn) {
      rulerBtn.addEventListener('click', function () {
        toggleRuler()
      })
    }

    // map loaded
    map.value.on('load', async () => {
      if (buildingsStore.addresses.length) {
        addDataToMap(true)
      }

      // Change cursor when hovering over markers
      // on clustered point
      map.value?.on('mouseenter', 'clusters', () => {
        if (map.value) map.value.getCanvas().style.cursor = 'pointer'
      })
      map.value?.on('mouseleave', 'clusters', () => {
        if (map.value) map.value.getCanvas().style.cursor = ''
      })
      // on unclustered point
      map.value?.on('mouseenter', 'unclustered-point', () => {
        if (map.value) map.value.getCanvas().style.cursor = 'pointer'
      })
      map.value?.on('mouseleave', 'unclustered-point', () => {
        if (map.value) map.value.getCanvas().style.cursor = ''
      })

      // inspect a cluster on click and zoom in
      map.value?.on('click', 'clusters', (e) => {
        const features: any = map.value?.queryRenderedFeatures(e.point, {
          layers: ['clusters']
        })

        if (features) {
          const clusterId = features[0].properties.cluster_id
          // @ts-ignore
          map.value?.getSource('markers')?.getClusterExpansionZoom(
            clusterId,
            (err, zoom) => {
              if (err) return

              map.value?.easeTo({
                center: features[0].geometry.coordinates,
                zoom
              })
            }
          )
        }
      })

      // Adding a click handler to markers
      map.value?.on('click', 'unclustered-point', (e: any) => {
        if (e.features) {
          const coordinates = e.features[0].geometry.coordinates.slice()
          let objectUrlObj = Tr.i18nRoute({name: 'objects', params: e.features[0].properties.id})
          let objectUrlString = ''

          if (objectUrlObj.params.locale) objectUrlString = `/${objectUrlObj.params.locale}`
          objectUrlString += `/${objectUrlObj.name}/O-${e.features[0].properties.id}`

          const popupComponent = createVNode(MarkerPopup, {
            media: e.features[0].properties.media || (commonStore.isDarkMode ? '/images/different/house-stub-dark.png' : '/images/different/house-stub.png'),
            inFavorites: e.features[0].properties.inFavorites,
            name: e.features[0].properties.name,
            region: e.features[0].properties.region,
            area: e.features[0].properties.area,
            areaLabel: t('objects.card.area_label'),
            fillPercentNumber: e.features[0].properties.fillPercent,
            fillPercent: e.features[0].properties.estateStatus !== 'BLUE' ?
              `${Math.round(e.features[0].properties.fillPercent)}% ${t('application.details.labels.percent_status')}` :
              t('application.details.labels.percent_status_full'),
            fillPercentLabel: t('application.details.labels.status'),
            id: e.features[0].properties.id,
            objectStatus: e.features[0].properties.objectStatus ? t(`sell_application.lists.house_condition_enums.${e.features[0].properties.objectStatus}`) : null,
            price: e.features[0].properties.price,
            priceLabel: t('objects.card.price_label_for_map'),
            btnLabel: t('map.popup.button'),
            btnUrl: objectUrlString
          })
          const container = document.createElement('div')
          render(popupComponent, container)

          if (map.value) {
            popup = new Popup()
              .setLngLat(coordinates)
              .setDOMContent(container)
              .addTo(map.value)
          }
        }
      })
    })

    // ruler
    map.value.on('click', (e) => {
      if (!isRulerActive.value) return

      const features = map.value?.queryRenderedFeatures(e.point, {
        layers: ['measure-points']
      })

      // Remove the linestring from the group
      // So we can redraw it based on the points collection
      if (geojson.features.length > 1) geojson.features.pop()

      // If a feature was clicked, remove it from the map
      if (features && features.length) {
        const id = features[0].properties.id
        geojson.features = geojson.features.filter((point: any) => {
          return point.properties.id !== id
        })

        if (geojson.features.length === 1) {
          geojson.features[0].properties.distance = ''
        }

      } else {
        const point = {
          'type': 'Feature',
          'geometry': {
            'type': 'Point',
            'coordinates': [e.lngLat.lng, e.lngLat.lat]
          },
          'properties': {
            'id': String(new Date().getTime()),
            'distance': ''
          }
        }

        geojson.features.push(point)
      }

      if (geojson.features.length > 1) {
        // count distance
        linestring.geometry.coordinates = geojson.features.map(
          (point: any) => {
            return point.geometry.coordinates
          }
        )

        geojson.features.push(linestring)
        // @ts-ignore
        const lengthInKilometers = turf.length(linestring)
        const formattedDistance = lengthInKilometers >= 1
          ? `${lengthInKilometers.toFixed(2)}km`
          : `${(lengthInKilometers * 1000).toFixed(0)}m`

        // set distance text label near last point on the map
        geojson.features[geojson.features.length - 2].properties.distance = formattedDistance

        for (let i = 0; i < geojson.features.length - 2; i++) {
          geojson.features[i].properties.distance = ''
        }
      }
      // @ts-ignore
      map.value.getSource('geojson').setData(geojson)
    })
  }
})

watch(
  () => [
    Tr.currentLocale,
    commonStore.isDarkMode,
    buildingsStore.addresses,
    userStore.favorites.total,
    buildingsStore.addresses.length
  ],
  (value, oldValue) => {
    if (value[0] !== oldValue[0]) {
      if (popup) popup.remove()
      updateLanguage(map.value, Tr.currentLocale)
    }

    if (value[1] !== oldValue[1]) {
      if (popup) popup.remove()

      const styleUrl = getMapStyleUrl(value[1] as boolean)

      if (map.value) map.value.setStyle(styleUrl)

      setTimeout(() => {
        addDataToMap()

        updateLanguage(map.value, Tr.currentLocale)
      }, 500)
    }

    if (map.value && map.value.getSource('markers') && value[1] === oldValue[1]) {
      // @ts-ignore
      map.value.getSource('markers').setData({
        type: 'FeatureCollection',
        // @ts-ignore
        features: value[2].map((marker: IObjectAddress) => ({
          type: 'Feature',
          properties: {
            // add info about address for modal
            media: marker.image,
            inFavorites: userStore.isObjectInFavorites(marker.id),
            name: marker.name,
            region: marker.region,
            area: marker.area,
            estateStatus: marker.estate_status,
            fillPercent: Math.round(marker.fill_percent),
            id: marker.id,
            objectStatus: marker.object_status,
            price: marker.price
          },
          geometry: {
            type: 'Point',
            coordinates: [
              marker.coordinates[1],
              marker.coordinates[0]
            ]
          }
        }))
      })

      let curRegion = (value[2][0]) ? value[2][0].region : false
      let curAddress = (value[2][0]) ? value[2][0].name : false

      if (
        (currentRegion.value !== curRegion ||
          currentAddress.value !== curAddress) &&
        value[3] === oldValue[3]
      ) {
        fitMarkersToScreen(value[2])
      }

      currentRegion.value = curRegion
      currentAddress.value = curAddress
    }

    if (value[4] !== oldValue[4]) {
      if (map.value && map.value.style._loaded) {
        addDataToMap(true)
      }
    }
  }
)

function fitMarkersToScreen(markers) {
  if (!markers.length) return

  const markerCoordinates = markers.map(i => {
    return {lat: i.coordinates[0], lng: i.coordinates[1]}
  })

  let bounds = new LngLatBounds()

  for (const coord of markerCoordinates) {
    bounds.extend([coord.lng, coord.lat])
  }

  map.value?.fitBounds(bounds, {
    padding: 50
  })
}

const toggleRuler = () => {
  isRulerActive.value = !isRulerActive.value

  if (isRulerActive.value && map.value) {
    if (rulerBtn) rulerBtn.classList.add('active')

    map.value.addSource('geojson', {
      'type': 'geojson',
      'data': geojson
    })

    // Add styles to the map
    map.value.addLayer({
      id: 'measure-points',
      type: 'circle',
      source: 'geojson',
      paint: {
        'circle-radius': 5,
        'circle-color': '#044275'
      },
      filter: ['in', '$type', 'Point']
    })

    map.value.addLayer({
      id: 'measure-point-labels',
      type: 'symbol',
      source: 'geojson',
      layout: {
        'text-field': ['get', 'distance'],
        'text-font': ['Metropolis Semi Bold'],
        'text-size': 12,
        'text-offset': [0, 1]
      },
      paint: {
        'text-color': '#044275'
      },
      filter: ['in', '$type', 'Point']
    })


    map.value.addLayer({
      id: 'measure-lines',
      type: 'line',
      source: 'geojson',
      layout: {
        'line-cap': 'round',
        'line-join': 'round'
      },
      paint: {
        'line-color': '#044275',
        'line-width': 2
      },
      filter: ['in', '$type', 'LineString']
    })

    map.value.getCanvas().style.cursor = 'crosshair'

  } else {
    if (rulerBtn) rulerBtn.classList.remove('active')

    if (map.value) {
      map.value.removeLayer('measure-points')
      map.value.removeLayer('measure-point-labels')
      map.value.removeLayer('measure-lines')

      geojson.features = []
      // @ts-ignore
      map.value.getSource('geojson').setData(geojson)
      map.value.removeSource('geojson')
      map.value.getCanvas().style.cursor = 'pointer'
    }
  }
}

onBeforeRouteLeave(() => {
  if (map.value) {
    const clustersLayer = map.value.getLayer('clusters')
    const unclusteredPointsLayer = map.value.getLayer('unclustered-point')
    const markersSource = map.value.getSource('markers')

    if (clustersLayer) map.value.removeLayer('clusters')
    if (unclusteredPointsLayer) map.value.removeLayer('unclustered-point')
    if (markersSource) map.value.removeSource('markers')

    map.value = null
  }

  buildingsStore.addresses = []
  buildingsStore.facets = {}
})

onUnmounted(() => {
  map.value?.remove()
})
</script>

<template>
  <div class="app-map">
    <ObjectsFilters/>

    <div
      class="app-map__map"
      ref="mapContainer"></div>

    <div class="app-map__legend">
      <MapLegend/>
    </div>

    <div class="app-map__btn">
      <LocalizedLink
        to="default-objects"
        :query="route.query">
        <AppButton
          accent
          small
          :label="$t('map.btn_label')">
          <svg width="18" height="14" fill="none">
            <use xlink:href="/icons/sprite.svg#icon-list"></use>
          </svg>
        </AppButton>
      </LocalizedLink>
    </div>
  </div>
</template>
