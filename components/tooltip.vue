<template>
  <v-card
    class="mx-auto"
    min-width="200"
    dark
    :style="`position:absolute;top:${offsetY}px;left:${offsetX}px;opacity:${opacity};z-index:2`"
  >
    <v-list-item>
      <v-list-item-title class="headline">
        <center>{{ _.get(country, 'country', 'N/A') }}</center>
      </v-list-item-title>
    </v-list-item>
    <v-list class="transparent">
      <v-list-item>
        <v-list-item-title pa-0 style="font-size:0.8em">
          Total Cases
        </v-list-item-title>
        <v-list-item-subtitle pt-2 mt-2 class="text-right" style="font-size:0.8em;text-align:right">
          {{ _.get(country, 'totalCases') || 'N/A' }}
        </v-list-item-subtitle>
      </v-list-item>
      <v-list-item>
        <v-list-item-title pa-0 style="font-size:0.8em">
          New Cases
        </v-list-item-title>
        <v-list-item-subtitle class="text-right" style="font-size:0.8em;text-align:right">
          <span style="color:red">{{ _.get(country, 'newCases') || 'N/A' }}</span>
        </v-list-item-subtitle>
      </v-list-item>
      <v-list-item>
        <v-list-item-title pa-0 style="font-size:0.8em">
          Total Deaths
        </v-list-item-title>
        <v-list-item-subtitle class="text-right" style="font-size:0.8em;text-align:right">
          {{ _.get(country, 'totalDeaths') || 'N/A' }}
        </v-list-item-subtitle>
      </v-list-item>
      <v-list-item>
        <v-list-item-title pa-0 style="font-size:0.8em">
          New Deaths
        </v-list-item-title>
        <v-list-item-subtitle pa-0 class="text-right" style="font-size:0.8em;text-align:right">
          <span style="color:red">{{ _.get(country, 'newDeaths') || 'N/A' }}</span>
        </v-list-item-subtitle>
      </v-list-item>
    </v-list>
  </v-card>
</template>
<script>
import * as _ from 'lodash'

export default {
  name: 'Tooltip',
  props: {
    country: {
      type: Object,
      default: null
    },
    offsetX: {
      type: Number,
      default: 0
    },
    offsetY: {
      type: Number,
      default: 0
    },
    opacity: {
      type: Number,
      default: 0
    }
  },
  computed: {
    _ () {
      return _
    }
  }
}
</script>
</template>
