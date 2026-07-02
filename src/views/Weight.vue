<template>
  <div class="tracker-container">
    <div class="header-actions">
      <h2>⚖️ 体重追踪</h2>
      <el-button type="primary" :size="isMobile ? 'default' : 'large'" @click="openCreateDialog">记录体重</el-button>
    </div>

    <el-row :gutter="20">
      <!-- 左侧/上方：图表区域 -->
      <el-col :xs="24" :lg="16">
        <el-card shadow="never" class="chart-card">
          <template #header>
            <div class="card-header-flex">
              <span class="title">📅 体重 / 体脂趋势</span>
              <div class="header-controls">
                <el-date-picker
                  v-model="dateRange"
                  type="daterange"
                  range-separator="至"
                  start-placeholder="开始"
                  end-placeholder="结束"
                  size="small"
                  style="width: 200px"
                  :editable="false"
                  @change="handleDateRangeChange"
                />
                <el-switch v-model="focusWeightRange" size="small" active-text="聚焦" inactive-text="自动" />
                <el-switch v-model="showBodyFat" size="small" active-text="体脂" />
              </div>
            </div>
          </template>
          <div v-loading="loading" class="chart-box" :style="{ height: isMobile ? '300px' : '400px' }">
            <v-chart :key="chartKey" :option="chartOption" autoresize />
          </div>
        </el-card>

        <el-card shadow="never" class="chart-card mt-20">
          <template #header>
            <div class="card-header-flex">
              <span class="title">📉 体重变化预测 (周)</span>
            </div>
          </template>
          <div class="predict-controls">
            <div class="control-item">
              <span class="label">预测时长(天)</span>
              <el-input-number v-model="predictDays" :min="7" :max="180" :step="7" size="small" />
            </div>
            <div class="control-item">
              <span class="label">每日缺口(kcal)</span>
              <el-input-number v-model="avgDailyDeficit" :min="0" :max="1000" :step="50" size="small" />
            </div>
          </div>
          <div v-loading="loading" class="chart-box" :style="{ height: isMobile ? '300px' : '360px' }">
            <v-chart :key="predictChartKey" :option="predictChartOption" autoresize />
          </div>
          <div class="predict-tip">
            💡 规则：每 7 天体重变化 ≈ (平均单日热量缺口 / 1100) kg
          </div>
        </el-card>

        <el-card shadow="never" class="chart-card mt-20">
          <template #header>
            <div class="card-header-flex">
              <span class="title">📈 减脂状态分析 (固定时间轴)</span>
              <div class="header-controls">
                <span class="label">展示月数</span>
                <el-input-number v-model="analysisMonths" :min="1" :max="24" size="small" style="width: 100px" />
              </div>
            </div>
          </template>
          <div v-loading="loading" class="chart-box" :style="{ height: isMobile ? '300px' : '400px' }">
            <v-chart :key="analysisChartKey" :option="analysisChartOption" autoresize />
          </div>
          <div class="predict-tip">
            💡 说明：横轴固定为时间刻度，不受数据点密度影响，适合观察真实减脂速率。
          </div>
        </el-card>

        <el-card shadow="never" class="chart-card mt-20">
          <template #header>
            <div class="card-header-flex">
              <span class="title">🕯️ 周体重K线（OHLC）+ 4周EMA + 周净热量差</span>
            </div>
          </template>
          <div v-loading="loading" class="chart-box" :style="{ height: isMobile ? '360px' : '480px' }">
            <v-chart :key="weeklyKChartKey" :option="weeklyKChartOption" autoresize />
          </div>
          <div class="predict-tip">
            💡 时间范围固定近6个月；O=周首日体重，H=周内最高，L=周内最低，C=周末日体重；柱状图=周净热量差（摄入-消耗）。
          </div>
        </el-card>
      </el-col>

      <!-- 右侧/下方：列表区域 -->
      <el-col :xs="24" :lg="8">
        <el-card shadow="never" class="list-card mobile-mt-20">
          <template #header>
            <div class="card-header-flex">
              <span class="title">📝 历史记录</span>
              <el-button size="small" :loading="loading" @click="fetchWeightLogs">刷新</el-button>
            </div>
          </template>

          <el-table :data="pagedLogs" v-loading="loading" stripe style="width: 100%" size="small">
            <el-table-column label="时间" min-width="110">
              <template #default="{ row }">{{ formatTimeShort(row.recorded_at) }}</template>
            </el-table-column>
            <el-table-column label="体重" prop="weight" width="70">
              <template #default="{ row }"><b>{{ row.weight }}</b></template>
            </el-table-column>
            <el-table-column label="操作" width="90" align="right">
              <template #default="{ row }">
                <el-button link type="primary" @click="openEditDialog(row)">改</el-button>
                <el-button link type="danger" @click="handleDelete(row.id)">删</el-button>
              </template>
            </el-table-column>
          </el-table>

          <div class="pagination-container">
            <el-pagination
              v-model:current-page="pagination.currentPage"
              v-model:page-size="pagination.pageSize"
              :total="pagination.total"
              layout="prev, pager, next"
              small
              @current-change="handlePageChange"
            />
          </div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 弹窗 -->
    <el-dialog
      v-model="dialogVisible"
      :title="isEdit ? '编辑体重' : '新增记录'"
      :width="isMobile ? '90%' : '420px'"
      top="15vh"
      destroy-on-close
    >
      <el-form ref="formRef" :model="form" :rules="rules" label-position="top">
        <el-form-item label="记录时间" prop="recorded_at">
          <el-date-picker v-model="form.recorded_at" type="datetime" style="width: 100%" :editable="false" />
        </el-form-item>
        <el-row :gutter="15">
          <el-col :span="12">
            <el-form-item label="体重(kg)" prop="weight">
              <el-input-number v-model="form.weight" :precision="2" :step="0.1" controls-position="right" style="width: 100%" />
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="体脂(%)" prop="body_fat_percentage">
              <el-input-number v-model="form.body_fat_percentage" :precision="1" :step="0.1" controls-position="right" style="width: 100%" />
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click="dialogVisible = false">取消</el-button>
          <el-button type="primary" :loading="submitting" @click="submitLog">保存</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import { supabase } from '../lib/supabase.js'
