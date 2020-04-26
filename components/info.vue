<template>
  <div class="headline font-weight-light" small>
    <svg id="svg" /><br>
    Total Cases: {{ _.get(generalInfo, 'totalCases', 'N/A') }}<br>
    Deaths: {{ _.get(generalInfo, 'deaths', 'N/A') }}<br>
    Recovered: {{ _.get(generalInfo, 'totalCases', 'N/A') }}<br>
    <v-switch
      color="red"
      style="z-index:2"
      :label="`Rotation ${!rotationToggle ? 'On' : 'Off'}`"
      @change="$emit('toggle')"
    />
  </div>
</template>
<script>
import * as _ from 'lodash'
import legend from '~/legend'

export default {
  name: 'Info',
  props: {
    offset: {
      type: Number,
      default: 0
    },
    generalInfo: {
      type: Object,
      default: null
    },
    colorFunc: {
      type: Function,
      default: null
    },
    rotationToggle: {
      type: Boolean,
      default: false
    }
  },
  computed: {
    _ () {
      return _
    }
  },
  mounted () {
    if (this.colorFunc) {
      legend({ color: this.colorFunc, title: 'No. of Cases', height: 50 })
    }
  }
}
</script>
