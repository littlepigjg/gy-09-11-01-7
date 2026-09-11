<template>
  <div>
    <div class="header">
      <h1>时序数据监控平台 · Time Series DB</h1>
      <div class="stats">
        <div class="stat"><div class="num">{{ metricsList.length }}</div><div class="label">指标</div></div>
        <div class="stat"><div class="num">{{ totalPoints.toLocaleString() }}</div><div class="label">数据点</div></div>
        <div class="stat">
          <div class="num" :style="{ color: live ? '#66bb6a' : '#7a869e' }">●</div>
          <div class="label">{{ live ? '实时采集中' : '已暂停' }}</div>
        </div>
      </div>
    </div>

    <div class="toolbar">
      <div class="group">
        <label>实例</label>
        <select v-model="instance" @change="onQuery">
          <option v-for="i in instances" :key="i" :value="i">{{ i || '全部' }}</option>
        </select>
      </div>
      <div class="group">
        <label>指标</label>
        <div class="metric-chips">
          <span
            v-for="m in metricNames"
            :key="m"
            class="chip"
            :class="{ on: selected.includes(m) }"
            @click="toggleMetric(m)"
          >{{ m }}</span>
        </div>
      </div>
    </div>

    <div class="toolbar">
      <div class="group">
        <label>时间范围</label>
        <button v-for="r in ranges" :key="r.sec" :class="{ active: rangeMode === 'preset' && rangeSec === r.sec }" @click="setRange(r.sec)">
          {{ r.label }}
        </button>
        <button :class="{ active: rangeMode === 'custom' }" @click="toggleCustomPanel">自定义</button>
        <span v-if="rangeMode === 'custom'" class="range-text">{{ customRangeText }}</span>
      </div>
      <div class="group">
        <label>聚合</label>
        <button v-for="a in ['min','max','avg','sum']" :key="a" :class="{ active: agg === a }" @click="setAgg(a)">
          {{ a.toUpperCase() }}
        </button>
      </div>
      <div class="group">
        <button :class="{ active: live }" @click="toggleLive">{{ live ? '暂停实时' : '开启实时' }}</button>
        <button class="primary" @click="onQuery">刷新查询</button>
      </div>
    </div>

    <div v-if="showCustom" class="custom-panel">
      <div class="row">
        <label>快捷选择</label>
        <button v-for="q in quickRanges" :key="q.label" @click="applyQuick(q)">{{ q.label }}</button>
      </div>
      <div class="row">
        <label>开始</label>
        <input type="datetime-local" step="1" v-model="customStartInput" />
        <label>结束</label>
        <input type="datetime-local" step="1" v-model="customEndInput" />
        <button class="primary" @click="applyCustom">应用</button>
        <button @click="showCustom = false">取消</button>
      </div>
      <div v-if="customError" class="error">{{ customError }}</div>
      <div class="hint">区间为左闭右开：包含开始秒、不包含结束秒（查全天请将结束设为次日 00:00:00）</div>
    </div>

    <div class="chart-card">
      <h3>多指标对比 · 降采样曲线（聚合: {{ agg.toUpperCase() }} / 桶: {{ queryMeta.bucket }}s ·
        数据源: {{ queryMeta.source === 'hourly' ? '小时预聚合表' : '原始分区表' }} ·
        耗时: {{ queryMeta.elapsed }}ms · 异常点: {{ anomalyCount }}）</h3>
      <div ref="historyChart" class="chart chart-tall"></div>
    </div>

    <div class="chart-card">
      <h3>实时曲线（最近 {{ liveWindow }} 秒原始点）· 异常点以红色标注</h3>
      <div ref="liveChart" class="chart"></div>
      <div class="meta">
        <span><span class="legend-dot" style="background:#ff5252"></span>异常点 (滑动窗口 Z-Score &gt; {{ anomalyThreshold }})</span>
        <span>异常总数: <span class="hl">{{ anomalyCount }}</span></span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount, nextTick } from 'vue'
import * as echarts from 'echarts'

const API = ''

