<template>
  <v-container
    class="fill-height"
    pa-0
    ma-0
    fluid
  >
    <div class="infobox">
      <info
        v-if="!overlay"
        :general-info="generalInfo"
        :offset="10"
        :color-func="color"
        :rotation-toggle="rotationToggle"
        @toggle="rotationToggle = !rotationToggle"
      />
    </div>
    <tooltip
      :country="country"
      :offset-x="tooltipOffsetX"
      :offset-y="tooltipOffsetY"
      :opacity="opacity"
    />
    <canvas id="canvas" />
    <v-overlay :value="overlay">
      <v-progress-circular
        :size="50"
        :width="5"
        indeterminate
      />
    </v-overlay>
  </v-container>
</template>
<script>
import * as d3 from 'd3'
import * as topojson from 'topojson-client'
import * as versor from 'versor'
import * as scrapeIt from 'scrape-it'
import * as _ from 'lodash'
import world110m from '~/data/world-110m.json'
import Tooltip from '~/components/tooltip'
import Info from '~/components/info'

const water = { type: 'Sphere' }
const projection = d3.geoOrthographic().precision(0.1)

const rotationDelay = 3000
const scaleFactor = 0.9
const degPerSec = 6
const angles = { x: -20, y: 40, z: 0 }
const colorWater = '#fff'
const colorLand = '#111'
const degPerMs = degPerSec / 1000

