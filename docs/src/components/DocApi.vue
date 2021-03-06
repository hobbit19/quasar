<template lang="pug">
q-card.doc-api.q-my-lg(v-if="ready", flat, bordered)
  q-toolbar.text-grey-8.bg-white
    card-title(:title="name", prefix="API--")
    q-space
    .col-auto.text-grey {{ type }}

  q-separator

  div.bg-grey-2.text-grey-7.flex.no-wrap
    q-tabs.col(v-model="currentTab", indicator-color="primary", align="left", :breakpoint="0", dense)
      q-tab(
        v-for="tab in tabs"
        :key="`api-tab-${tab}`"
        :name="tab"
        :label="tab"
      )

    q-input.q-mx-sm(
      v-if="$q.screen.gt.xs"
      ref="input",
      v-model="filter",
      dense,
      input-class="text-right",
      borderless,
      placeholder="Filter..."
      style="min-width: 150px"
    )
      template(v-slot:append)
        q-icon.cursor-pointer(
          :name="filter !== '' ? 'clear' : 'search'"
          @click="onFilterClick"
        )

  q-separator

  q-tab-panels(v-model="currentTab", animated)
    q-tab-panel(v-for="tab in tabs", :name="tab", :key="tab" class="q-pa-none")
      .row.no-wrap.api-container(v-if="aggregationModel[tab]")
        .col-auto.row.items-center.bg-grey-1.text-grey-7.q-py-lg
          q-tabs(
            v-model="currentInnerTab[tab]",
            active-color="primary",
            indicator-color="primary",
            :breakpoint="0",
            vertical,
            dense,
            shrink
          )
            q-tab(
              v-for="category in apiTabs(tab)"
              :key="`api-inner-tab-${category}`"
              class="inner-tab"
              :name="category"
            )
              .row.no-wrap.items-center
                q-badge(v-if="apiCount(tab, category)") {{ formattedApiCount(tab, category) }}
                span.q-ml-xs.text-capitalize.text-weight-medium {{ category }}

        q-separator(vertical)

        q-tab-panels.col(v-model="currentInnerTab[tab]", animated)
          q-tab-panel(v-for="category in apiTabs(tab)", :name="category", :key="category", class="q-pa-none")
            ApiRows(:which="tab", :apiKey="category", :api="filteredApi[tab]")
      .api-container(v-else)
        ApiRows(:which="tab", :api="filteredApi")
</template>

<script>
import ApiRows from './ApiRows.js'
import CardTitle from './CardTitle.vue'
import { format } from 'quasar'
const { pad } = format

const groupBy = (list, groupKey, defaultGroupKeyValue) => {
  const res = {}

  for (let key in list) {
    if (list.hasOwnProperty(key)) {
      let value = list[key]
      let groupKeyValue = (value[groupKey] || defaultGroupKeyValue).split('|')
      for (let groupKeyV of groupKeyValue) {
        if (res[groupKeyV] === void 0) {
          res[groupKeyV] = {}
        }
        res[groupKeyV][key] = value
      }
    }
  }

  return res
}

export default {
  name: 'DocApi',

  components: {
    ApiRows,
    CardTitle
  },

  props: {
    file: {
      type: String,
      required: true
    }
  },

  data () {
    return {
      ready: false,
      currentTab: null,
      currentInnerTab: {
        props: null
      },
      filter: '',
      filteredApi: {}
    }
  },

  watch: {
    filter (val) {
      val = val.trim().toLowerCase()

      if (val === '') {
        this.filteredApi = this.api
        return
      }

      const api = {}

      this.tabs.forEach(tab => {
        if (tab === 'injection') {
          api[tab] = this.api[tab]
          return
        }

        const filterApi = tabApi => {
          const filtered = {}

          Object.keys(tabApi).forEach(name => {
            if (
              (name.indexOf(val) > -1) ||
              (tabApi[name].desc !== void 0 && tabApi[name].desc.toLowerCase().indexOf(val) > -1)
            ) {
              filtered[name] = tabApi[name]
            }
          })
          return filtered
        }

        if (this.aggregationModel[tab]) {
          api[tab] = {}
          for (let group in this.api[tab]) {
            if (this.api[tab].hasOwnProperty(group)) {
              api[tab][group] = filterApi(this.api[tab][group])
            }
          }

          if (this.currentTab === tab) {
            let apiWithResultsCount = 0,
              lastFoundApiWithResults = null
            for (let group in this.api[tab]) {
              if (Object.keys(api[tab][group]).length > 0) {
                apiWithResultsCount++
                lastFoundApiWithResults = group
              }
            }

            if (apiWithResultsCount === 1) {
              this.currentInnerTab[tab] = lastFoundApiWithResults
            }
          }
        }
        else {
          api[tab] = filterApi(this.api[tab])
        }
      })

      this.filteredApi = api
    }
  },

  methods: {
    parseJson (name, { type, behavior, ...api }) {
      this.aggregationModel = {}

      if (type === 'component') {
        for (let apiGroup of ['props']) {
          api[apiGroup] = groupBy(api[apiGroup], 'category', 'general')
          this.currentInnerTab[apiGroup] = this.apiTabs(apiGroup, api)[0]
          this.aggregationModel[apiGroup] = true
        }
      }
      this.api = api
      this.filteredApi = api
      this.apiType = type

      this.name = name
      this.type = `${type === 'plugin' ? 'Quasar' : 'Vue'} ${type.charAt(0).toUpperCase()}${type.substring(1)}`
      this.tabs = Object.keys(api)

      if (
        behavior !== void 0 &&
        behavior.$listeners !== void 0
      ) {
        !this.tabs.includes('events') && this.tabs.push('events')
        this.api.events = {
          $listeners: behavior.$listeners,
          ...(this.api.events || {})
        }
      }

      this.currentTab = this.tabs[0]
    },

    onFilterClick () {
      if (this.filter !== '') {
        this.filter = ''
      }
      this.$refs.input.focus()
    },

    apiTabs (tab, api) {
      return Object.keys((api || this.filteredApi)[tab]).sort()
    },

    apiCount (tab, category) {
      return Object.keys(this.filteredApi[tab][category]).length
    },

    formattedApiCount (tab, category) {
      return pad(this.apiCount(tab, category), (this.currentTabMaxCategoryPropCount + '').length)
    }
  },

  computed: {
    currentTabMaxCategoryPropCount () {
      const calculateFn = () => {
        let max = -1
        for (let category in this.filteredApi[this.currentTab]) {
          let count = this.apiCount(this.currentTab, category)
          if (count > max) {
            max = count
          }
        }
        return max
      }
      return this.aggregationModel[this.currentTab] ? calculateFn() : 0
    }
  },

  mounted () {
    import(
      /* webpackChunkName: "quasar-api" */
      /* webpackMode: "lazy-once" */
      `quasar/dist/api/${this.file}.json`
    ).then(json => {
      this.parseJson(this.file, json.default)
      this.ready = true
    })
  }
}
</script>

<style lang="stylus">
.doc-api
  .q-tab
    height 40px

  .inner-tab
    justify-content left

  .api-container
    max-height 600px
</style>
