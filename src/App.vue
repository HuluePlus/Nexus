<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'
import {
  Activity,
  AlertTriangle,
  ArrowDownRight,
  ArrowUpRight,
  Check,
  ChevronDown,
  Clock3,
  Database,
  Gauge,
  Menu,
  Moon,
  Pause,
  Play,
  RefreshCcw,
  Search,
  ShieldCheck,
  SunMedium,
  Terminal,
  Wifi,
  Zap,
} from 'lucide-vue-next'

import CyberButton from './components/cyber-ui/CyberButton.vue'
import CyberIconButton from './components/cyber-ui/CyberIconButton.vue'
import CyberPanel from './components/cyber-ui/CyberPanel.vue'
import CyberPanelHeader from './components/cyber-ui/CyberPanelHeader.vue'
import CyberStatusBadge from './components/cyber-ui/CyberStatusBadge.vue'
import CyberStatusDot from './components/cyber-ui/CyberStatusDot.vue'
import CyberTextField from './components/cyber-ui/CyberTextField.vue'

const STATUS_API = 'https://status.input.im/api/status'
const emptyModel = {
  id: 'awaiting-telemetry',
  name: '等待数据',
  provider: 'STATUS.INPUT.IM',
  family: '等待实时数据流',
  status: 'degraded',
  statusLabel: '等待中',
  latency: '--',
  latencyMs: null,
  uptime: '--',
  uptimeValue: 0,
  samples: [],
  history: [],
  trend: [0, 0],
  requests: '--',
}

const providerByModel = {
  'gpt-5.6-sol': 'OPENAI',
  'gpt-5.6-terra': 'OPENAI',
  'gpt-5.6-luna': 'OPENAI',
  'gpt-5.5': 'OPENAI',
  'gpt-5.4': 'OPENAI',
  'gpt-5.4-mini': 'OPENAI',
}

const familyByModel = {
  'gpt-5.6-sol': '推理 / 前沿模型',
  'gpt-5.6-terra': '通用 / 旗舰模型',
  'gpt-5.6-luna': '创作 / 旗舰模型',
  'gpt-5.5': '通用 / 上一代模型',
  'gpt-5.4': '推理 / 旗舰模型',
  'gpt-5.4-mini': '推理 / 轻量模型',
}

function slugify(value) {
  return value.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')
}

function formatLatency(value) {
  if (value === null || value === undefined) return '--'
  return value >= 1000 ? `${(value / 1000).toFixed(2)}s` : `${Math.round(value)}ms`
}

function formatApiTime(timestamp) {
  if (!timestamp) return '--'
  return new Date(timestamp * 1000).toLocaleTimeString('en-GB', { hour: '2-digit', minute: '2-digit', second: '2-digit' })
}

function parseErrorCode(error) {
  const match = error?.match(/HTTP (\d+)/)
  return match ? `HTTP-${match[1]}` : 'ERR-API'
}

function mapService(service) {
  const history = Array.isArray(service.history) ? service.history : []
  let lastKnownLatency = service.last?.latency_ms ?? 0
  const trend = history.map((sample) => {
    if (sample.latency_ms !== null && sample.latency_ms !== undefined) lastKnownLatency = sample.latency_ms
    return lastKnownLatency
  })
  const status = service.last?.ok ? 'operational' : 'degraded'

  return {
    id: slugify(service.model),
    name: service.model,
    provider: providerByModel[service.model] || 'UNKNOWN',
    family: familyByModel[service.model] || '实时模型接口',
    status,
    statusLabel: status === 'operational' ? '在线' : '失败',
    latency: formatLatency(service.last?.latency_ms),
    latencyMs: service.last?.latency_ms ?? null,
    uptime: `${Number(service.uptime_pct || 0).toFixed(2)}%`,
    uptimeValue: Number(service.uptime_pct || 0),
    samples: history.slice(-36),
    history,
    trend: trend.length > 1 ? trend : [lastKnownLatency, lastKnownLatency],
    requests: `${history.length} 条`,
  }
}

