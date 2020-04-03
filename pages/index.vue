<template>
  <v-container
    class="fill-height"
    pa-0
    ma-0
    fluid
  >
    <canvas id="canvas" :width="width" :height="height" />
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

export default {
  components: {
  },
  data () {
    return {
      height: 800,
      width: 800,
      projection: d3.geoOrthographic().precision(0.1),
      context: null,
      path: null,
      sphere: { type: 'Sphere' },
      country: null,
      countryData: [],
      overlay: true
    }
  },
  async mounted () {
    window.addEventListener('resize', this.generateMap)
    this.generateMap()
    await this.loadData()
    this.overlay = false
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.generateMap)
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
        }
      }
      const { data } = await scrapeIt('https://cors-anywhere.herokuapp.com/https://www.worldometers.info/coronavirus/', options)
      this.countryData = _.groupBy(_.filter(data.countries, 'country'), 'country')
    },
    generateMap () {
      this.width = document.getElementById('canvas').offsetWidth
      this.context = document.getElementById('canvas').getContext('2d')
      this.path = d3.geoPath(this.projection, this.context)

      const [[x0, y0], [x1, y1]] = d3.geoPath(this.projection.fitWidth(this.width, this.sphere)).bounds(this.sphere)
      this.height = Math.ceil(y1 - y0)
      const l = Math.min(Math.ceil(x1 - x0), this.height)
      this.projection.scale(this.projection.scale() * (l - 1) / l).precision(0.2)

      const borders = topojson.mesh(world110m, world110m.objects.countries, (a, b) => a !== b)
      const countries = topojson.feature(world110m, world110m.objects.countries).features
      const land = topojson.feature(world110m, world110m.objects.land)

      d3.select(this.context.canvas)
        .call(this.drag(this.projection)
          .on('drag.render', () => this.render(land, borders, countries))
          .on('end.render', () => this.render(land, borders, countries)))
        .call(() => this.render(land, borders, countries))
        .on('mousemove', () => this.mousemove(countries, land, borders))
        .node()

      d3.select('body').append('div')
        .append('div')
        .attr('id', 'tooltip')
        .attr('style', 'position: absolute; opacity: 0;')
    },

    mousemove (countries, land, borders) {
      const canvas = this.context.canvas
      const country = this.getCountry(d3.mouse(canvas), countries)
      if (country) {
        this.render(land, borders, country)
        this.country = country
        this.renderTooltip()
      } else {
        this.country = null
        d3.select('#tooltip')
          .style('opacity', 0)
      }
    },

    renderTooltip () {
      const props = this.country.properties
      const found = _.first(_.get(this.countryData, props.name) || _.get(this.countryData, props.alternateName))
      if (found) {
        d3.select('#tooltip')
          .style('left', (d3.event.pageX + 10) + 'px')
          .style('top', (d3.event.pageY + 10) + 'px')
          .style('opacity', 1)
          .html(this.renderTable(props, found))
      } else if (this.country) {
        d3.select('#tooltip')
          .style('left', (d3.event.pageX + 10) + 'px')
          .style('top', (d3.event.pageY + 10) + 'px')
          .style('opacity', 1)
          .html(`<strong>${props.name}</strong>`)
      }
    },

    renderTable (props, country) {
      return `
        <strong>${props.name}</strong>
        <table>
          <tr><td>Total Cases</td><td>${country.totalCases || 'N/A'}</td></tr>
          <tr><td>New Cases</td><td>${country.newCases || 'N/A'}</td></tr>
          <tr><td>Total Deaths</td><td>${country.totalDeaths || 'N/A'}</td></tr>
          <tr><td>New Deaths</td><td><span style="color:#f00">${country.newDeaths || 'N/A'}</span></td></tr>
        </table>
      `
    },

    getCountry (coords, countries) {
      const pos = this.projection.invert(coords)
      return _.find(countries, (f) => {
        return _.find(f.geometry.coordinates, (c1) => {
          return this.polygonContains(c1, pos) || _.find(c1, (c2) => {
            return this.polygonContains(c2, pos)
          })
        })
      })
    },

    polygonContains (polygon, point) {
      const n = polygon.length
      let p = polygon[n - 1]
      const [x, y] = [point[0], point[1]]
      let [x0, y0] = [p[0], p[1]]
      let x1, y1
      let inside = false
      for (let i = 0; i < n; ++i) {
        p = polygon[i];
        [x1, y1] = [p[0], p[1]]
        if (((y1 > y) !== (y0 > y)) && (x < (x0 - x1) * (y - y1) / (y0 - y1) + x1)) {
          inside = !inside
        }
        [x0, y0] = [x1, y1]
      }

      return inside
    },

    render (land, borders, country) {
      const context = this.context
      context.clearRect(0, 0, this.width, this.height)

      context.beginPath()
      this.path(land)
      context.fillStyle = '#000'
      context.fill()

      context.beginPath()
      this.path(country)
      context.fillStyle = '#fff'
      context.fill()
      context.lineWidth = 0.5
      context.strokeStyle = '#f00'
      context.stroke()

      context.beginPath()
      this.path(borders)
      context.strokeStyle = '#fff'
      context.lineWidth = 0.5
      context.stroke()

      context.beginPath()
      this.path(this.sphere)
      context.strokeStyle = '#000'
      context.lineWidth = 1.5
      context.stroke()

      return context.canvas
    },

    drag (projection) {
      let v0, q0, r0
      const dragstarted = () => {
        v0 = versor.cartesian(this.projection.invert([d3.event.x, d3.event.y]))
        q0 = versor(r0 = this.projection.rotate())
      }

      const dragged = () => {
        const v1 = versor.cartesian(this.projection.rotate(r0).invert([d3.event.x, d3.event.y]))
        const q1 = versor.multiply(q0, versor.delta(v0, v1))
        this.projection.rotate(versor.rotation(q1))
      }
      return d3.drag()
        .on('start', dragstarted)
        .on('drag', dragged)
    }

  }
}

</script>

<style>
.container {
  margin: 0 auto;
  min-height: 100vh;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

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

canvas{
  width: 304px;
}

@media (min-width: 480px) {
  canvas{
    width: 480px;
  }
}

@media (min-width: 768px) {
  canvas{
    width: 640px;
  }
}

@media (min-width: 992px) {
  canvas{
    width: 720px;
  }
}

@media (min-width: 1200px) {
  canvas{
    width: 800px;
  }
}
</style>