const metricsList = ref([])
const totalPoints = ref(0)
const instances = ref([''])
const instance = ref('')
const metricNames = ref([])
const selected = ref(['cpu.usage', 'mem.usage'])
const ranges = [
  { sec: 900, label: '15分钟' },
  { sec: 3600, label: '1小时' },
  { sec: 21600, label: '6小时' },
  { sec: 86400, label: '24小时' },
  { sec: 172800, label: '2天' },
]
const rangeSec = ref(21600)
// 自定义时间范围: 'preset' 走固定档位, 'custom' 走用户输入的起止时间(秒级)
const rangeMode = ref('preset')
const showCustom = ref(false)
const customStartInput = ref('')
const customEndInput = ref('')
const customError = ref('')
const customRange = reactive({ start: 0, end: 0 }) // Unix 秒
const MAX_SPAN_SEC = 31 * 86400 // 与后端跨度上限一致

// 日期起点辅助: 返回今天偏移 offsetDays 天的 00:00:00 (本地时区)
function startOfDay(offsetDays) {
  const d = new Date()
  d.setHours(0, 0, 0, 0)
  d.setDate(d.getDate() + offsetDays)
  return d
}

// 常用时间段快捷选项: 返回 [起, 止] Date
// 注意: 后端查询为左闭右开区间 (ts >= start AND ts < end),
// 因此"全天"类区间的结束时间必须取下一天 00:00:00,
// 若取当天 23:59:59 会漏掉最后一秒的数据
const quickRanges = [
  {
    label: '今天',
    get: () => [startOfDay(0), new Date()],
  },
  {
    label: '昨天',
    get: () => [startOfDay(-1), startOfDay(0)],
  },
  {
    label: '前天',
    get: () => [startOfDay(-2), startOfDay(-1)],
  },
  {
    label: '本周',
    get: () => {
      const s = startOfDay(0)
      s.setDate(s.getDate() - ((s.getDay() + 6) % 7)) // 周一
      return [s, new Date()]
    },
  },
  {
    label: '最近7天',
    get: () => [new Date(Date.now() - 7 * 86400000), new Date()],
  },
]

