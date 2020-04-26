<template>
  <div class="headline font-weight-light" small>
    <svg id="svg" /><br>
    Total Cases: {{ _.get(generalInfo, 'totalCases', 'N/A') }}<br>
    Deaths: {{ _.get(generalInfo, 'deaths', 'N/A') }}<br>
    Recovered: {{ _.get(generalInfo, 'totalCases', 'N/A') }}<br>
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
