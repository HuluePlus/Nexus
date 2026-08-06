<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { Badge, Button, Card, Input, Tag } from '@hulueplus/cyber-ui'
import {
  Activity,
  AlertTriangle,
  ArrowDownRight,
  ArrowUpRight,
  Bell,
  BellOff,
  BellRing,
  Check,
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
  Wifi,
  X,
  Zap,
} from 'lucide-vue-next'

const STATUS_API = 'https://status.input.im/api/status'
const WATCHED_MODEL = 'gpt-5.6-sol'
const ACKNOWLEDGED_EVENTS_KEY = 'nexus-acknowledged-events'
const NOTIFICATIONS_ENABLED_KEY = 'nexus-notifications-enabled'
const THEME_KEY = 'nexus-theme'
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

function formatDuration(seconds) {
  if (!Number.isFinite(seconds) || seconds <= 0) return '不足 1 分钟'
  const minutes = Math.max(1, Math.round(seconds / 60))
  if (minutes < 60) return `${minutes} 分钟`
  const hours = Math.floor(minutes / 60)
  const remainingMinutes = minutes % 60
  return remainingMinutes ? `${hours} 小时 ${remainingMinutes} 分钟` : `${hours} 小时`
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
const acknowledgedEventIds = ref(new Set())
const liveError = ref('')
const apiAllOk = ref(null)
const theme = ref('cyan')
const now = ref(new Date())
const lastSync = ref('--')
const notificationsEnabled = ref(false)
const notificationPermission = ref('default')
const notificationRequestPending = ref(false)
const statusNotice = ref(null)
let clockTimer
let syncTimer
let noticeTimer
let lastWatchedModelAvailability = null

const selectedModel = computed(() => models.value.find((model) => model.id === selectedId.value) || models.value[0] || emptyModel)
const onlineCount = computed(() => models.value.filter((model) => model.status === 'operational').length)
const degradedCount = computed(() => models.value.filter((model) => model.status === 'degraded').length)
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
  if (isRefreshing.value) return '同步中'
  if (isPaused.value) return '监控已暂停'
  if (liveError.value) return '数据异常'
  return apiAllOk.value ? '全部正常' : '需要关注'
})
const liveStatusTone = computed(() => {
  if (isRefreshing.value) return 'processing'
  if (isPaused.value) return 'warning'
  if (liveError.value) return 'error'
  return apiAllOk.value ? 'success' : 'warning'
})
const streamStatusText = computed(() => {
  if (isRefreshing.value) return '正在同步'
  if (isPaused.value) return '自动轮询已暂停'
  if (liveError.value) return '数据流连接异常'
  return '遥测已连接'
})
const streamStatusFlag = computed(() => {
  if (isRefreshing.value) return '同步'
  if (isPaused.value) return '暂停'
  if (liveError.value) return '异常'
  return '实时'
})
const liveFeedMessage = computed(() => {
  if (isRefreshing.value) return '同步中...'
  if (isPaused.value) return `已暂停 · 最后更新 ${lastSync.value}`
  if (liveError.value) return liveError.value
  return `已更新 ${lastSync.value}`
})
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
const recentEvents = computed(() => models.value
  .flatMap((model) => model.history.filter((sample) => !sample.ok).map((sample) => ({
    id: `${model.id}:${sample.ts}`,
    timestamp: sample.ts,
    time: formatApiTime(sample.ts),
    type: 'danger',
    title: sample.error?.includes('429') ? '上游限流' : '探针失败',
    detail: `${model.name} · ${sample.error || '未知上游错误'}`,
    code: parseErrorCode(sample.error),
    acknowledged: acknowledgedEventIds.value.has(`${model.id}:${sample.ts}`),
  })))
  .sort((a, b) => b.timestamp - a.timestamp))