export default {
  components: {
    Tooltip,
    Info
  },
  data () {
    return {
      height: null,
      width: null,
      canvas: null,
      context: null,
      path: null,
      sphere: { type: 'Sphere' },
      country: null,
      countryData: [],
      choroplethData: [],
      generalInfo: {},
      overlay: true,
      autorotate: null,
      scaleFactor: 0.9,
      v0: null,
      q0: null,
      r0: null,
      rotationDelay: 3000,
      land: null,
      borders: null,
      countries: null,
      lastTime: d3.now(),
      current: '',
      tooltipOffsetX: 0,
      tooltipOffsetY: 0,
      opacity: 0,
      offset: null,
      color: null,
      rotationToggle: true
    }
  },
  computed: {
    _ () {
      return _
    }
  },
  watch: {
    rotationToggle (value) {
      if (value) {
        this.startRotation()
      } else {
        this.stopRotation()
      }
    }
  },
  async mounted () {
    await this.loadData()
    this.generateMap()
    this.offset = document.documentElement.clientWidth - this.height
    this.overlay = false
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.scale)
  },
  methods: {
    async loadData () {
      const options = {
        countries: {
          listItem: 'tr',
          data: {
            country: 'td:nth-child(1)',
            totalCases: 'td:nth-child(2)',
            newCases: 'td:nth-child(3)',
            totalDeaths: 'td:nth-child(4)',
            newDeaths: 'td:nth-child(5)'
          }
        },
        totalCases: '.maincounter-number > span[style="color:#aaa"]',
        deaths: '.maincounter-number:not([style]) > span:not([style])',
        recovered: '.maincounter-number[style="color:#8ACA2B "] > span'
      }
      const { data } = await scrapeIt('https://cors-anywhere.herokuapp.com/https://www.worldometers.info/coronavirus/', options)
      this.countryData = _.groupBy(_.filter(data.countries, 'country'), 'country')
      this.generalInfo = _.pick(data, ['totalCases', 'deaths', 'recovered'])
      this.choroplethData = data.countries
    },
    someFunc () {
      this.rotationToggle = !this.rotationToggle
    },
    generateMap () {
      this.current = d3.select('#current')
      this.canvas = d3.select('#canvas')
      this.context = this.canvas.node().getContext('2d')
      this.path = d3.geoPath(projection).context(this.context)

      this.setAngles()

      window.d3 = d3

      this.canvas
        .call(d3.drag()
          .on('start', this.dragStarted)
          .on('drag', this.dragged)
          .on('end', this.dragEnded)
        )
        .on('mousemove', this.mouseMove)

      this.borders = topojson.mesh(world110m, world110m.objects.countries, (a, b) => a !== b)
      this.countries = topojson.feature(world110m, world110m.objects.countries).features
      this.land = topojson.feature(world110m, world110m.objects.land)

      this.loadChoropleth()

      window.addEventListener('resize', this.scale)
      this.scale()
      this.autorotate = d3.timer(this.rotate)
    },

    loadChoropleth () {
      this.choroplethData = new Map(_.map(_.filter(this.choroplethData, 'country'), (item) => {
        return [item.country, _.toNumber(_.replace(item.totalCases, /,/g, ''))]
      }))

      _.each(this.countries, (country) => {
        const props = country.properties
        if (!this.choroplethData.get(props.name)) {
          const value = this.choroplethData.get(props.alternateName)
          this.choroplethData.set(props.name, value)
        }
      })
      this.color = d3.scaleSequential()
        .domain(d3.extent(Array.from(this.choroplethData.values())))
        .interpolator(d3.interpolateRdGy)
        .unknown('#ccc')
    },

    enter (country) {
      const props = country.properties
      const found = _.first(_.get(this.countryData, props.name) || _.get(this.countryData, props.alternateName))
      if (found) {
        this.country = found
        this.current.text(props.name + 'found data')
        this.opacity = 1
        this.stopRotation()
      } else {
        this.current.text(props.name || '')
      }
    },

    leave (country) {
      this.current.text('')
      this.opacity = 0
      if (this.rotationToggle) {
        this.startRotation(3000)
      }
    },

    setAngles () {
      const rotation = projection.rotate()
      rotation[0] = angles.y
      rotation[1] = angles.x
      rotation[2] = angles.z
      projection.rotate(rotation)
    },

    scale () {
      this.width = document.documentElement.clientWidth
      this.height = document.documentElement.clientHeight
      this.canvas.attr('width', this.width).attr('height', this.height)
      projection
        .scale((scaleFactor * Math.min(this.width, this.height)) / 2)
        .translate([this.width / 2, this.height / 2])
      this.render()
    },

    startRotation (delay) {
      this.autorotate.restart(this.rotate, delay || 0)
    },

    stopRotation () {
      this.autorotate.stop()
    },

    dragStarted () {
      this.v0 = versor.cartesian(projection.invert([d3.event.x, d3.event.y]))
      this.r0 = projection.rotate()
      this.q0 = versor(this.r0)
      this.stopRotation()
    },

    dragged () {
      const v1 = versor.cartesian(projection.rotate(this.r0).invert([d3.event.x, d3.event.y]))
      const q1 = versor.multiply(this.q0, versor.delta(this.v0, v1))
      const r1 = versor.rotation(q1)
      projection.rotate(r1)
      this.render()
    },

    dragEnded () {
      this.startRotation(rotationDelay)
    },

    render () {
      this.context.clearRect(0, 0, this.width, this.height)
      // this.fill(water, colorWater)
      // this.fill(this.land, colorLand)
      // this.stroke(this.borders, colorWater, 0.5)
      this.stroke(water, colorLand, 1.5)
      this.renderChoroplethData()
      if (this.currentCountry) {
        this.fill(this.currentCountry, colorWater)
        this.stroke(this.currentCountry, '#f00', 1)
      }
    },

    renderChoroplethData () {
      const color = this.color
      _.each(this.countries, (country) => {
        const name = country.properties.name
        const colorVal = color(this.choroplethData.get(name))
        this.fill(country, colorVal)
      })
    },

    fill (obj, color) {
      const context = this.context
      context.beginPath()
      this.path(obj)
      context.fillStyle = color
      context.fill()
    },

    stroke (obj, color, lineWidth = null) {
      const context = this.context
      context.beginPath()
      this.path(obj)
      context.strokeStyle = color
      if (lineWidth) {
        context.lineWidth = lineWidth
      }
      context.stroke()
    },

    rotate (elapsed) {
      const now = d3.now()
      const diff = now - this.lastTime
      if (diff < elapsed) {
        const rotation = projection.rotate()
        rotation[0] += diff * degPerMs
        projection.rotate(rotation)
        this.render()
      }
      this.lastTime = now
    },

    polygonContains (polygon, point) {
      const n = polygon.length
      const p = polygon[n - 1]
      const [x, y] = [point[0], point[1]]
      let [x0, y0] = [p[0], p[1]]
      let inside = false
      for (let i = 0; i < n; ++i) {
        const p = polygon[i]
        const [x1, y1] = [p[0], p[1]]
        if (((y1 > y) !== (y0 > y)) && (x < (x0 - x1) * (y - y1) / (y0 - y1) + x1)) { inside = !inside }
        [x0, y0] = [x1, y1]
      }
      return inside
    },

    mouseMove () {
      this.tooltipOffsetX = d3.event.x + 20
      this.tooltipOffsetY = d3.event.y + 20
      const c = this.getCountry([d3.event.x, d3.event.y], this.countries)
      if (!c) {
        if (this.currentCountry) {
          this.leave(this.currentCountry)
          this.currentCountry = undefined
          this.render()
        }
        return
      }
      if (c === this.currentCountry) {
        return
      }
      this.currentCountry = c
      this.render()
      this.enter(c)
    },

    getCountry (coords, countries) {
      const pos = projection.invert(coords)
      return _.find(countries, (f) => {
        return _.find(f.geometry.coordinates, (c1) => {
          return this.polygonContains(c1, pos) || _.find(c1, (c2) => {
            return this.polygonContains(c2, pos)
          })
        })
      })
    }
  }
}
</script>

<style>

#tooltip {
  line-height: 1;
  font-weight: bold;
  padding: 12px;
  background: rgba(0, 0, 0, 0.8);
  color: #fff;
  border-radius: 2px;
  font-size: 12px;
}
#tooltip > strong {
  font-size: 14px;
}
.v-list-item {
  min-height: 32px;
}
.infobox {
  position:absolute;
  top: 10px;
  left: 10px;
}
</style>
