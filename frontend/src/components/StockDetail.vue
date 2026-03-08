<script setup>
import { computed, onMounted, onBeforeUnmount, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { GetStockKLineWithPeriod, GetStockMinutePriceLineData, GetConfig } from '../../wailsjs/go/main/App'
import * as echarts from 'echarts'
import { ArrowBackOutline } from '@vicons/ionicons5'

const route = useRoute()
const router = useRouter()

const stockCode = ref(route.query.code || route.params.code)
const stockName = ref(route.query.name || route.params.name)

const darkTheme = ref(false)

const kLineChartRef = ref(null)
const fenshiChartRef = ref(null)
let kLineChart = null
let fenshiChart = null

const activeTab = ref('fenshi')
const currentPeriod = ref('day')
const feishiInterval = ref(null)

const upColor = '#ec0000'
const downColor = '#00da3c'

onMounted(() => {
  GetConfig().then(res => {
    darkTheme.value = res.darkTheme
  })

  if (stockCode.value) {
    initCharts()
    loadFenshiData()
  }
})

onBeforeUnmount(() => {
  clearInterval(feishiInterval.value)
  if (kLineChart) {
    kLineChart.dispose()
  }
  if (fenshiChart) {
    fenshiChart.dispose()
  }
})

function initCharts() {
  if (kLineChartRef.value) {
    kLineChart = echarts.init(kLineChartRef.value)
  }
  if (fenshiChartRef.value) {
    fenshiChart = echarts.init(fenshiChartRef.value)
  }
}

watch(activeTab, (newTab) => {
  clearInterval(feishiInterval.value)
  if (newTab === 'fenshi') {
    setTimeout(() => {
      if (!fenshiChart) fenshiChart = echarts.init(fenshiChartRef.value)
      loadFenshiData()
      feishiInterval.value = setInterval(() => {
        loadFenshiData()
      }, 1000 * 10)
    }, 100)
  } else {
    setTimeout(() => {
      if (!kLineChart) kLineChart = echarts.init(kLineChartRef.value)
      loadKLineData(newTab)
    }, 100)
  }
})

watch(darkTheme, () => {
  if (activeTab.value === 'fenshi') {
    loadFenshiData()
  } else {
    loadKLineData(activeTab.value)
  }
})

function loadFenshiData() {
  GetStockMinutePriceLineData(stockCode.value, stockName.value).then(result => {
    const priceData = result.priceData
    let category = []
    let price = []
    let openprice = 0
    let closeprice = 0
    let volume = []
    let min = 0
    let max = 0

    if(!priceData || priceData.length === 0) return

    openprice = priceData[0].price
    closeprice = priceData[priceData.length - 1].price

    for (let i = 0; i < priceData.length; i++) {
      category.push(priceData[i].time)
      price.push(priceData[i].price)
      if (min === 0 || min > priceData[i].price) min = priceData[i].price
      if (max < priceData[i].price) max = priceData[i].price

      if (i > 0) {
        let b = priceData[i].volume - priceData[i - 1].volume
        volume.push(b)
      } else {
        volume.push(priceData[i].volume)
      }
    }

    let option = {
      title: {
        text: `${stockName.value} (${stockCode.value}) 分时图`,
        subtext: "[" + result.date + "] 开盘:" + openprice + " 最新:" + closeprice,
        left: 'center',
        top: '10',
        textStyle: { color: darkTheme.value ? '#ccc' : '#333' }
      },
      darkMode: darkTheme.value,
      tooltip: {
        trigger: 'axis',
        axisPointer: { type: 'cross', animation: false }
      },
      axisPointer: { link: [{ xAxisIndex: 'all' }] },
      xAxis: [
        { type: 'category', data: category, axisLabel: { show: false } },
        { gridIndex: 1, type: 'category', data: category }
      ],
      yAxis: [
        {
          name: "股价", type: 'value',
          min: (min - min * 0.01).toFixed(2),
          max: (max + max * 0.01).toFixed(2),
          splitLine: { show: false }
        },
        {
          gridIndex: 1, name: "成交量", type: 'value',
          splitLine: { show: false }
        }
      ],
      grid: [
        { left: '8%', right: '8%', height: '50%' },
        { left: '8%', right: '8%', top: '70%', height: '15%' }
      ],
      series: [
        {
          name: "股价", data: price, type: 'line',
          showSymbol: false, lineStyle: { width: 2 },
          markLine: {
            symbol: 'none',
            data: [
              { yAxis: openprice, name: '开盘价', lineStyle: { color: '#FFCB00' } },
              { yAxis: closeprice, name: '收盘价', lineStyle: { color: 'red' } }
            ]
          }
        },
        {
          xAxisIndex: 1, yAxisIndex: 1, name: "成交量",
          data: volume, type: 'bar'
        }
      ]
    }

    if (fenshiChart) {
      fenshiChart.setOption(option)
    }
  })
}

function calculateMA(dayCount, values) {
  var result = []
  for (var i = 0, len = values.length; i < len; i++) {
    if (i < dayCount) {
      result.push('-')
      continue
    }
    var sum = 0
    for (var j = 0; j < dayCount; j++) {
      sum += +values[i - j][1]
    }
    result.push((sum / dayCount).toFixed(2))
  }
  return result
}

function loadKLineData(period) {
  let days = 365
  if(period === 'week') days = 365 * 3
  if(period === 'month') days = 365 * 5
  if(period === 'm60') days = 60

  GetStockKLineWithPeriod(stockCode.value, stockName.value, days, period).then(result => {
    if(!result) return
    const categoryData = []
    const values = []
    const volumns = []

    for (let i = 0; i < result.length; i++) {
      let resultElement = result[i]
      categoryData.push(resultElement.day)
      let flag = resultElement.close > resultElement.open ? 1 : -1
      values.push([
        resultElement.open,
        resultElement.close,
        resultElement.low,
        resultElement.high
      ])
      volumns.push([i, resultElement.volume / 10000, flag])
    }

    let option = {
      darkMode: darkTheme.value,
      animation: true,
      title: {
        text: `${stockName.value} (${stockCode.value}) K线图`,
        left: 'center',
        textStyle: { color: darkTheme.value ? '#ccc' : '#333' }
      },
      legend: {
        bottom: 10, left: 'center',
        data: ['K线', 'MA5', 'MA10', 'MA20'],
        textStyle: { color: darkTheme.value ? '#ccc' : '#456' }
      },
      tooltip: {
        trigger: 'axis',
        axisPointer: { type: 'cross' }
      },
      axisPointer: { link: [{ xAxisIndex: 'all' }] },
      visualMap: {
        show: false, seriesIndex: 5, dimension: 2,
        pieces: [{ value: -1, color: downColor }, { value: 1, color: upColor }]
      },
      grid: [
        { left: '10%', right: '8%', height: '50%' },
        { left: '10%', right: '8%', top: '63%', height: '16%' }
      ],
      xAxis: [
        { type: 'category', data: categoryData, boundaryGap: false, min: 'dataMin', max: 'dataMax' },
        { type: 'category', gridIndex: 1, data: categoryData, boundaryGap: false, axisLabel: { show: false }, min: 'dataMin', max: 'dataMax' }
      ],
      yAxis: [
        { scale: true, splitArea: { show: true } },
        { scale: true, gridIndex: 1, splitNumber: 2, axisLabel: { show: false }, splitLine: { show: false } }
      ],
      dataZoom: [
        { type: 'inside', xAxisIndex: [0, 1], start: 70, end: 100 },
        { show: true, type: 'slider', top: '85%', xAxisIndex: [0, 1], start: 70, end: 100 }
      ],
      series: [
        {
          name: 'K线', type: 'candlestick', data: values,
          itemStyle: { color: upColor, color0: downColor }
        },
        { name: 'MA5', type: 'line', data: calculateMA(5, values), smooth: true, showSymbol: false, lineStyle: { opacity: 0.6 } },
        { name: 'MA10', type: 'line', data: calculateMA(10, values), smooth: true, showSymbol: false, lineStyle: { opacity: 0.6 } },
        { name: 'MA20', type: 'line', data: calculateMA(20, values), smooth: true, showSymbol: false, lineStyle: { opacity: 0.6 } },
        { name: 'MA30', type: 'line', data: calculateMA(30, values), smooth: true, showSymbol: false, lineStyle: { opacity: 0.6 } },
        { name: '成交量(万手)', type: 'bar', xAxisIndex: 1, yAxisIndex: 1, data: volumns }
      ]
    }

    if(kLineChart) {
      kLineChart.setOption(option)
    }
  })
}

function goBack() {
  router.back()
}
</script>

<template>
  <div class="stock-detail-page">
    <n-card class="detail-card">
      <div class="header">
        <n-button quaternary circle @click="goBack">
          <n-icon :component="ArrowBackOutline" size="24" />
        </n-button>
        <h2 class="title">{{ stockName }} <span class="code">{{ stockCode }}</span></h2>
      </div>

      <n-tabs v-model:value="activeTab" type="segment" animated>
        <n-tab-pane name="fenshi" tab="分时">
          <div ref="fenshiChartRef" class="chart-container"></div>
        </n-tab-pane>
        <n-tab-pane name="m60" tab="60分钟">
          <div v-show="activeTab !== 'fenshi'" ref="kLineChartRef" class="chart-container"></div>
        </n-tab-pane>
        <n-tab-pane name="day" tab="日K">
          <div v-show="activeTab !== 'fenshi'" ref="kLineChartRef" class="chart-container"></div>
        </n-tab-pane>
        <n-tab-pane name="week" tab="周K">
          <div v-show="activeTab !== 'fenshi'" ref="kLineChartRef" class="chart-container"></div>
        </n-tab-pane>
        <n-tab-pane name="month" tab="月K">
          <div v-show="activeTab !== 'fenshi'" ref="kLineChartRef" class="chart-container"></div>
        </n-tab-pane>
      </n-tabs>
    </n-card>
  </div>
</template>

<style scoped>
.stock-detail-page {
  padding: 20px;
  height: calc(100vh - 40px);
  box-sizing: border-box;
}

.detail-card {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.header {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}

.title {
  margin: 0 0 0 16px;
  font-size: 24px;
  font-weight: 600;
}

.code {
  font-size: 16px;
  color: #888;
  margin-left: 8px;
}

.chart-container {
  width: 100%;
  height: calc(100vh - 200px);
  min-height: 500px;
}
</style>