const liveEvents = computed(() => recentEvents.value.slice(0, 4))
const unacknowledgedEventCount = computed(() => recentEvents.value.filter((event) => !event.acknowledged).length)
const formattedTime = computed(() => now.value.toLocaleTimeString('en-GB', { hour: '2-digit', minute: '2-digit', second: '2-digit' }))
const notificationStateText = computed(() => {
  if (notificationsEnabled.value) return '已开启'
  if (notificationPermission.value === 'unsupported') return '当前浏览器不可用'
  if (notificationPermission.value === 'denied') return '权限已被阻止'
  return '已关闭'
})
const statusTone = (status) => status === 'operational' ? 'success' : status === 'degraded' || status === 'offline' ? 'error' : 'processing'
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

watch(filteredModels, (nextModels) => {
  if (!nextModels.some((model) => model.id === selectedId.value)) selectedId.value = nextModels[0]?.id || ''
})

function applyTheme(nextTheme) {
  const normalizedTheme = nextTheme === 'magenta' ? 'magenta' : 'cyan'
  theme.value = normalizedTheme
  document.documentElement.classList.remove('cyan', 'magenta')
  document.documentElement.classList.add(normalizedTheme)
  document.documentElement.dataset.cyberTheme = normalizedTheme
  window.localStorage.setItem(THEME_KEY, normalizedTheme)
}

function toggleTheme() {
  applyTheme(theme.value === 'cyan' ? 'magenta' : 'cyan')
}

function showStatusNotice(title, detail, tone = 'info') {
  window.clearTimeout(noticeTimer)
  statusNotice.value = { title, detail, tone }
  noticeTimer = window.setTimeout(() => {
    statusNotice.value = null
  }, 6500)
}

function closeStatusNotice() {
  window.clearTimeout(noticeTimer)
  statusNotice.value = null
}

function persistNotificationsEnabled(enabled) {
  notificationsEnabled.value = enabled
  window.localStorage.setItem(NOTIFICATIONS_ENABLED_KEY, String(enabled))
}

function syncNotificationPermission() {
  if (!('Notification' in window) || !window.isSecureContext) {
    notificationPermission.value = 'unsupported'
    if (notificationsEnabled.value) persistNotificationsEnabled(false)
    return
  }

  notificationPermission.value = window.Notification.permission
  if (notificationsEnabled.value && notificationPermission.value !== 'granted') {
    persistNotificationsEnabled(false)
  }
}

async function toggleNotifications() {
  if (notificationsEnabled.value) {
    persistNotificationsEnabled(false)
    showStatusNotice('状态通知已关闭', `${WATCHED_MODEL} 状态变化时不再发送通知。`)
    return
  }

  syncNotificationPermission()
  if (notificationPermission.value === 'unsupported') {
    showStatusNotice('无法开启系统通知', '请使用支持通知且通过 HTTPS 访问的浏览器。', 'warning')
    return
  }
  if (notificationPermission.value === 'denied') {
    showStatusNotice('通知权限已被阻止', '请在浏览器的网站权限中允许通知后重试。', 'warning')
    return
  }

  notificationRequestPending.value = true
  try {
    if (window.Notification.permission === 'default') {
      notificationPermission.value = await window.Notification.requestPermission()
    }
    if (notificationPermission.value !== 'granted') {
      showStatusNotice('未开启状态通知', '浏览器没有授予通知权限。', 'warning')
      return
    }
    persistNotificationsEnabled(true)
    showStatusNotice('状态通知已开启', `${WATCHED_MODEL} 可用状态变化时会在这里和系统通知中提醒。`, 'success')
  } catch {
    showStatusNotice('无法开启系统通知', '浏览器通知权限请求失败，请稍后重试。', 'warning')
  } finally {
    notificationRequestPending.value = false
  }
}

