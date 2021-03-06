<template>
  <data-view
    :title="title"
    :title-id="titleId"
    :date="date"
    :head-title="title + infoTitles.join(',')"
  >
    <template v-slot:description>
      <slot name="description" />
    </template>
    <h4 :id="`${titleId}-graph`" class="visually-hidden">
      {{ $t(`{title}のグラフ`, { title }) }}
    </h4>
    <scrollable-chart v-show="canvas" :display-data="displayData">
      <template v-slot:chart="{ chartWidth }">
        <bar
          :ref="'barChart'"
          :chart-id="chartId"
          :chart-data="displayData"
          :options="displayOption"
          :height="240"
          :width="chartWidth"
        />
      </template>
      <template v-slot:sticky-chart>
        <bar
          class="sticky-legend"
          :chart-id="`${chartId}-header`"
          :chart-data="displayDataHeader"
          :options="displayOptionHeader"
          :plugins="yAxesBgPlugin"
          :height="240"
        />
      </template>
    </scrollable-chart>
    <template v-slot:additionalDescription>
      <slot name="additionalDescription" />
    </template>
    <template v-slot:dataTable>
      <client-only>
        <data-view-table :headers="tableHeaders" :items="tableData" />
      </client-only>
    </template>
    <template v-slot:dataSetPanel>
      <data-view-data-set-panel
        :title="infoTitles[0]"
        :l-text="displayInfo[0].lText"
        :s-text="displayInfo[0].sText"
        :unit="displayInfo[0].unit"
      />
    </template>
    <template v-slot:footer>
      <open-data-link v-show="url" :url="url" />
    </template>
  </data-view>
</template>

<script lang="ts">
import Vue from 'vue'
import { TranslateResult } from 'vue-i18n'
import { ThisTypedComponentOptionsWithRecordProps } from 'vue/types/options'
import { Chart } from 'chart.js'
import dayjs from 'dayjs'
import { GraphDataType } from '@/utils/formatGraph'
import DataView from '@/components/DataView.vue'
import DataViewTable, {
  TableHeader,
  TableItem,
} from '@/components/DataViewTable.vue'
import DataViewDataSetPanel from '@/components/DataViewDataSetPanel.vue'
import ScrollableChart from '@/components/ScrollableChart.vue'
import OpenDataLink from '@/components/OpenDataLink.vue'
import {
  DisplayData,
  DataSets,
  DataSetsPoint,
  yAxesBgPlugin,
} from '@/plugins/vue-chart'

import { getGraphSeriesStyle } from '@/utils/colors'
import { calcDayBeforeRatio } from '@/utils/formatDayBeforeRatio'

type Data = {
  dataKind: 'transition'
  canvas: boolean
}
type Methods = {}

type Computed = {
  makeDashedRectangleData: {
    x: string
    y: number
  }[]
  displayInfo: [
    {
      lText: string
      sText: string
      unit: string
    }
  ]
  displayData: {
    labels: string[]
    datasets: (DataSets | DataSetsPoint)[]
  }
  displayOption: Chart.ChartOptions
  displayDataHeader: DisplayData
  displayOptionHeader: Chart.ChartOptions
  scaledTicksYAxisMax: number
  tableHeaders: TableHeader[]
  tableData: TableItem[]
}

type Props = {
  title: string
  titleId: string
  infoTitles: string[]
  chartId: string
  chartData: GraphDataType[]
  date: string
  unit: string
  url: string
  dashedRectangleRange: string
  addedValue: number
  tableLabels: string[] | TranslateResult[]
  yAxesBgPlugin: Chart.PluginServiceRegistrationOptions[]
}

const options: ThisTypedComponentOptionsWithRecordProps<
  Vue,
  Data,
  Methods,
  Computed,
  Props