const models = ref([])
const selectedId = ref('')
const activeFilter = ref('all')
const searchQuery = ref('')
const sidebarOpen = ref(false)
const isRefreshing = ref(false)
const isPaused = ref(false)
const incidentAcknowledged = ref(false)
const loadingStatus = ref(true)
const liveError = ref('')
const apiAllOk = ref(null)
const theme = ref('cyan')
const now = ref(new Date())
const lastSync = ref('--')
let clockTimer
let syncTimer

const selectedModel = computed(() => models.value.find((model) => model.id === selectedId.value) || models.value[0] || emptyModel)
const onlineCount = computed(() => models.value.filter((model) => model.status === 'operational').length)
const degradedCount = computed(() => models.value.filter((model) => model.status === 'degraded').length)
const activeIncidentCount = computed(() => models.value.filter((model) => model.status !== 'operational').length)
const averageUptime = computed(() => {
  if (!models.value.length) return '--'
  return `${(models.value.reduce((sum, model) => sum + model.uptimeValue, 0) / models.value.length).toFixed(2)}%`
})
const medianLatency = computed(() => {
  const values = models.value.map((model) => model.latencyMs).filter((value) => value !== null).sort((a, b) => a - b)
  if (!values.length) return '--'
  return formatLatency(values[Math.floor(values.length / 2)])
})
const p95Latency = computed(() => {
  const values = models.value.flatMap((model) => model.history.map((sample) => sample.latency_ms).filter((value) => value !== null)).sort((a, b) => a - b)
  if (!values.length) return '--'
  return formatLatency(values[Math.min(values.length - 1, Math.floor(values.length * 0.95))])
})
const liveStatusLabel = computed(() => {
  if (loadingStatus.value) return '同步中'
  if (liveError.value) return '数据异常'
  return apiAllOk.value ? '全部正常' : '需要关注'
})
const liveStatusTone = computed(() => (apiAllOk.value ? 'success' : 'danger'))
const filteredModels = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  return models.value.filter((model) => {
    const matchesQuery = !query || `${model.name} ${model.provider} ${model.family}`.toLowerCase().includes(query)
    const matchesFilter = activeFilter.value === 'all' || (activeFilter.value === 'online' ? model.status === 'operational' : model.status === activeFilter.value)
    return matchesQuery && matchesFilter
  })
})
const chartPoints = computed(() => {
  const values = selectedModel.value.trend
  const max = Math.max(...values)
  const min = Math.min(...values)
  const range = Math.max(max - min, 1)
  return values.map((value, index) => `${(index / Math.max(values.length - 1, 1)) * 440},${106 - ((value - min) / range) * 76}`).join(' ')
})
const chartLastPoint = computed(() => {
  const values = selectedModel.value.trend
  const max = Math.max(...values)
  const min = Math.min(...values)
  const range = Math.max(max - min, 1)
  return { x: 440, y: 106 - ((values[values.length - 1] - min) / range) * 76 }
})
const selectedDelta = computed(() => {
  const values = selectedModel.value.history.map((sample) => sample.latency_ms).filter((value) => value !== null)
  if (values.length < 2 || !values[values.length - 2]) return '--'
  const delta = ((values[values.length - 1] - values[values.length - 2]) / values[values.length - 2]) * 100
  return `${delta >= 0 ? '+' : ''}${delta.toFixed(1)}% 较上次采样`
})
const pulseBars = computed(() => {
  const histories = models.value.map((model) => model.history.map((sample) => sample.latency_ms))
  const length = Math.max(...histories.map((history) => history.length), 0)
  if (!length) return [20, 20, 20, 20, 20, 20]
  const values = Array.from({ length }, (_, index) => {
    const samples = histories.map((history) => history[index]).filter((value) => value !== null && value !== undefined)
    return samples.length ? samples.reduce((sum, value) => sum + value, 0) / samples.length : 0
  })
  const max = Math.max(...values, 1)
  return values.slice(-30).map((value) => Math.max(8, (value / max) * 100))
})
const liveEvents = computed(() => models.value
  .flatMap((model) => model.history.filter((sample) => !sample.ok).map((sample) => ({
    time: formatApiTime(sample.ts),
    type: 'danger',
    title: sample.error?.includes('429') ? '上游限流' : '探针失败',
    detail: `${model.name} · ${sample.error || '未知上游错误'}`,
    code: parseErrorCode(sample.error),
  })))
  .sort((a, b) => b.time.localeCompare(a.time))
  .slice(0, 4))