function announceWatchedModelAvailability(model) {
  if (!notificationsEnabled.value) return

  const isAvailable = model.status === 'operational'
  const title = isAvailable ? `${WATCHED_MODEL} 已恢复可用` : `${WATCHED_MODEL} 已不可用`
  const previousDuration = getPreviousAvailabilityDuration(model)
  const durationDetail = previousDuration === null
    ? ''
    : isAvailable
      ? `此前异常持续 ${formatDuration(previousDuration)}。`
      : `此前正常持续 ${formatDuration(previousDuration)}。`
  const detail = isAvailable
    ? `${durationDetail}服务已恢复正常，当前延迟 ${model.latency}。`
    : `${durationDetail}最新探针请求失败，请打开状态面板查看详情。`
  const tone = isAvailable ? 'success' : 'danger'
  showStatusNotice(title, detail, tone)

  syncNotificationPermission()
  if (!notificationsEnabled.value) return

  try {
    const notification = new window.Notification(title, {
      body: detail,
      icon: `${window.location.origin}/nexus-icon.svg`,
      tag: `nexus-${WATCHED_MODEL}-availability`,
      renotify: true,
    })
    notification.onclick = () => {
      window.focus()
      notification.close()
    }
  } catch {
    // The in-page notice remains available if the OS notification bridge fails.
  }
}

function getPreviousAvailabilityDuration(model) {
  const history = model.history
  if (!Array.isArray(history) || history.length < 2) return null

  const currentIndex = history.length - 1
  const currentSample = history[currentIndex]
  const previousIndex = currentIndex - 1
  const previousAvailable = Boolean(history[previousIndex]?.ok)
  if (previousAvailable === Boolean(currentSample?.ok)) return null

  let startIndex = previousIndex
  while (startIndex > 0 && Boolean(history[startIndex - 1]?.ok) === previousAvailable) startIndex -= 1

  const currentTimestamp = Number(currentSample?.ts)
  const startTimestamp = Number(history[startIndex]?.ts)
  if (!Number.isFinite(currentTimestamp) || !Number.isFinite(startTimestamp)) return null
  return Math.max(0, currentTimestamp - startTimestamp)
}

function trackWatchedModelAvailability(nextModels) {
  const watchedModel = nextModels.find((model) => model.name === WATCHED_MODEL)
  if (!watchedModel) return

  const isAvailable = watchedModel.status === 'operational'
  if (lastWatchedModelAvailability === null) {
    lastWatchedModelAvailability = isAvailable
    return
  }
  if (lastWatchedModelAvailability === isAvailable) return

  lastWatchedModelAvailability = isAvailable
  announceWatchedModelAvailability(watchedModel)
}

async function refreshData() {
  if (isRefreshing.value) return
  isRefreshing.value = true
  try {
    const response = await fetch(STATUS_API, { cache: 'no-store' })
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    const payload = await response.json()
    const nextModels = Array.isArray(payload.services) ? payload.services.map(mapService) : []
    trackWatchedModelAvailability(nextModels)
    models.value = nextModels
    apiAllOk.value = Boolean(payload.all_ok)
    lastSync.value = formatApiTime(payload.generated_at)
    liveError.value = ''
    if (!nextModels.some((model) => model.id === selectedId.value)) selectedId.value = nextModels[0]?.id || ''
  } catch (error) {
    liveError.value = error instanceof Error ? error.message : '无法读取状态接口'
  } finally {
    isRefreshing.value = false
  }
}

function toggleMonitoring() {
  isPaused.value = !isPaused.value
  if (!isPaused.value) refreshData()
}

function acknowledgeRecentEvents() {
  const nextIds = new Set(acknowledgedEventIds.value)
  recentEvents.value.forEach((event) => nextIds.add(event.id))
  acknowledgedEventIds.value = nextIds
  window.localStorage.setItem(ACKNOWLEDGED_EVENTS_KEY, JSON.stringify([...nextIds].slice(-500)))
}