> = {
  created() {
    this.canvas = process.browser
  },
  components: {
    DataView,
    DataViewTable,
    DataViewDataSetPanel,
    ScrollableChart,
    OpenDataLink,
  },
  props: {
    title: {
      type: String,
      default: '',
    },
    titleId: {
      type: String,
      default: '',
    },
    infoTitles: {
      type: Array,
      default: () => [],
    },
    chartId: {
      type: String,
      default: 'dashed-rectangle-time-bar-chart',
    },
    chartData: {
      type: Array,
      default: () => [],
    },
    date: {
      type: String,
      required: true,
    },
    unit: {
      type: String,
      default: '',
    },
    url: {
      type: String,
      default: '',
    },
    dashedRectangleRange: {
      type: String,
      required: true,
    },
    addedValue: {
      type: Number,
      default: 0,
    },
    tableLabels: {
      type: Array,
      default: () => [],
    },
    yAxesBgPlugin: {
      type: Array,
      default: () => yAxesBgPlugin,
    },
  },
  data: () => ({
    dataKind: 'transition',
    canvas: true,
  }),
  computed: {
    makeDashedRectangleData() {
      const max = Math.max(...this.chartData.map((d) => d.transition))
      const firstDay = this.chartData[0].label
      return [
        { x: firstDay, y: 0 },
        { x: firstDay, y: max + this.addedValue },
        { x: this.dashedRectangleRange, y: max + this.addedValue },
        { x: this.dashedRectangleRange, y: 0 },
        { x: firstDay, y: 0 },
      ]
    },
    displayInfo() {
      const { lastDay, lastDayData, dayBeforeRatio } = calcDayBeforeRatio({
        displayData: this.displayData as DisplayData,
      })
      return [
        {
          lText: lastDayData,
          sText: `${this.$t('{date} の数値', {
            date: lastDay,
          })}（${this.$t('前日比')}: ${dayBeforeRatio} ${this.unit}）`,
          unit: this.unit,
        },
      ]
    },
    displayData() {
      const style = getGraphSeriesStyle(1)[0]
      return {
        labels: this.chartData.map((d) => d.label),
        datasets: [
          {
            type: 'bar',
            label: this.dataKind,
            data: this.chartData.map((d) => {
              return d.transition
            }),
            backgroundColor: style.fillColor,
            borderColor: style.strokeColor,
            borderWidth: 1,
            order: 1,
          },
          {
            type: 'line',
            label: 'dashed-rectangle',
            data: this.makeDashedRectangleData,
            pointBackgroundColor: 'rgba(0,0,0,0)',
            pointBorderColor: 'rgba(0,0,0,0)',
            borderColor: '#1b4d30',
            borderWidth: 2,
            borderDash: [4, 4],
            fill: false,
            order: 0,
            lineTension: 0,
          },
        ],
      }
    },
    displayOption() {
      const unit = this.unit
      const options: Chart.ChartOptions = {
        tooltips: {
          displayColors: false,
          filter(tooltipItem) {
            return tooltipItem.datasetIndex !== 1
          },
          callbacks: {
            label(tooltipItem) {
              return `${parseInt(tooltipItem.value!).toLocaleString()} ${unit}`
            },
            title(tooltipItem, data) {
              return data.labels![tooltipItem[0].index!] as string[]
            },
          },
        },
        maintainAspectRatio: false,
        legend: {
          display: false,
        },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false,
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: '#808080',
                maxRotation: 0,
                callback: (label: string) => {
                  return label.split('/')[1]
                },
              },
              // #2384: If you set "type" to "time", make sure that the bars at both ends are not hidden.
              // #2384: typeをtimeに設定する時はグラフの両端が見切れないか確認してください
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: true,
                drawBorder: false,
                tickMarkLength: 10,
              },
              ticks: {
                fontSize: 11,
                fontColor: '#808080',
                padding: 3,
                fontStyle: 'bold',
              },
              type: 'time',
              time: {
                unit: 'month',
                parser: 'M/D',
                displayFormats: {
                  month: 'MMM',
                },
              },
            },
          ],
          yAxes: [
            {
              stacked: true,
              gridLines: {
                display: true,
                color: '#E5E5E5',
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMax,
              },
            },
          ],
        },
      }
      if (this.$route.query.ogp === 'true') {
        Object.assign(options, { animation: { duration: 0 } })
      }
      return options
    },
    displayDataHeader() {
      return {
        labels: ['2020/1/1'],
        datasets: [
          {
            data: [Math.max(...this.chartData.map((d) => d.transition))],
            backgroundColor: 'transparent',
            borderWidth: 0,
          },
        ],
      }
    },
    displayOptionHeader() {
      const options: Chart.ChartOptions = {
        maintainAspectRatio: false,
        legend: {
          display: false,
        },
        tooltips: { enabled: false },
        scales: {
          xAxes: [
            {
              id: 'day',
              stacked: true,
              gridLines: {
                display: false,
              },
              ticks: {
                fontSize: 9,
                maxTicksLimit: 20,
                fontColor: 'transparent',
                maxRotation: 0,
                minRotation: 0,
                callback: (label: string) => {
                  return dayjs(label).format('D')
                },
              },
            },
            {
              id: 'month',
              stacked: true,
              gridLines: {
                drawOnChartArea: false,
                drawTicks: false, // true -> false
                drawBorder: false,
                tickMarkLength: 10,
              },
              ticks: {
                fontSize: 11,
                fontColor: 'transparent', // #808080
                padding: 13, // 3 + 10(tickMarkLength)
                fontStyle: 'bold',
              },
              type: 'time',
              time: {
                unit: 'month',
              },
            },
          ],
          yAxes: [
            {
              stacked: true,
              gridLines: {
                display: true,
                drawOnChartArea: false,
                color: '#E5E5E5', // #E5E5E5
              },
              ticks: {
                suggestedMin: 0,
                maxTicksLimit: 8,
                fontColor: '#808080', // #808080
                suggestedMax: this.scaledTicksYAxisMax,
              },
            },
          ],
        },
        animation: { duration: 0 },
      }
      return options
    },
    scaledTicksYAxisMax() {
      const values = this.chartData.map((d) => d.transition)
      return Math.max(...values) + this.addedValue
    },
    tableHeaders() {
      return [
        { text: this.$t('日付'), value: 'text' },
        {
          text: `${this.tableLabels[0]} (${this.$t('日別')})`,
          value: 'transition',
          align: 'end',
        },
      ]
    },
    tableData() {
      return this.chartData
        .map((d, _) => {
          return {
            text: d.label,
            transition: d.transition.toLocaleString(),
          }
        })
        .sort((a, b) => dayjs(a.text).unix() - dayjs(b.text).unix())
        .reverse()
    },
  },
  mounted() {
    const barChart = this.$refs.barChart as Vue
    const barElement = barChart.$el
    const canvas = barElement.querySelector('canvas')
    const labelledbyId = `${this.titleId}-graph`

    if (canvas) {
      canvas.setAttribute('role', 'img')
      canvas.setAttribute('aria-labelledby', labelledbyId)
    }
  },
}

export default Vue.extend(options)
</script>