const formattedTime = computed(() => now.value.toLocaleTimeString('en-GB', { hour: '2-digit', minute: '2-digit', second: '2-digit' }))
const statusTone = (status) => status === 'operational' ? 'success' : status === 'degraded' || status === 'offline' ? 'danger' : 'accent'
const statusText = (status) => status === 'operational' ? '正常' : status === 'degraded' ? '异常' : '离线'
const sampleTooltip = (sample) => {
  const state = sample.ok ? '正常' : '失败'
  const latency = sample.latency_ms === null || sample.latency_ms === undefined ? '--' : formatLatency(sample.latency_ms)
  const error = sample.error ? `\n错误：${sample.error}` : ''
  return `${state} · ${formatApiTime(sample.ts)}\n延迟：${latency}${error}`
}

function selectModel(model) {
  selectedId.value = model.id
}

function setFilter(filter) {
  activeFilter.value = filter
}

function applyTheme(nextTheme) {
  theme.value = nextTheme
  document.documentElement.classList.remove('cyan', 'magenta')
  document.documentElement.classList.add(nextTheme)
  document.documentElement.dataset.cyberTheme = nextTheme
}

function toggleTheme() {
  applyTheme(theme.value === 'cyan' ? 'magenta' : 'cyan')
}

async function refreshData() {
  if (isRefreshing.value) return
  isRefreshing.value = true
  try {
    const response = await fetch(STATUS_API, { cache: 'no-store' })
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    const payload = await response.json()
    const nextModels = Array.isArray(payload.services) ? payload.services.map(mapService) : []
    models.value = nextModels
    apiAllOk.value = Boolean(payload.all_ok)
    lastSync.value = formatApiTime(payload.generated_at)
    liveError.value = ''
    if (!nextModels.some((model) => model.id === selectedId.value)) selectedId.value = nextModels[0]?.id || ''
  } catch (error) {
    liveError.value = error instanceof Error ? error.message : '无法读取状态接口'
  } finally {
    loadingStatus.value = false
    isRefreshing.value = false
  }
}

function runProbe() {
  refreshData()
}

onMounted(() => {
  applyTheme('cyan')
  refreshData()
  clockTimer = window.setInterval(() => {
    now.value = new Date(now.value.getTime() + 1000)
  }, 1000)
  syncTimer = window.setInterval(() => {
    if (!isPaused.value) refreshData()
  }, 5000)
})

onUnmounted(() => {
  window.clearInterval(clockTimer)
  window.clearInterval(syncTimer)
})
</script>