import { currentUserId } from '../lib/userContext.js'
import { ElMessage, ElMessageBox } from 'element-plus'

import VChart from 'vue-echarts'
import * as echarts from 'echarts/core'
import { LineChart, BarChart, CandlestickChart } from 'echarts/charts'
import { TooltipComponent, GridComponent, LegendComponent, DataZoomComponent } from 'echarts/components'
import { CanvasRenderer } from 'echarts/renderers'

echarts.use([LineChart, BarChart, CandlestickChart, TooltipComponent, GridComponent, LegendComponent, DataZoomComponent, CanvasRenderer])


const loading = ref(false)
const submitting = ref(false)
const dialogVisible = ref(false)
const isEdit = ref(false)
const formRef = ref(null)
const allLogs = ref([])

const showBodyFat = ref(true)
const focusWeightRange = ref(true)
const predictDays = ref(56)
const avgDailyDeficit = ref(700)
const analysisMonths = ref(6)

// 日期筛选范围
const dateRange = ref(null)

const isMobile = ref(false)
const updateIsMobile = () => { isMobile.value = window.innerWidth <= 768 }

const pagination = reactive({ currentPage: 1, pageSize: 12, total: 0 })
const pagedLogs = computed(() => {
  const start = (pagination.currentPage - 1) * pagination.pageSize
  return allLogs.value.slice().sort((a,b) => new Date(b.recorded_at) - new Date(a.recorded_at)).slice(start, start + pagination.pageSize)
})

const handlePageChange = (page) => { pagination.currentPage = page }

const defaultForm = { id: null, recorded_at: new Date(), weight: 90, body_fat_percentage: null }
const form = reactive({ ...defaultForm })

const rules = {
  recorded_at: [{ required: true, message: '请选择时间', trigger: 'change' }],
  weight: [{ required: true, message: '请输入体重', trigger: 'change' }]
}

