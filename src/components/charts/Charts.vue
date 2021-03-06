<template>
  <div class="is-fluid">
    <div class="box">
      <h3 class="price-header">BTC Price - <span class="price">{{btcCurrentPrice}}</span> <span class="price-diff gain" v-bind:class="{ lost: btcPercentChange24h < 0 }">({{btcPercentChange24h}}%)</span></h3>
      <canvas id="btc-chart"></canvas>
    </div>
    <div class="box">
      <h3 class="price-header">ETH Price - <span class="price">{{ethCurrentPrice}}</span> <span class="price-diff gain" v-bind:class="{ lost: ethPercentChange24h < 0 }">({{ethPercentChange24h}}%)</span></h3>
      <canvas id="eth-chart"></canvas>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js'
import axios from 'axios'
import config from '@/config.js'
import moment from 'moment'

export default {
  name: 'ChartComponent',
  data () {
    return {
      samples: 0,
      speed: 0,
      timeout: 0,
      values: [],
      charts: [],
      value: 0,
      scale: 1,
      btcValues: [],
      ethValues: [],
      instance: null,
      btcMinPrice: config.chart.btcMin,
      btcMaxPrice: config.chart.btcMax,
      ethMinPrice: config.chart.ethMin,
      ethMaxPrice: config.chart.ethMax,
      btcCurrentPrice: 0,
      ethCurrentPrice: 0,
      btcPercentChange24h: 0,
      ethPercentChange24h: 0,
    }
  },
  async mounted () {
    this.instance = axios.create({
      baseURL: config.chart.endPoint,
      headers: {
        'Content-Type': 'application/json'
      }
    })
    this.samples = 2000
    this.speed = 2000
    
    // Load BTC
    this.btcValues = this.loadBTCPriceListFromStorage()
    if (this.btcValues.length === 0) {
      this.addEmptyValues(this.btcValues, this.samples)
    }
    this.updateBTCMinMax()

    // Load ETH
    this.ethValues = this.loadETHPriceListFromStorage()
    if (this.ethValues.length === 0) {
      this.addEmptyValues(this.ethValues, this.samples)
    }
    this.updateETHMinMax()

    this.initialize()
    if (config.isProd) {
      this.runChartPriceProd()
    } else {
      this.runChartPriceDev()
    }
  },
  methods: {
    async runChartPriceDev () {
      for (let i = 0; i < 100; i++) {
        this.advance()
        await this.sleep(10)
      }
    },
    async runChartPriceProd () {
      for (;;) {
        this.advance()
        await this.sleep(10)
      }
    },
    initialize () {
      this.charts.push(new Chart(document.getElementById('btc-chart'), {
        type: 'line',
        data: {
          datasets: [{
            data: this.btcValues,
            backgroundColor: 'rgba(255, 99, 132, 0.1)',
            borderColor: 'rgb(255, 99, 132)',
            borderWidth: 2
          }]
        },
        options: {
          responsive: true,
          animation: {
            duration: this.speed * 0.15,
            easing: 'linear'
          },
          legend: false,
          scales: {
            xAxes: [{
              type: 'time',
              display: true
            }],
            yAxes: [{
              ticks: {
                max: this.btcMaxPrice,
                min: this.btcMinPrice,
                // stepSize: 0.25,
                maxTicksLimit: 10
              }
            }]
          }
        }
      }), new Chart(document.getElementById('eth-chart'), {
        type: 'line',
        data: {
          datasets: [{
            data: this.ethValues,
            backgroundColor: 'rgba(0, 71, 187, 0.1)',
            borderColor: 'rgb(0, 71, 187)',
            borderWidth: 2
          }]
        },
        options: {
          responsive: true,
          animation: {
            duration: this.speed * 0.15,
            easing: 'linear'
          },
          legend: false,
          scales: {
            xAxes: [{
              type: 'time',
              display: true
            }],
            yAxes: [{
              ticks: {
                max: this.ethMaxPrice,
                min: this.ethMinPrice,
                // stepSize: 0.25,
                maxTicksLimit: 10
              }
            }]
          }
        }
      }))
    },
    addEmptyValues (arr, n) {
      for (var i = 0; i < n; i++) {
        arr.push({
          x: moment().subtract((n - i) * this.speed, 'milliseconds').toDate(),
          y: null
        })
      }
    },
    updateCharts () {
      this.charts.forEach(
        (chart) => {
          // chart.config.options.scales.yAxes[0].ticks.max = this.ethMaxPrice
          // console.log('chart: ', chart.canvas.id)
          if (chart.canvas.id === 'btc-chart') {
            chart.config.options.scales.yAxes[0].ticks.max = this.btcMaxPrice * 1.01
            chart.config.options.scales.yAxes[0].ticks.min = this.btcMinPrice * 0.99
          }else if (chart.canvas.id === 'eth-chart') {
            chart.config.options.scales.yAxes[0].ticks.max = this.ethMaxPrice * 1.01
            chart.config.options.scales.yAxes[0].ticks.min = this.ethMinPrice * 0.99
          }
          chart.update()
        }
      )
    },
    progressBTCPrice () {
      this.loadData('1/').then(
        (data) => {
          var price = data.quotes.USD.price
          this.btcValues.push({
            x: new Date(),
            y: price
          })
          this.btcCurrentPrice = price
          this.updateBTCMinMax()
          this.btcValues.shift()

          this.btcPercentChange24h = data.quotes.USD.percent_change_24h
        }
      )
      this.saveBTCPriceListToStorage()
    },
    updateBTCMinMax () {
      this.btcMaxPrice = Math.max(...this.btcValues.map(btcValue => btcValue.y))
      this.btcMinPrice = Math.min(...this.btcValues.filter(btcValue => btcValue.y !== null && btcValue.y !== 0).map(btcValue => btcValue.y))
      console.log(`btcMaxPrice ${this.btcMaxPrice}`)
      console.log(`btcMinPrice ${this.btcMinPrice}`)
    },
    updateETHMinMax () {
      this.ethMaxPrice = Math.max(...this.ethValues.map(ethValue => ethValue.y))
      this.ethMinPrice = Math.min(...this.ethValues.filter(ethValue => ethValue.y !== null && ethValue.y !== 0).map(ethValue => ethValue.y))
      console.log(`ethMaxPrice ${this.ethMaxPrice}`)
      console.log(`ethMinPrice ${this.ethMinPrice}`)
    },
    progressETHPrice () {
      this.loadData('1027/').then(
        (data) => {
          var price = data.quotes.USD.price
          this.ethValues.push({
            x: new Date(),
            y: price
          })
          this.ethCurrentPrice = price
          this.updateETHMinMax()
          this.ethValues.shift()

          this.ethPercentChange24h = data.quotes.USD.percent_change_24h
        }
      )
      this.saveETHPriceListToStorage()
    },
    advance () {
      if (this.values[0] !== null && this.scale < 4) {
        // rescale();
        this.updateCharts()
      }
      
      this.progressBTCPrice()
      this.progressETHPrice()
      this.updateCharts()
    },
    sleep (time) {
      return new Promise(resolve => setTimeout(resolve, time * 1000))
    },
    async loadData (endpoint) {
      const response = await this.instance.get(endpoint)
      return response.data.data
    },
    saveBTCPriceListToStorage () {
      localStorage.setItem('BTCPriceList', JSON.stringify(this.btcValues));
    },
    saveETHPriceListToStorage () {
      localStorage.setItem('ETHPriceList', JSON.stringify(this.ethValues));
    },
    loadBTCPriceListFromStorage () {
      if (localStorage.getItem('BTCPriceList')) {
        return JSON.parse(localStorage.getItem('BTCPriceList'));
      } else {
        return []
      }
    },
    loadETHPriceListFromStorage () {
      if (localStorage.getItem('ETHPriceList')) {
        return JSON.parse(localStorage.getItem('ETHPriceList'));
      } else {
        return []
      }
    },
  }
}
</script>

<style>
.price-header {
  font-size: 2em;
  font-weight: bold;
}

.price {
  font-weight: bold;
}

.price-diff {
  font-weight: bold;
  margin-left: 12px;
}

.gain {
  color: green;
}

.lost {
  color: red;
}
</style>
