<template>
  <div v-if="ui" class="relative">
    <div class="flex justify-between">
      <div>
        <el-button type="primary" :icon="state.status === 'init' ? 'milk-tea': 'refresh-right'" class="focus:outline-none" :disabled="!allowClick()" plain   @click="fetchData()" :loading="state.status === 'loading'">{{state.status === 'init' ? ui.button.load: ui.button.update}}</el-button>
        <el-button icon="folder-opened" @click="saveExcel" class="focus:outline-none" :disabled="!gachaData"  type="success" plain>{{ui.button.excel}}</el-button>
        <el-button class="focus:outline-none" plain icon="edit" :type="state.showAnalyseStatus ? 'warning' : 'info'" @click="showAnalyse">{{state.showAnalyseStatus ? '返回详细信息' : '分析抽卡数据'}}</el-button>
        <el-tooltip v-if="detail && state.status !== 'loading'" :content="ui.hint.newAccount" placement="bottom">
          <el-button @click="newUser()" plain icon="plus" class="focus:outline-none"></el-button>
        </el-tooltip>
        <el-tooltip v-if="state.status === 'updated'" :content="ui.hint.relaunchHint" placement="bottom">
          <el-button @click="relaunch()" type="success" icon="refresh"   class="focus:outline-none" style="margin-left: 48px">{{ui.button.directUpdate}}</el-button>
        </el-tooltip>
      </div>
      <div class="flex gap-2">
        <el-select v-if="state.status !== 'loading' && state.dataMap && (state.dataMap.size > 1 || (state.dataMap.size === 1 && state.current === 0))" class="w-44"   @change="changeCurrent" v-model="uidSelectText">
          <el-option
            v-for="item of state.dataMap"
            :key="item[0]"
            :label="maskUid(item[0])"
            :value="item[0]">
          </el-option>
        </el-select>
        <el-dropdown @command="optionCommand"  >
          <el-button @click="showSetting(true)" class="focus:outline-none" plain type="info" icon="more"  >{{ui.button.option}}</el-button>
          <template #dropdown>
            <el-dropdown-menu>
              <el-dropdown-item command="setting" icon="setting">{{ui.button.setting}}</el-dropdown-item>
              <el-dropdown-item :disabled="!allowClick() || state.status === 'loading'" command="url" icon="link">{{ui.button.url}}</el-dropdown-item>
              <el-dropdown-item :disabled="!allowClick() || state.status === 'loading'" command="proxy" icon="position">{{ui.button.startProxy}}</el-dropdown-item>
            </el-dropdown-menu>
          </template>
        </el-dropdown>
      </div>
    </div>
    <p class="text-gray-400 my-2 text-xs">{{hint}}</p>
    <div v-if="detail && !state.showAnalyseStatus" class="gap-4 grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 2xl:grid-cols-4">
      <div class="mb-4" v-for="(item, i) of detail" :key="i">
        <div :class="{hidden: state.config.hideNovice && item[0] === '100'}">
          <p class="text-center text-gray-600 my-2">{{typeMap.get(item[0])}}</p>
          <pie-chart :data="item" :i18n="state.i18n" :typeMap="typeMap"></pie-chart>
          <gacha-detail :i18n="state.i18n" :data="item" :typeMap="typeMap"></gacha-detail>
        </div>
      </div>
    </div>
    <div v-show="state.showAnalyseStatus" style="height: 450px; width: 90%; margin: 0 auto;">
      <div id="line-graph" style="width: 100%; height: 100%;"></div>
      <p style="width: 100%; color: grey; font-size: 1em;">本页面统计数据来源于对https://github.com/OneBST/GI_gacha_dataset数据集的分析</p>
      <!-- <echarts :options="option" style="width: 1000px; height: 800px;"></echarts> -->
    </div>
    <Setting v-show="state.showSetting" :i18n="state.i18n" @changeLang="getI18nData()" @close="showSetting(false)"></Setting>

    <el-dialog :title="ui.urlDialog.title" v-model="state.showUrlDlg" width="90%" custom-class="max-w-md">
      <p class="mb-4 text-gray-500">{{ui.urlDialog.hint}}</p>
      <el-input  type="textarea" :autosize="{minRows: 4}" :placeholder="ui.urlDialog.placeholder" v-model="state.urlInput" spellcheck="false"></el-input>
      <template #footer>
        <span class="dialog-footer">
          <el-button  @click="state.showUrlDlg = false" class="focus:outline-none">{{ui.common.cancel}}</el-button>
          <el-button  type="primary" @click="state.showUrlDlg = false, fetchData(state.urlInput)" class="focus:outline-none">{{ui.common.ok}}</el-button>
        </span>
      </template>
    </el-dialog>

  </div>
</template>