const fetchWeightLogs = async () => {
  loading.value = true
  const { data, error } = await supabase
    .from('weight_logs')
    .select('*')
    .eq('userid', currentUserId.value)
    .order('recorded_at', { ascending: true })
  if (!error) {
    allLogs.value = data || []
    pagination.total = allLogs.value.length
  }
  loading.value = false
}

const fetchWeeklyCalorieData = async (startDate, endDate) => {
  const { data, error } = await supabase
    .from('diet_logs')
    .select('recorded_at, calories, quantity, food_database(calories_per_100g)')
    .eq('userid', currentUserId.value)
    .gte('recorded_at', startDate.toISOString())
    .lte('recorded_at', endDate.toISOString())
    .order('recorded_at', { ascending: true })

  if (error || !data) return []

  // 转成每天总摄入热量
  const dayIntakeMap = new Map()
  for (const row of data) {
    const d = new Date(row.recorded_at)
    const dayKey = `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`

    let cals = Number(row.calories)
    if (!Number.isFinite(cals)) {
      const c100 = Number(row.food_database?.calories_per_100g)
      const qty = Number(row.quantity)
      cals = Number.isFinite(c100) && Number.isFinite(qty) ? (c100 * qty / 100) : 0
    }

    dayIntakeMap.set(dayKey, (dayIntakeMap.get(dayKey) || 0) + (Number.isFinite(cals) ? cals : 0))
  }

  // 推断TDEE：使用当前页面预测参数（缺口 + 最近7天平均摄入）作为估算
  // deficit = tdee - intake => tdee ≈ intake + deficit
  const dayKeys = Array.from(dayIntakeMap.keys()).sort()
  const intakeValues = dayKeys.map(k => dayIntakeMap.get(k) || 0)
  const recent = intakeValues.slice(-7)
  const avgIntake = recent.length ? recent.reduce((a,b)=>a+b,0) / recent.length : 2000
  const estimatedTdee = avgIntake + Number(avgDailyDeficit.value || 0)

  // 每日净热量差 = intake - tdee
  const dayBalanceMap = new Map()
  for (const [k, intake] of dayIntakeMap.entries()) {
    dayBalanceMap.set(k, intake - estimatedTdee)
  }

  return dayBalanceMap
}

const openCreateDialog = () => {
  isEdit.value = false
  Object.assign(form, defaultForm)
  form.recorded_at = new Date()
  if (allLogs.value.length) {
    const last = allLogs.value[allLogs.value.length - 1]
    form.weight = Number(last.weight)
    form.body_fat_percentage = last.body_fat_percentage
  }
  dialogVisible.value = true
}

const openEditDialog = (row) => {
  isEdit.value = true
  Object.assign(form, {
    id: row.id,
    recorded_at: new Date(row.recorded_at),
    weight: Number(row.weight),
    body_fat_percentage: row.body_fat_percentage
  })
  dialogVisible.value = true
}

const submitLog = async () => {
  if (!formRef.value) return
  await formRef.value.validate(async (valid) => {
    if (!valid) return
    submitting.value = true
    try {
      const payload = {
        userid: currentUserId.value,
        recorded_at: form.recorded_at,
        weight: form.weight,
        body_fat_percentage: form.body_fat_percentage
      }
      const { error } = isEdit.value 
        ? await supabase.from('weight_logs').update(payload).eq('id', form.id)
        : await supabase.from('weight_logs').insert(payload)
      if (error) throw error
      ElMessage.success('已保存')
      dialogVisible.value = false
      fetchWeightLogs()
    } catch (e) {
      ElMessage.error(e.message)
    } finally { submitting.value = false }
  })
}

const handleDelete = async (id) => {
  try {
    await ElMessageBox.confirm('确定删除吗？', '提示', { type: 'warning' })
    const { error } = await supabase.from('weight_logs').delete().eq('id', id)
    if (!error) { ElMessage.success('已删除'); fetchWeightLogs() }
  } catch {}
}