// Date -> datetime-local 输入框格式 (本地时区, 精确到秒)
function fmtInput(d) {
  const p = n => String(n).padStart(2, '0')
  return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}T${p(d.getHours())}:${p(d.getMinutes())}:${p(d.getSeconds())}`
}
// datetime-local 值按本地时区解析为 Unix 秒
function parseInput(s) {
  const t = new Date(s).getTime()
  return Number.isNaN(t) ? NaN : Math.floor(t / 1000)
}
const fmtSec = ts => {
  const p = n => String(n).padStart(2, '0')
  const d = new Date(ts * 1000)
  return `${p(d.getMonth() + 1)}-${p(d.getDate())} ${p(d.getHours())}:${p(d.getMinutes())}:${p(d.getSeconds())}`
}
const customRangeText = computed(() =>
  rangeMode.value === 'custom' ? `${fmtSec(customRange.start)} ~ ${fmtSec(customRange.end)}` : ''
)

// 当前生效的查询区间 [start, end] (Unix 秒)
function effectiveRange() {
  if (rangeMode.value === 'custom') return [customRange.start, customRange.end]
  const end = Math.floor(Date.now() / 1000)
  return [end - rangeSec.value, end]
}

function toggleCustomPanel() {
  showCustom.value = !showCustom.value
  if (showCustom.value) {
    // 预填当前生效区间, 便于在现有档位基础上微调
    const [s, e] = effectiveRange()
    customStartInput.value = fmtInput(new Date(s * 1000))
    customEndInput.value = fmtInput(new Date(e * 1000))
    customError.value = ''
  }
}
function applyQuick(q) {
  const [s, e] = q.get()
  customStartInput.value = fmtInput(s)
  customEndInput.value = fmtInput(e)
  customError.value = ''
}
function applyCustom() {
  const start = parseInput(customStartInput.value)
  const end = parseInput(customEndInput.value)
  if (Number.isNaN(start) || Number.isNaN(end)) { customError.value = '请填写完整的起止时间'; return }
  if (end <= start) { customError.value = '结束时间必须大于开始时间'; return }
  if (end - start > MAX_SPAN_SEC) { customError.value = '时间跨度不能超过 31 天'; return }
  customRange.start = start
  customRange.end = end
  rangeMode.value = 'custom'
  showCustom.value = false
  onQuery()
}

const agg = ref('avg')
const live = ref(true)
const liveWindow = 300
const anomalyThreshold = 3.0
const anomalyCount = ref(0)

const queryMeta = reactive({ bucket: '-', source: 'raw', elapsed: '-' })

const historyChart = ref(null)
const liveChart = ref(null)
let histInst = null
let liveInst = null
let liveTimer = null
let liveAnomalyTimer = null

const COLORS = ['#4fc3f7', '#ffb74d', '#81c784', '#ba68c8', '#f06292', '#4dd0e1']

async function fetchJson(url) {
  const r = await fetch(API + url)
  return r.json()
}

async function loadMetrics() {
  const rows = await fetchJson('/api/metrics')
  metricsList.value = rows
  totalPoints.value = rows.reduce((s, r) => s + Number(r.points || 0), 0)
  const instSet = [...new Set(rows.map(r => r.instance))]
  instances.value = ['', ...instSet]
  const names = [...new Set(rows.map(r => r.name))]
  metricNames.value = names
  if (!selected.value.some(s => names.includes(s)) && names.length) {
    selected.value = names.slice(0, 2)
  }
}

function toggleMetric(m) {
  const i = selected.value.indexOf(m)
  if (i >= 0) selected.value.splice(i, 1)
  else selected.value.push(m)
  onQuery()
}
function setRange(s) { rangeMode.value = 'preset'; rangeSec.value = s; onQuery() }
function setAgg(a) { agg.value = a; onQuery() }
function toggleLive() { live.value = !live.value; scheduleLive() }

// ---------- 历史查询: 多指标对比 + 降采样 ----------
async function onQuery() {
  if (!selected.value.length) { histInst.setOption({ series: [] }); return }
  const [start, end] = effectiveRange()
  const q = new URLSearchParams({
    metrics: selected.value.join(','),
    instance: instance.value,
    start: String(start), end: String(end),
    agg: agg.value,
  })
  const data = await fetchJson('/api/query?' + q.toString())
  if (data.detail) { queryMeta.elapsed = '查询失败: ' + data.detail; return }
  queryMeta.bucket = data.bucket_seconds
  queryMeta.source = data.source
  queryMeta.elapsed = data.elapsed_ms

  const series = Object.entries(data.series || {}).map(([name, pts], idx) => ({
    name,
    type: 'line',
    smooth: true,
    showSymbol: false,
    lineStyle: { width: 2 },
    itemStyle: { color: COLORS[idx % COLORS.length] },
    data: pts.map(p => [p.ts * 1000, p.value]),
    connectNulls: true,
  }))

  // 对第一个选中指标做异常点检测, 在历史曲线上以红色散点标注
  // 跨度超过 7 天后端会拒绝(需拉全部原始点), 前端直接跳过
  const anomalies = (end - start) <= 7 * 86400
    ? await fetchAnomalies(selected.value[0], start, end)
    : []
  anomalyCount.value = anomalies.length
  if (anomalies.length) {
    series.push({
      name: '异常点',
      type: 'scatter',
      symbolSize: 11,
      itemStyle: { color: '#ff5252', borderColor: '#fff', borderWidth: 1 },
      data: anomalies.map(a => [a.ts, a.value]),
      z: 10,
    })
  }

  histInst.setOption(buildBaseOption(false, series), true)
}

async function fetchAnomalies(metric, start, end) {
  if (!metric) return []
  const q = new URLSearchParams({
    metric, instance: instance.value,
    start: String(start), end: String(end),
    threshold: String(anomalyThreshold),
  })
  const data = await fetchJson('/api/anomalies?' + q.toString())
  return data.anomalies || []
}

// ---------- 实时曲线 + 异常点标注 ----------
async function refreshLive() {
  if (!selected.value.length) return
  const q = new URLSearchParams({
    metrics: selected.value.join(','),
    instance: instance.value,
    window: String(liveWindow),
  })
  const data = await fetchJson('/api/latest?' + q.toString())

  const series = Object.entries(data.series || {}).map(([name, pts], idx) => ({
    name,
    type: 'line',
    smooth: true,
    showSymbol: false,
    lineStyle: { width: 2 },
    itemStyle: { color: COLORS[idx % COLORS.length] },
    data: pts.map(p => [p.ts, p.value]),
    connectNulls: true,
  }))
  liveInst.setOption(buildBaseOption(true, series), { replaceMerge: ['series'] })
}

async function refreshAnomalies() {
  // 对第一个选中指标做异常点标注
  const metric = selected.value[0]
  if (!metric) return
  const end = Math.floor(Date.now() / 1000)
  const start = end - liveWindow
  const q = new URLSearchParams({ metric, instance: instance.value, start: String(start), end: String(end), threshold: String(anomalyThreshold) })
  const data = await fetchJson('/api/anomalies?' + q.toString())
  const anomalies = data.anomalies || []
  anomalyCount.value = anomalies.length
  const scatter = {
    name: '异常点',
    type: 'scatter',
    symbolSize: 12,
    itemStyle: { color: '#ff5252', borderColor: '#fff', borderWidth: 1 },
    data: anomalies.map(a => [a.ts, a.value]),
    z: 10,
    tooltip: {
      formatter: p => `异常点<br/>值: ${p.value[1].toFixed(2)}<br/>${new Date(p.value[0]).toLocaleTimeString()}`,
    },
  }
  // 追加异常点散点序列(不清空已有曲线)
  const opt = liveInst.getOption()
  liveInst.setOption({ series: [...opt.series.filter(s => s.name !== '异常点'), scatter] })
}

function buildBaseOption(isLive, series) {
  return {
    backgroundColor: 'transparent',
    tooltip: {
      trigger: 'axis',
      backgroundColor: '#1e2940',
      borderColor: '#31405e',
      textStyle: { color: '#d7dde8' },
    },
    legend: {
      data: series.map(s => s.name),
      textStyle: { color: '#8b96ad' },
      top: 0,
    },
    grid: { left: 56, right: 24, top: 40, bottom: 56 },
    xAxis: {
      type: 'time',
      axisLine: { lineStyle: { color: '#31405e' } },
      axisLabel: { color: '#7a869e' },
      splitLine: { show: false },
    },
    yAxis: {
      type: 'value',
      scale: true,
      axisLine: { lineStyle: { color: '#31405e' } },
      axisLabel: { color: '#7a869e' },
      splitLine: { lineStyle: { color: '#1d273b' } },
    },
    dataZoom: isLive
      ? [{ type: 'inside' }, { type: 'slider', height: 18, bottom: 12, borderColor: '#31405e', fillerColor: 'rgba(79,195,247,0.15)' }]
      : [{ type: 'inside' }, { type: 'slider', height: 18, bottom: 12, borderColor: '#31405e', fillerColor: 'rgba(79,195,247,0.15)' }],
    series,
  }
}

function scheduleLive() {
  clearInterval(liveTimer)
  clearInterval(liveAnomalyTimer)
  if (live.value) {
    liveTimer = setInterval(refreshLive, 2000)
    liveAnomalyTimer = setInterval(refreshAnomalies, 5000)
    refreshLive()
    refreshAnomalies()
  }
}

onMounted(async () => {
  await nextTick()
  histInst = echarts.init(historyChart.value, 'dark')
  liveInst = echarts.init(liveChart.value, 'dark')
  window.addEventListener('resize', () => { histInst.resize(); liveInst.resize() })
  await loadMetrics()
  await onQuery()
  scheduleLive()
  // 指标列表每 10 秒刷新一次统计
  setInterval(loadMetrics, 10000)
})

onBeforeUnmount(() => {
  clearInterval(liveTimer)
  clearInterval(liveAnomalyTimer)
})
</script>