<script setup>
import { reactive, computed, watch, onMounted } from 'vue'
import PieChart from './components/PieChart.vue'
import GachaDetail from './components/GachaDetail.vue'
import Setting from './components/Setting.vue'
import * as echarts from 'echarts'
import gachaDetail from './gachaDetail'
import { version } from '../../package.json'
const { ipcRenderer } = require('electron')
const { setTimeout } = require('timers')

const state = reactive({
  status: 'init',
  log: '',
  data: null,
  dataMap: new Map(),
  current: 0,
  showSetting: false,
  i18n: null,
  showUrlDlg: false,
  urlInput: '',
  config: {},
  showAnalyseStatus: false,
  myChart: null,
})

const ui = computed(() => {
  if (state.i18n) {
    return state.i18n.ui
  }
})

const gachaData = computed(() => {
  return state.dataMap.get(state.current)
})

const uidSelectText = computed(() => {
  if (state.current === 0) {
    return state.i18n.ui.select.newAccount
  } else {
    return state.current
  }
})

const allowClick = () => {
  const data = state.dataMap.get(state.current)
  if (!data) return true
  if (Date.now() - data.time < 1000 * 60) {
    return false
  }
  return true
}

const hint = computed(() => {
  const data = state.dataMap.get(state.current)
  if (!state.i18n) {
    return 'Loading...'
  }
  const { hint } = state.i18n.ui
  const { colon } = state.i18n.symbol
  if (state.status === 'init') {
    return hint.init
  } else if (state.status === 'loaded') {
    return `${hint.lastUpdate}${colon}${new Date(data.time).toLocaleString()}`
  } else if (state.status === 'loading') {
    return state.log || 'Loading...'
  } else if (state.status === 'updated') {
    return state.log
  } else if (state.status === 'failed') {
    return state.log + ` - ${hint.failed}`
  }
  return '　'
})

const detail = computed(() => {
  const data = state.dataMap.get(state.current)
  if (data) {
    return gachaDetail(data.result)
  }
})

const typeMap = computed(() => {
  const data = state.dataMap.get(state.current)
  return data.typeMap
})

const option = {
  xAxis: {
    type: 'category',
    data: null
  },
  yAxis: {
    type: 'value'
  },
  tooltip: {
    show: true,
    trigger: 'axis',
  },
  legend: {
    show: true,
    left: 'right',
    selectedMode: false
  },
  series: [
    {
      // 分析数据
      name: '统计数据',
      data: [0.008370895, 0.00772698, 0.00643915, 0.007083065, 0.00386349, 0.009658725, 0.005795235, 0.00643915, 0.004507405, 0.01030264,
             0.00643915, 0.005795235, 0.00643915, 0.008370895, 0.003219575, 0.004507405, 0.00515132, 0.00643915, 0.00772698, 0.00772698, 
             0.00386349, 0.00772698, 0.00772698, 0.004507405, 0.009658725, 0.00515132, 0.001931745, 0.00257566, 0.004507405, 0.00515132,
             0.005795235, 0.00386349, 0.007083065, 0.004507405, 0.004507405, 0.009658725, 0.00257566, 0.008370895, 0.003219575, 0.00515132, 
             0.00515132, 0.004507405, 0.00386349, 0.003219575, 0.00515132, 0.00257566, 0.00772698, 0.003219575, 0.00257566, 0.004507405, 
             0.003219575, 0.005795235, 0.00386349, 0.00386349, 0.003219575, 0.00386349, 0.00257566, 0.00386349, 0.00515132, 0.00643915, 
             0.00257566, 0.00386349, 0.003219575, 0.003219575, 0.003219575, 0.003219575, 0.00386349, 0.001931745, 0.00257566, 0.003219575, 
             0.004507405, 0.00257566, 0.003219575, 0.03992273, 0.072762395, 0.105602061, 0.104314231, 0.09916291, 0.085640695, 0.053444945, 
             0.031551835, 0.023824855, 0.008370895, 0.005795235, 0.00386349, 0.0, 0.000643915, 0.0, 0.0, 0.0],
      type: 'line',
      smooth: true,
      markLine: {
        data: [
          [
            {
              name: '平均出金点',
              label: {
                position: 'end'
              },
              coord: ['63', 0],
              symbol: 'none'
            },
            {
              coord: ['63', 1.0],
              symbol: 'none'
            }
          ],
          [
            {
              name: '最高出金点',
              label: {
                position: 'end'
              },
              coord: ['76', 0],
              symbol: 'none'
            },
            {
              coord: ['76', 1.0],
              symbol: 'none'
            }
          ]
        ]
      }
    },
    {
      name: '你的数据',
      data: null,
      type: 'line',
      smooth: true,
      markLine: {
        data: [
          [
            {
              name: '平均出金点',
              label: {
                position: 'middle'
              },
              coord: ['63', 0],
              symbol: 'none'
            },
            {
              coord: ['63', 1.0],
              symbol: 'none'
            }
          ]
        ]
      }
    },
  ]
}