const formatTimeShort = (iso) => {
  const d = new Date(iso)
  return `${d.getMonth()+1}/${d.getDate()} ${String(d.getHours()).padStart(2,'0')}:${String(d.getMinutes()).padStart(2,'0')}`
}

const handleDateRangeChange = () => {
  // 仅仅触发 computed 重新计算，不需要手动操作数据
}

const chartKey = computed(() => `${focusWeightRange.value}-${showBodyFat.value}-${allLogs.value.length}-${dateRange.value}`)
const chartOption = computed(() => {
  // 过滤数据
  let filteredData = allLogs.value;
  if (dateRange.value && dateRange.value.length === 2) {
    const start = new Date(dateRange.value[0]).getTime();
    const end = new Date(dateRange.value[1]).getTime() + 86400000; // 包含结束当天
    filteredData = allLogs.value.filter(r => {
      const t = new Date(r.recorded_at).getTime();
      return t >= start && t <= end;
    });
  }

  const x = filteredData.map(r => {
    const d = new Date(r.recorded_at)
    return `${d.getFullYear()}-${d.getMonth() + 1}-${d.getDate()}`
  })
  const wY = filteredData.map(r => Number(r.weight))
  const series = [{ name: '体重(kg)', type: 'line', smooth: true, data: wY, symbolSize: 6, lineStyle: { width: 3 } }]
  
  if (showBodyFat.value) {
    series.push({ 
      name: '体脂(%)', type: 'line', smooth: true, 
      data: filteredData.map(r => r.body_fat_percentage), 
      yAxisIndex: 1, symbolSize: 6 
    })
  }

  return {
    tooltip: { trigger: 'axis', confine: true },
    legend: { top: 0, type: 'scroll' },
    grid: { left: '3%', right: '4%', top: '15%', bottom: '10%', containLabel: true },
    xAxis: { type: 'category', data: x, axisLabel: { rotate: x.length > 10 ? 45 : 0 } },
    yAxis: [
      { type: 'value', min: focusWeightRange.value ? 80 : 'dataMin', max: focusWeightRange.value ? 100 : 'dataMax', scale: true },
      { type: 'value', scale: true, show: showBodyFat.value }
    ],
    series,
    dataZoom: [{ type: 'inside' }]
  }
})

const predictChartKey = computed(() => `${allLogs.value.length}-${predictDays.value}-${avgDailyDeficit.value}`)
const predictChartOption = computed(() => {
  if (!allLogs.value.length) return {}
  const startW = Number(allLogs.value[allLogs.value.length-1].weight)
  const dropPerWeek = avgDailyDeficit.value / 1100
  const x = [], y = []
  const startD = new Date()
  for(let i=0; i<=Math.floor(predictDays.value/7); i++) {
    const d = new Date(startD); d.setDate(d.getDate() + i*7)
    x.push(`${d.getMonth()+1}/${d.getDate()}`)
    y.push(Number((startW - dropPerWeek * i).toFixed(2)))
  }
  return {
    tooltip: { trigger: 'axis', confine: true },
    grid: { left: '3%', right: '4%', top: '10%', bottom: '10%', containLabel: true },
    xAxis: { type: 'category', data: x },
    yAxis: { type: 'value', scale: true },
    series: [{ name: '预测体重', type: 'line', smooth: true, data: y, lineStyle: { type: 'dashed', color: '#6366f1' } }]
  }
})

