<template>
  <div class="display-1 font-weight-light" :style="`text-align:right;position:absolute;top:10px;left:${offset}px;`">
    Total Cases: {{ _.get(generalInfo, 'totalCases', 'N/A') }}<br>
    Deaths: {{ _.get(generalInfo, 'deaths', 'N/A') }}<br>
    Recovered: {{ _.get(generalInfo, 'totalCases', 'N/A') }}<br>
  </div>
</template>
<script>
import * as _ from 'lodash'

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
    }
  },
  computed: {
    _ () {
      return _
    }
  }
}
</script>