const showAnalyse = async () => {
  state.showAnalyseStatus = ! state.showAnalyseStatus
  // console.log(state.showAnalyseStatus)
  if(!state.showAnalyseStatus){
    return;
  }

  let tmp = []
  for(let i = 0; i < 90; i++){
    tmp.push("" + (i + 1))
  }
  option.xAxis.data = tmp

  const data = await ipcRenderer.invoke('READ_DATA')
  let calcData = await getAnalyseData(data)
  option.series[1].data = calcData.freq
  option.series[1].markLine.data[0][0].coord[0] = "" + calcData.avr
  option.series[1].markLine.data[0][1].coord[0] = "" + calcData.avr
  if(!state.myChart){
    let chartDom = document.getElementById('line-graph')
    state.myChart = echarts.init(chartDom)
    state.getMyChart = true
  }
  let myChart = state.myChart
  myChart.setOption(option)

  let rangeY = myChart.getModel().getComponent('yAxis').axis.scale._extent;
  for(let itemIndex1 in option.series){
    let item1 = option.series[itemIndex1]
    if(!item1.markLine){
      continue
    }
    for(let itemIndex2 in item1.markLine.data){
      let item2 = item1.markLine.data[itemIndex2]
      item2[1].coord[1] = rangeY[1]
    }
  }
  myChart.clear()
  myChart.setOption(option)
}

const getAnalyseData = async (mapData) => {
  let current = mapData.current
  
  let userData = null
  for(let item of mapData.dataMap){
    if(item[0] === current){
      userData = item[1]
      break
    }
  }

  let rawData = null
  for(let item of userData.result){
    if(item[0] === '301'){ // 角色活动祈愿
      rawData = item[1]
    }
  }

  let countData = []  // 各抽出的频数
  let count = 0  // 总五星数
  let lastCount = 0  // 距上一抽
  let average = 0
  for(let i = 0; i < 90; i++){
    countData.push(0)
  }
  for(let itemIndex in rawData){
    let item = rawData[itemIndex]
    lastCount++
    if(item[3] === 5){
      if(lastCount > 90){
        console.log("?")
        throw new TypeError("lastCount should between 1 and 90, but get " + lastCount)
      }
      countData[lastCount - 1]++
      count ++
      lastCount = 0
    }
  }

  for(let i = 0; i < 90; i++){
    average += countData[i] * (i + 1)
    countData[i] = countData[i] / count
  }
  average /= count
  average = Math.round(average)

  return {
      freq: countData,
      avr: average
    }
}

const fetchData = async (url) => {
  state.status = 'loading'
  const data = await ipcRenderer.invoke('FETCH_DATA', url)
  if (data) {
    state.dataMap = data.dataMap
    state.current = data.current
    state.status = 'loaded'
  } else {
    state.status = 'failed'
  }
}

const readData = async () => {
  const data = await ipcRenderer.invoke('READ_DATA')
  if (data) {
    state.dataMap = data.dataMap
    state.current = data.current
    if (data.dataMap.get(data.current)) {
      state.status = 'loaded'
    }
  }
}

const getI18nData = async () => {
  const data = await ipcRenderer.invoke('I18N_DATA')
  if (data) {
    state.i18n = data
    setTitle()
  }
}

const saveExcel = async () => {
  await ipcRenderer.invoke('SAVE_EXCEL')
}

const changeCurrent = async (uid) => {
  if (uid === 0) {
    state.status = 'init'
  } else {
    state.status = 'loaded'
  }
  state.current = uid
  await ipcRenderer.invoke('CHANGE_UID', uid)
}

const newUser = async () => {
  await changeCurrent(0)
}

const relaunch = async () => {
  await ipcRenderer.invoke('RELAUNCH')
}

const maskUid = (uid) => {
  return `${uid}`.replace(/(.{3})(.+)(.{3})$/, '$1***$3')
}

const showSetting = (show) => {
  if (show) {
    state.showSetting = true
  } else {
    state.showSetting = false
    updateConfig()
  }
}

const optionCommand = (type) => {
  if (type === 'setting') {
    showSetting(true)
  } else if (type === 'url') {
    state.urlInput = ''
    state.showUrlDlg = true
  } else if (type === 'proxy') {
    fetchData('proxy')
  }
}

const setTitle = () => {
  document.title = `${state.i18n.ui.win.title} - v${version}`
}

const updateConfig = async () => {
  state.config = await ipcRenderer.invoke('GET_CONFIG')
}

onMounted(async () => {
  await readData()
  await getI18nData()

  ipcRenderer.on('LOAD_DATA_STATUS', (event, message) => {
    state.log = message
  })

  ipcRenderer.on('ERROR', (event, err) => {
    console.error(err)
  })

  ipcRenderer.on('UPDATE_HINT', (event, message) => {
    state.log = message
    state.status = 'updated'
  })

  await updateConfig()
})
</script>