const analysisChartKey = computed(() => `${allLogs.value.length}-${analysisMonths.value}`)
const analysisChartOption = computed(() => {
  if (!allLogs.value.length) return {}

  // 1. 确定终点（最新记录日期）和起点（终点减去 X 个月）
  const lastLog = allLogs.value[allLogs.value.length - 1]
  const endDate = new Date(lastLog.recorded_at)
  const startDate = new Date(endDate)
  startDate.setMonth(startDate.getMonth() - analysisMonths.value)

  // 2. 准备数据
  const seriesData = allLogs.value
    .filter(log => {
      const d = new Date(log.recorded_at)
      return d >= startDate && d <= endDate
    })
    .map(log => [new Date(log.recorded_at), Number(log.weight)])

  return {
    tooltip: {
      trigger: 'axis',
      confine: true,
      formatter: (params) => {
        const p = params[0]
        const d = new Date(p.value[0])
        return `${d.getFullYear()}-${d.getMonth() + 1}-${d.getDate()}<br/>体重: <b>${p.value[1]}</b> kg`
      }
    },
    grid: { left: '3%', right: '4%', top: '10%', bottom: '10%', containLabel: true },
    xAxis: {
      type: 'time',
      min: startDate,
      max: endDate,
      axisLabel: {
        formatter: (value) => {
          const d = new Date(value)
          return `${d.getMonth() + 1}/${d.getDate()}`
        }
      }
    },
    yAxis: {
      type: 'value',
      scale: true,
      min: (value) => Math.floor(value.min - 1),
      max: (value) => Math.ceil(value.max + 1)
    },
    series: [{
      name: '体重',
      type: 'line',
      smooth: true,
      showSymbol: seriesData.length < 50,
      data: seriesData,
      lineStyle: { width: 3, color: '#10b981' },
      areaStyle: {
        color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
          { offset: 0, color: 'rgba(16, 185, 129, 0.3)' },
          { offset: 1, color: 'rgba(16, 185, 129, 0)' }
        ])
      }
    }]
  }
})

const getWeekStart = (date) => {
  const d = new Date(date)
  d.setHours(0, 0, 0, 0)
  const day = d.getDay()
  const diff = day === 0 ? -6 : 1 - day
  d.setDate(d.getDate() + diff)
  return d
}

const weeklyKChartKey = computed(() => `weekly-k-${allLogs.value.length}-${avgDailyDeficit.value}`)
const weeklyKChartOption = computed(() => {
  if (!allLogs.value.length) return {}

  const sorted = allLogs.value.slice().sort((a, b) => new Date(a.recorded_at) - new Date(b.recorded_at))
  const latest = new Date(sorted[sorted.length - 1].recorded_at)
  const sixMonthsAgo = new Date(latest)
  sixMonthsAgo.setMonth(sixMonthsAgo.getMonth() - 6)

  const filtered = sorted.filter(r => {
    const t = new Date(r.recorded_at)
    return t >= sixMonthsAgo && t <= latest
  })

  if (!filtered.length) return {}

  const weekMap = new Map()
  for (const row of filtered) {
    const t = new Date(row.recorded_at)
    const w = Number(row.weight)
    if (!Number.isFinite(w)) continue

    const ws = getWeekStart(t)
    const key = ws.toISOString().slice(0, 10)
    if (!weekMap.has(key)) weekMap.set(key, { weekStart: ws, rows: [] })
    weekMap.get(key).rows.push({ t, w })
  }

  const weeks = Array.from(weekMap.values()).sort((a, b) => a.weekStart - b.weekStart)
  const x = []
  const kData = []
  const closeList = []

  for (const wk of weeks) {
    wk.rows.sort((a, b) => a.t - b.t)
    const weights = wk.rows.map(i => i.w)
    const O = wk.rows[0].w
    const C = wk.rows[wk.rows.length - 1].w
    const H = Math.max(...weights)
    const L = Math.min(...weights)

    x.push(`${wk.weekStart.getMonth() + 1}/${wk.weekStart.getDate()}`)
    kData.push([O, C, L, H])
    closeList.push(C)
  }

  const alpha = 2 / (4 + 1)
  const ema4 = closeList.map((c, i) => {
    if (i === 0) return Number(c.toFixed(3))
    return Number((alpha * c + (1 - alpha) * ema4[i - 1]).toFixed(3))
  })

  const weekStartDate = getWeekStart(sixMonthsAgo)
  const weekEndDate = latest

  const volumeSeriesPromise = fetchWeeklyCalorieData(weekStartDate, weekEndDate)

  return {
    tooltip: {
      trigger: 'axis',
      axisPointer: { type: 'cross' }
    },
    legend: {
      top: 0,
      data: ['周体重K线', 'EMA(4)', '周净热量差']
    },
    grid: [
      { left: '3%', right: '4%', top: '12%', height: '52%', containLabel: true },
      { left: '3%', right: '4%', top: '70%', height: '18%', containLabel: true }
    ],
    xAxis: [
      {
        type: 'category',
        data: x,
        boundaryGap: true,
        axisLine: { onZero: false },
        splitLine: { show: false },
        min: 'dataMin',
        max: 'dataMax',
        axisLabel: { show: false }
      },
      {
        type: 'category',
        gridIndex: 1,
        data: x,
        boundaryGap: true,
        axisLine: { onZero: false },
        splitLine: { show: false },
        min: 'dataMin',
        max: 'dataMax',
        axisLabel: { rotate: x.length > 10 ? 45 : 0 }
      }
    ],
    yAxis: [
      {
        type: 'value',
        scale: true,
        splitArea: { show: false }
      },
      {
        type: 'value',
        gridIndex: 1,
        scale: true,
        splitLine: { show: true }
      }
    ],
    dataZoom: [
      { type: 'inside', xAxisIndex: [0, 1], start: 0, end: 100 },
      { show: !isMobile.value, type: 'slider', xAxisIndex: [0, 1], bottom: 0, start: 0, end: 100 }
    ],
    series: [
      {
        name: '周体重K线',
        type: 'candlestick',
        data: kData,
        itemStyle: {
          color: '#ef4444',
          color0: '#10b981',
          borderColor: '#ef4444',
          borderColor0: '#10b981'
        }
      },
      {
        name: 'EMA(4)',
        type: 'line',
        data: ema4,
        smooth: true,
        symbol: 'none',
        lineStyle: { width: 2, color: '#6366f1' }
      },
      {
        name: '周净热量差',
        type: 'bar',
        xAxisIndex: 1,
        yAxisIndex: 1,
        data: x.map(() => 0),
        itemStyle: {
          color: (params) => (params.value >= 0 ? '#ef4444' : '#10b981')
        }
      }
    ]
  }
})