<template>
  <main class="app-shell" :class="{ 'sidebar-is-open': sidebarOpen }">
    <aside class="sidebar">
      <div class="brand-lockup">
        <div class="brand-glyph"><img src="/nexus-icon.svg" alt="" /></div>
        <div>
          <strong>NEXUS</strong>
          <span>模型监控</span>
        </div>
      </div>

      <div class="sidebar-identity">
        <span class="identity-dot" />
        <div>
          <span>私有实例</span>
          <strong>edge-lab / 01</strong>
        </div>
        <ChevronDown :size="14" />
      </div>

      <div class="nav-label">当前视图 <span>LIVE</span></div>
      <div class="sidebar-current"><Activity :size="16" /><span>模型状态总览</span><CyberStatusDot tone="success" /></div>

      <div class="sidebar-spacer" />

      <div class="sidebar-stream">
        <div class="stream-heading"><Wifi :size="14" /> 数据流频道</div>
        <div class="stream-status"><CyberStatusDot tone="success" /><span>遥测已连接</span><strong>实时</strong></div>
        <div class="stream-wave" aria-hidden="true">
          <span v-for="bar in 24" :key="bar" :style="{ height: `${18 + ((bar * 17) % 58)}%` }" />
        </div>
        <small>最近数据包 <b>{{ lastSync }}</b></small>
      </div>

      <div class="sidebar-footer">
        <div class="operator-chip"><span>OP</span><div><strong>操作员</strong><small>仅本地</small></div></div>
      </div>
    </aside>

    <section class="workspace">
      <header class="topbar">
        <CyberIconButton label="打开导航" class="mobile-menu" @click="sidebarOpen = true"><Menu :size="18" /></CyberIconButton>
        <div class="breadcrumbs"><span>NEXUS</span><i>/</i><strong>状态总览</strong><em>节点 01</em></div>
        <div class="topbar-actions">
          <div class="topbar-clock"><Clock3 :size="14" /><span>本地 // {{ formattedTime }}</span></div>
          <CyberIconButton label="切换主题色" @click="toggleTheme"><SunMedium v-if="theme === 'cyan'" :size="16" /><Moon v-else :size="16" /></CyberIconButton>
        </div>
      </header>

      <div class="content-scroll">
        <section class="page-intro">
          <div>
            <div class="eyebrow-line"><span /> 可观测网格 <b>v2.6.4</b></div>
            <h1>模型状态中心</h1>
            <p>集中查看所有模型节点、实时探针和异常信号，保持你的私有模型集群始终可见。</p>
            <div class="live-feed-note"><span class="live-feed-pip" :class="{ 'live-feed-pip--error': liveError }" /> 数据源 <b>status.input.im/api/status</b><span>·</span> 轮询 5 秒 <span>·</span> {{ liveError ? liveError : loadingStatus ? '连接中...' : `已更新 ${lastSync}` }}</div>
          </div>
          <div class="health-orbit">
            <div class="orbit-core"><CyberStatusDot :tone="liveStatusTone" /><span>{{ liveStatusLabel }}</span><strong>{{ onlineCount }}/{{ models.length }}</strong></div>
            <div class="orbit-ring ring-one" /><div class="orbit-ring ring-two" />
          </div>
        </section>

        <section class="stat-grid" aria-label="节点概览">
          <CyberPanel variant="metric" class="stat-card stat-card--cyan">
            <div class="stat-top"><span>节点健康</span><Activity :size="16" /></div>
            <div class="stat-value">{{ onlineCount }}<small>/{{ models.length }}</small></div>
            <div class="stat-footer"><CyberStatusDot tone="success" /> 在线节点 <span>实时</span></div>
          </CyberPanel>
          <CyberPanel variant="metric" class="stat-card">
            <div class="stat-top"><span>可用率 / 30 天</span><ShieldCheck :size="16" /></div>
            <div class="stat-value">{{ averageUptime }}<small v-if="averageUptime !== '--'">平均</small></div>
            <div class="stat-footer"><ArrowUpRight :size="13" /> 来自实时状态接口</div>
          </CyberPanel>
          <CyberPanel variant="metric" class="stat-card">
            <div class="stat-top"><span>延迟中位数</span><Gauge :size="16" /></div>
            <div class="stat-value">{{ medianLatency }}<small v-if="medianLatency !== '--'">中位</small></div>
            <div class="stat-footer stat-footer--warning"><Zap :size="13" /> p95：{{ p95Latency }}</div>
          </CyberPanel>
          <CyberPanel variant="metric" class="stat-card stat-card--danger">
            <div class="stat-top"><span>活跃事件</span><AlertTriangle :size="16" /></div>
            <div class="stat-value">{{ Math.max(activeIncidentCount - (incidentAcknowledged ? 1 : 0), 0) }}<small> 个</small></div>
            <div class="stat-footer stat-footer--danger"><CyberStatusDot tone="danger" /> {{ degradedCount }} 个失败探针</div>
          </CyberPanel>
        </section>

        <section class="control-bar">
          <CyberTextField id="model-filter" v-model="searchQuery" label="" placeholder="筛选模型...">
            <template #icon><Search :size="16" /></template>
            <template #suffix>{{ filteredModels.length }} 个节点</template>
          </CyberTextField>
          <div class="filter-chips" role="group" aria-label="筛选模型状态">
            <button type="button" :class="{ active: activeFilter === 'all' }" @click="setFilter('all')">全部 <span>{{ models.length }}</span></button>
            <button type="button" :class="{ active: activeFilter === 'online' }" @click="setFilter('online')">在线 <span>{{ onlineCount }}</span></button>
            <button type="button" :class="{ active: activeFilter === 'degraded' }" @click="setFilter('degraded')">异常 <span>{{ degradedCount }}</span></button>
          </div>
          <CyberButton variant="ghost" :loading="isRefreshing" loading-label="同步中" @click="refreshData">
            <template #icon><RefreshCcw :size="15" :class="{ 'spin-icon': isRefreshing }" /></template>
            立即同步
          </CyberButton>
        </section>

        <section class="dashboard-grid">
          <CyberPanel class="fleet-panel">
            <CyberPanelHeader>
              <span><span class="header-signal" /> 实时模型矩阵</span>
              <span class="panel-meta">60 分钟窗口 <b>·</b> 探测间隔 60 秒</span>
            </CyberPanelHeader>
            <div class="matrix-heading"><span>模型 / 提供方</span><span>状态</span><span>延迟</span><span>可用率</span><span>历史信号</span></div>
            <div class="model-list">
              <button v-for="model in filteredModels" :key="model.id" type="button" class="model-row" :class="{ selected: selectedId === model.id }" @click="selectModel(model)">
                <div class="model-name"><span class="provider-mark" :class="`provider-mark--${model.provider.toLowerCase()}`">{{ model.provider.slice(0, 2) }}</span><div><strong>{{ model.name }}</strong><small>{{ model.provider }} · {{ model.family }}</small></div></div>
                <CyberStatusBadge><CyberStatusDot :tone="statusTone(model.status)" /> {{ statusText(model.status) }}</CyberStatusBadge>
                <strong class="row-metric" :class="{ 'row-metric--muted': model.status === 'offline' }">{{ model.latency }}</strong>
                <span class="row-uptime">{{ model.uptime }}</span>
                <div class="sample-strip" :aria-label="`${model.name} 历史采样`"><span v-for="(sample, index) in model.samples" :key="`${model.id}-${index}`" :class="`sample-${sample.ok ? 'up' : 'down'}`" :data-tooltip="sampleTooltip(sample)" :title="sampleTooltip(sample)" /></div>
                <ArrowUpRight :size="14" class="row-arrow" />
              </button>
              <div v-if="!filteredModels.length" class="empty-state"><Search :size="18" /> 当前没有匹配的节点</div>
            </div>
            <div class="panel-foot"><span><span class="legend-dot legend-dot--up" /> 正常响应</span><span><span class="legend-dot legend-dot--warn" /> 延迟提醒</span><span><span class="legend-dot legend-dot--down" /> 失败采样</span><span class="foot-spacer" /> 最近同步 <b>{{ lastSync }}</b></div>
          </CyberPanel>

          <CyberPanel class="detail-panel">
            <CyberPanelHeader>
              <span><span class="header-signal header-signal--magenta" /> 当前节点</span>
              <span class="panel-meta">信号 / 24 小时</span>
            </CyberPanelHeader>
            <div class="detail-content">
              <div class="detail-title-row"><div><div class="detail-kicker">{{ selectedModel.provider }} // {{ selectedModel.id }}</div><h2>{{ selectedModel.name }}</h2><p>{{ selectedModel.family }}</p></div><CyberStatusBadge><CyberStatusDot :tone="statusTone(selectedModel.status)" /> {{ statusText(selectedModel.status) }}</CyberStatusBadge></div>
              <div class="detail-metrics"><div><span>最近延迟</span><strong>{{ selectedModel.latency }}</strong></div><div><span>可用率</span><strong>{{ selectedModel.uptime }}</strong></div><div><span>历史采样</span><strong>{{ selectedModel.requests }}</strong></div></div>
              <div class="chart-wrap"><div class="chart-heading"><span>响应时间 / 毫秒</span><strong><ArrowDownRight :size="13" /> {{ selectedDelta }}</strong></div><svg class="signal-chart" viewBox="0 0 460 130" role="img" aria-label="当前模型响应时间趋势图"><path class="chart-grid" d="M0 14H460 M0 44H460 M0 74H460 M0 104H460 M0 129H460 M0 0V130 M88 0V130 M176 0V130 M264 0V130 M352 0V130 M440 0V130" /><polyline :points="chartPoints" /><circle :cx="chartLastPoint.x" :cy="chartLastPoint.y" r="4" /><circle :cx="chartLastPoint.x" :cy="chartLastPoint.y" r="9" class="chart-pulse" /></svg><div class="chart-axis"><span>00:00</span><span>06:00</span><span>12:00</span><span>18:00</span><span>现在</span></div></div>
              <div class="detail-actions"><CyberButton variant="primary" :loading="isRefreshing" loading-label="探测中" @click="runProbe"><template #icon><Play :size="15" /></template>立即探测</CyberButton><CyberButton variant="secondary" @click="isPaused = !isPaused"><template #icon><Pause v-if="!isPaused" :size="15" /><Play v-else :size="15" /></template>{{ isPaused ? '恢复监控' : '暂停监控' }}</CyberButton></div>
            </div>
          </CyberPanel>
        </section>

        <section class="lower-grid">
          <CyberPanel class="pulse-panel">
            <CyberPanelHeader><span><span class="header-signal" /> 延迟脉冲</span><span class="panel-meta">全节点平均</span></CyberPanelHeader>
            <div class="pulse-body"><div class="pulse-summary"><div><strong>{{ medianLatency }}</strong><small>实时延迟中位数</small></div><div class="pulse-delta"><Database :size="14" /> <span>{{ models.length }} 个节点<small>{{ models.length ? '实时采样池' : '等待数据流' }}</small></span></div></div><div class="bar-chart"><div v-for="(bar, index) in pulseBars" :key="index" class="bar-column"><span :style="{ height: `${bar}%` }" /></div></div><div class="bar-axis"><span>-60 分钟</span><span>-45 分钟</span><span>-30 分钟</span><span>-15 分钟</span><span>现在</span></div></div>
          </CyberPanel>

          <CyberPanel class="events-panel">
            <CyberPanelHeader><span><span class="header-signal header-signal--warning" /> 信号日志</span><span class="panel-meta">实时</span></CyberPanelHeader>
            <div class="event-list">
              <div v-for="event in liveEvents" :key="`${event.code}-${event.time}-${event.detail}`" class="event-row" :class="`event-row--${event.type}`"><span class="event-time">{{ event.time }}</span><span class="event-icon"><AlertTriangle :size="13" /></span><div><strong>{{ event.title }}</strong><small>{{ event.detail }}</small></div><code>{{ event.code }}</code></div>
              <div v-if="!liveEvents.length" class="event-empty"><Check :size="15" /> 当前数据流没有失败采样</div>
            </div>
            <div class="events-footer"><button type="button" @click="incidentAcknowledged = !incidentAcknowledged"><Check :size="14" /> {{ incidentAcknowledged ? '事件已确认' : '确认最新事件' }}</button></div>
          </CyberPanel>

          <CyberPanel class="regions-panel">
            <CyberPanelHeader><span><span class="header-signal header-signal--magenta" /> 数据源契约</span><span class="panel-meta">实时 JSON</span></CyberPanelHeader>
            <div class="source-contract"><div><span>接口</span><strong>/api/status</strong></div><div><span>模型数量</span><strong>{{ models.length }} 个</strong></div><div><span>每节点采样</span><strong>{{ models[0]?.history.length || '--' }} 条</strong></div><div><span>访问方式</span><strong>跨域：*</strong></div></div>
            <div class="region-footer"><Database :size="14" /><span>{{ liveError ? liveError : '直连数据流 · 无模拟数据' }}</span><ShieldCheck :size="14" /></div>
          </CyberPanel>
        </section>

        <footer class="workspace-footer"><span><span class="footer-mark" /> NEXUS 可观测网格</span><span>私有实例 // 不同步云端</span><span>构建 026.08</span></footer>
      </div>
    </section>

    <div v-if="sidebarOpen" class="sidebar-scrim" @click="sidebarOpen = false" />
  </main>
</template>