onMounted(() => {
  try {
    const storedEventIds = JSON.parse(window.localStorage.getItem(ACKNOWLEDGED_EVENTS_KEY) || '[]')
    if (Array.isArray(storedEventIds)) acknowledgedEventIds.value = new Set(storedEventIds.filter((id) => typeof id === 'string'))
  } catch {
    window.localStorage.removeItem(ACKNOWLEDGED_EVENTS_KEY)
  }
  syncNotificationPermission()
  if (window.localStorage.getItem(NOTIFICATIONS_ENABLED_KEY) === 'true' && notificationPermission.value === 'granted') {
    notificationsEnabled.value = true
  }
  applyTheme(window.localStorage.getItem(THEME_KEY) || 'cyan')
  refreshData()
  window.addEventListener('focus', syncNotificationPermission)
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
  window.clearTimeout(noticeTimer)
  window.removeEventListener('focus', syncNotificationPermission)
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
      </div>

      <div class="nav-label">当前视图 <span>LIVE</span></div>
      <div class="sidebar-current"><Activity :size="16" /><span>模型状态总览</span><Badge :status="liveStatusTone" /></div>

      <div class="sidebar-spacer" />

      <div class="sidebar-stream">
        <div class="stream-heading"><Wifi :size="14" /> 数据流频道</div>
        <div class="stream-status" :class="{ 'stream-status--paused': isPaused, 'stream-status--error': liveError && !isPaused }"><Badge :status="liveStatusTone" /><span>{{ streamStatusText }}</span><strong>{{ streamStatusFlag }}</strong></div>
        <div class="stream-wave" aria-hidden="true">
          <span v-for="bar in 24" :key="bar" :style="{ height: `${18 + ((bar * 17) % 58)}%` }" />
        </div>
        <small>最近数据包 <b>{{ lastSync }}</b></small>
        <div class="notification-control">
          <div class="notification-control__label">
            <BellRing :size="14" />
            <span><strong>状态通知</strong><small>{{ notificationStateText }}</small></span>
          </div>
          <button
            type="button"
            class="notification-switch"
            :class="{ 'notification-switch--active': notificationsEnabled }"
            role="switch"
            :aria-checked="notificationsEnabled"
            :aria-label="notificationsEnabled ? '关闭 gpt-5.6-sol 状态通知' : '开启 gpt-5.6-sol 状态通知'"
            :disabled="notificationRequestPending"
            @click="toggleNotifications"
          >
            <span><Bell v-if="notificationsEnabled" :size="11" /><BellOff v-else :size="11" /></span>
          </button>
        </div>
      </div>

      <div class="sidebar-footer">
        <div class="operator-chip"><span>OP</span><div><strong>操作员</strong><small>仅本地</small></div></div>
      </div>
    </aside>

    <section class="workspace">
      <header class="topbar">
        <Button type="text" shape="circle" title="打开导航" aria-label="打开导航" class="icon-button mobile-menu" @click="sidebarOpen = true"><Menu :size="18" /></Button>
        <div class="breadcrumbs"><span>NEXUS</span><i>/</i><strong>状态总览</strong><em>节点 01</em></div>
        <div class="topbar-actions">
          <div class="topbar-clock"><Clock3 :size="14" /><span>本地 // {{ formattedTime }}</span></div>
          <Button type="text" shape="circle" title="切换主题色" aria-label="切换主题色" class="icon-button" @click="toggleTheme"><SunMedium v-if="theme === 'cyan'" :size="16" /><Moon v-else :size="16" /></Button>
        </div>
      </header>

      <div class="content-scroll">
        <section class="page-intro">
          <div>
            <div class="eyebrow-line"><span /> 可观测网格 <b>v2.6.4</b></div>
            <h1>模型状态中心</h1>
            <p>集中查看所有模型节点、实时探针和异常信号，保持你的私有模型集群始终可见。</p>
            <div class="live-feed-note"><span class="live-feed-pip" :class="{ 'live-feed-pip--error': liveError, 'live-feed-pip--paused': isPaused }" /> 数据源 <b>status.input.im/api/status</b><span>·</span> 轮询 5 秒 <span>·</span> {{ liveFeedMessage }}</div>
          </div>
          <div class="health-orbit">
            <div class="orbit-core"><Badge :status="liveStatusTone" /><span>{{ liveStatusLabel }}</span><strong>{{ onlineCount }}/{{ models.length }}</strong></div>
            <div class="orbit-ring ring-one" /><div class="orbit-ring ring-two" />
          </div>
        </section>

        <section class="stat-grid" aria-label="节点概览">
          <Card class="stat-card stat-card--cyan">
            <div class="stat-top"><span>节点健康</span><Activity :size="16" /></div>
            <div class="stat-value">{{ onlineCount }}<small>/{{ models.length }}</small></div>
            <div class="stat-footer"><Badge :status="liveStatusTone" /> 在线节点 <span>{{ isRefreshing ? '同步中' : isPaused ? '已暂停' : liveError ? '异常' : '实时' }}</span></div>
          </Card>
          <Card class="stat-card">
            <div class="stat-top"><span>可用率 / 30 天</span><ShieldCheck :size="16" /></div>
            <div class="stat-value">{{ averageUptime }}<small v-if="averageUptime !== '--'">平均</small></div>
            <div class="stat-footer"><ArrowUpRight :size="13" /> 来自实时状态接口</div>
          </Card>
          <Card class="stat-card">
            <div class="stat-top"><span>延迟中位数</span><Gauge :size="16" /></div>
            <div class="stat-value">{{ medianLatency }}<small v-if="medianLatency !== '--'">中位</small></div>
            <div class="stat-footer stat-footer--warning"><Zap :size="13" /> p95：{{ p95Latency }}</div>
          </Card>
          <Card class="stat-card stat-card--danger">
            <div class="stat-top"><span>未确认事件</span><AlertTriangle :size="16" /></div>
            <div class="stat-value">{{ unacknowledgedEventCount }}<small> 个</small></div>
            <div class="stat-footer stat-footer--danger"><Badge status="error" /> 最近 60 分钟</div>
          </Card>
        </section>

        <section class="control-bar">
          <Input id="model-filter" v-model:value="searchQuery" allow-clear class="model-filter" placeholder="筛选模型...">
            <template #prefix><Search :size="16" /></template>
            <template #suffix>{{ filteredModels.length }} 个节点</template>
          </Input>
          <div class="filter-chips" role="group" aria-label="筛选模型状态">
            <button type="button" :aria-pressed="activeFilter === 'all'" :class="{ active: activeFilter === 'all' }" @click="setFilter('all')">全部 <span>{{ models.length }}</span></button>
            <button type="button" :aria-pressed="activeFilter === 'online'" :class="{ active: activeFilter === 'online' }" @click="setFilter('online')">在线 <span>{{ onlineCount }}</span></button>
            <button type="button" :aria-pressed="activeFilter === 'degraded'" :class="{ active: activeFilter === 'degraded' }" @click="setFilter('degraded')">异常 <span>{{ degradedCount }}</span></button>
          </div>
          <Button type="text" class="sync-button" :loading="isRefreshing" @click="refreshData">
            <template #icon><RefreshCcw :size="15" :class="{ 'spin-icon': isRefreshing }" /></template>
            {{ isRefreshing ? '同步中' : '立即同步' }}
          </Button>
        </section>

        <section class="dashboard-grid" :class="{ 'dashboard-grid--single': !filteredModels.length }">
          <Card class="fleet-panel">
            <template #title>
              <span><span class="header-signal" /> 实时模型矩阵</span>
              <span class="panel-meta">60 分钟窗口 <b>·</b> 采样间隔 60 秒 <b v-if="isPaused">· 已暂停</b></span>
            </template>
            <div class="matrix-heading"><span>模型 / 提供方</span><span>状态</span><span>延迟</span><span>可用率</span><span>历史信号</span></div>
            <div class="model-list">
              <button v-for="model in filteredModels" :key="model.id" type="button" class="model-row" :aria-pressed="selectedId === model.id" :class="{ selected: selectedId === model.id }" @click="selectModel(model)">
                <div class="model-name"><span class="provider-mark" :class="`provider-mark--${model.provider.toLowerCase()}`">{{ model.provider.slice(0, 2) }}</span><div><strong>{{ model.name }}</strong><small>{{ model.provider }} · {{ model.family }}</small></div></div>
                <Tag class="status-badge"><Badge :status="statusTone(model.status)" /> {{ statusText(model.status) }}</Tag>
                <strong class="row-metric" :class="{ 'row-metric--muted': model.status === 'offline' }">{{ model.latency }}</strong>
                <span class="row-uptime">{{ model.uptime }}</span>
                <div class="sample-strip" :aria-label="`${model.name} 历史采样`"><span v-for="(sample, index) in model.samples" :key="`${model.id}-${index}`" :class="`sample-${sample.ok ? 'up' : 'down'}`" :data-tooltip="sampleTooltip(sample)" :title="sampleTooltip(sample)" /></div>
                <ArrowUpRight :size="14" class="row-arrow" />
              </button>
              <div v-if="!filteredModels.length" class="empty-state"><Search :size="18" /> 当前没有匹配的节点</div>
            </div>
            <div class="panel-foot"><span><span class="legend-dot legend-dot--up" /> 正常响应</span><span><span class="legend-dot legend-dot--warn" /> 延迟提醒</span><span><span class="legend-dot legend-dot--down" /> 失败采样</span><span class="foot-spacer" /> 最近同步 <b>{{ lastSync }}</b></div>
          </Card>

          <Card v-if="filteredModels.length" class="detail-panel">
            <template #title>
              <span><span class="header-signal header-signal--magenta" /> 当前节点</span>
              <span class="panel-meta">信号 / 60 分钟</span>
            </template>
            <div class="detail-content">
              <div class="detail-title-row"><div><div class="detail-kicker">{{ selectedModel.provider }} // {{ selectedModel.id }}</div><h2>{{ selectedModel.name }}</h2><p>{{ selectedModel.family }}</p></div><Tag class="status-badge"><Badge :status="statusTone(selectedModel.status)" /> {{ statusText(selectedModel.status) }}</Tag></div>
              <div class="detail-metrics"><div><span>最近延迟</span><strong>{{ selectedModel.latency }}</strong></div><div><span>可用率</span><strong>{{ selectedModel.uptime }}</strong></div><div><span>历史采样</span><strong>{{ selectedModel.requests }}</strong></div></div>
              <div class="chart-wrap"><div class="chart-heading"><span>响应时间 / 毫秒</span><strong><ArrowDownRight :size="13" /> {{ selectedDelta }}</strong></div><svg class="signal-chart" viewBox="0 0 460 130" role="img" aria-label="当前模型响应时间趋势图"><path class="chart-grid" d="M0 14H460 M0 44H460 M0 74H460 M0 104H460 M0 129H460 M0 0V130 M88 0V130 M176 0V130 M264 0V130 M352 0V130 M440 0V130" /><polyline :points="chartPoints" /><circle :cx="chartLastPoint.x" :cy="chartLastPoint.y" r="4" /><circle :cx="chartLastPoint.x" :cy="chartLastPoint.y" r="9" class="chart-pulse" /></svg><div class="chart-axis"><span>00:00</span><span>06:00</span><span>12:00</span><span>18:00</span><span>现在</span></div></div>
              <div class="detail-actions"><Button class="secondary-action" :aria-pressed="isPaused" @click="toggleMonitoring"><template #icon><Pause v-if="!isPaused" :size="15" /><Play v-else :size="15" /></template>{{ isPaused ? '恢复监控' : '暂停监控' }}</Button></div>
            </div>
          </Card>
        </section>

        <section class="lower-grid">
          <Card class="pulse-panel">
            <template #title><span><span class="header-signal" /> 延迟脉冲</span><span class="panel-meta">全节点平均</span></template>
            <div class="pulse-body"><div class="pulse-summary"><div><strong>{{ medianLatency }}</strong><small>实时延迟中位数</small></div><div class="pulse-delta"><Database :size="14" /> <span>{{ models.length }} 个节点<small>{{ models.length ? '实时采样池' : '等待数据流' }}</small></span></div></div><div class="bar-chart"><div v-for="(bar, index) in pulseBars" :key="index" class="bar-column"><span :style="{ height: `${bar}%` }" /></div></div><div class="bar-axis"><span>-60 分钟</span><span>-45 分钟</span><span>-30 分钟</span><span>-15 分钟</span><span>现在</span></div></div>
          </Card>

          <Card class="events-panel">
            <template #title><span><span class="header-signal header-signal--warning" /> 信号日志</span><span class="panel-meta">{{ unacknowledgedEventCount }} 个未确认</span></template>
            <div class="event-list">
              <div v-for="event in liveEvents" :key="event.id" class="event-row" :class="[`event-row--${event.type}`, { 'event-row--acknowledged': event.acknowledged }]"><span class="event-time">{{ event.time }}</span><span class="event-icon"><Check v-if="event.acknowledged" :size="13" /><AlertTriangle v-else :size="13" /></span><div><strong>{{ event.title }}</strong><small>{{ event.detail }}</small></div><code>{{ event.acknowledged ? 'ACK' : event.code }}</code></div>
              <div v-if="!liveEvents.length" class="event-empty"><Check :size="15" /> 当前数据流没有失败采样</div>
            </div>
            <div v-if="recentEvents.length" class="events-footer"><Button type="text" class="ack-button" :disabled="unacknowledgedEventCount === 0" @click="acknowledgeRecentEvents"><template #icon><Check :size="14" /></template>{{ unacknowledgedEventCount ? `确认全部事件 (${unacknowledgedEventCount})` : '当前事件已确认' }}</Button></div>
          </Card>

          <Card class="regions-panel">
            <template #title><span><span class="header-signal header-signal--magenta" /> 数据源契约</span><span class="panel-meta">{{ isPaused ? '缓存 JSON' : '实时 JSON' }}</span></template>
            <div class="source-contract"><div><span>接口</span><strong>/api/status</strong></div><div><span>模型数量</span><strong>{{ models.length }} 个</strong></div><div><span>每节点采样</span><strong>{{ models[0]?.history.length || '--' }} 条</strong></div><div><span>访问方式</span><strong>跨域：*</strong></div></div>
            <div class="region-footer"><Database :size="14" /><span>{{ isPaused ? '自动轮询已暂停 · 可手动同步' : liveError ? liveError : '直连数据流 · 无模拟数据' }}</span><ShieldCheck :size="14" /></div>
          </Card>
        </section>

        <footer class="workspace-footer"><span><span class="footer-mark" /> NEXUS 可观测网格</span><span>私有实例 // 不同步云端</span><span>构建 026.08</span></footer>
      </div>
    </section>

    <button v-if="sidebarOpen" type="button" class="sidebar-scrim" aria-label="关闭导航" @click="sidebarOpen = false" />

    <Transition name="status-notice">
      <div v-if="statusNotice" class="status-notice" :class="`status-notice--${statusNotice.tone}`" role="status" aria-live="polite">
        <span class="status-notice__icon"><BellRing :size="17" /></span>
        <div><strong>{{ statusNotice.title }}</strong><p>{{ statusNotice.detail }}</p></div>
        <Button type="text" shape="circle" title="关闭通知" aria-label="关闭通知" class="status-notice__close" @click="closeStatusNotice"><X :size="15" /></Button>
      </div>
    </Transition>
  </main>
</template>