watch(currentUserId, () => {
  fetchWeightLogs()
})
onMounted(() => { updateIsMobile(); window.addEventListener('resize', updateIsMobile); fetchWeightLogs() })
onBeforeUnmount(() => window.removeEventListener('resize', updateIsMobile))
</script>

<style scoped>
.tracker-container { 
  max-width: 1400px; 
  margin: 0 auto; 
  padding: 24px; 
  box-sizing: border-box; 
  width: 100%;
}

.header-actions { 
  display: flex; 
  justify-content: space-between; 
  align-items: center; 
  margin-bottom: 24px; 
}

.header-actions h2 { margin: 0; font-size: 22px; font-weight: 600; color: #111827; }

.chart-card, .list-card { 
  border: none; 
  border-radius: 12px; 
  box-shadow: 0 1px 3px rgba(0,0,0,0.05); 
  width: 100%;
  box-sizing: border-box;
}

.card-header-flex { display: flex; justify-content: space-between; align-items: center; width: 100%; }
.title { font-weight: 600; font-size: 15px; color: #111827; }
.header-controls { display: flex; gap: 10px; align-items: center; flex-wrap: wrap; justify-content: flex-end; }

.predict-controls { 
  display: flex; 
  gap: 16px; 
  padding: 12px; 
  background: #f9fafb; 
  border-radius: 8px; 
  margin-bottom: 12px; 
  flex-wrap: wrap; 
}

.control-item { display: flex; align-items: center; gap: 8px; }
.label { font-size: 12px; color: #6b7280; }
.predict-tip { font-size: 12px; color: #9ca3af; margin-top: 8px; text-align: center; }

.pagination-container { margin-top: 20px; display: flex; justify-content: center; }
.mt-20 { margin-top: 20px; }

@media (max-width: 1200px) {
  .mobile-mt-20 { margin-top: 20px; }
}

@media (max-width: 768px) {
  .tracker-container { padding: 12px; }
  .header-actions h2 { font-size: 18px; }
  .predict-controls { gap: 10px; padding: 10px; }
  .header-controls { justify-content: center; width: 100%; margin-top: 10px; }
}
</style>
