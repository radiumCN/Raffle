<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import * as XLSX from 'xlsx'

const themeOptions = [
  {
    id: 'ai-lab',
    name: 'AI前沿实验室',
    style: '科技感、动态、未来风',
    colors: ['#0b1026', '#1b2a55', '#6cf7ff', '#a855f7'],
  },
  {
    id: 'trend-consumer',
    name: '潮流消费志',
    style: '时尚、活泼、视觉冲击',
    colors: ['#fff1f2', '#fecdd3', '#fb7185', '#f97316'],
  },
  {
    id: 'creative-studio',
    name: '创意工坊',
    style: '灵感、多元、轻快',
    colors: ['#fef9c3', '#fde68a', '#34d399', '#3b82f6'],
  },
  {
    id: 'future-explorer',
    name: '未来探索站',
    style: '科幻、冒险、简洁',
    colors: ['#0f172a', '#1e293b', '#94a3b8', '#38bdf8'],
  },
  {
    id: 'energy-life',
    name: '能量生活家',
    style: '健康、清新、积极',
    colors: ['#ecfccb', '#d9f99d', '#22c55e', '#facc15'],
  },
  {
    id: 'market-radar',
    name: '市场风向标',
    style: '专业、简洁、数据驱动',
    colors: ['#e0f2fe', '#bae6fd', '#2563eb', '#1e3a8a'],
  },
  {
    id: 'culture-next',
    name: '文化新次元',
    style: '国潮、融合、灵动',
    colors: ['#fff7ed', '#fdba74', '#f97316', '#7c3aed'],
  },
  {
    id: 'happy-community',
    name: '乐享生活圈',
    style: '温馨、欢快、社区感',
    colors: ['#fef2f2', '#fecaca', '#fb7185', '#facc15'],
  },
]

const viewOptions = [
  { id: 'home', name: '官网' },
  { id: 'screen', name: '大屏' },
]

const currentView = ref('home')
const showAdmin = ref(false)
const currentThemeId = ref('ai-lab')

const participants = ref([])
const prizes = ref([])
const winners = ref([])

const participantName = ref('')
const participantGroup = ref('')
const prizeName = ref('')
const prizeCount = ref(1)
const drawCount = ref(1)
const drawDuration = ref(1800)
const drawSpeed = ref(90)
const selectedPrizeId = ref('')
const isDrawing = ref(false)
const rollingName = ref('')
const currentBatchWinners = ref([])

const importMessage = ref('')
const exportMessage = ref('')

const storageKeys = {
  participants: 'raffle_participants',
  prizes: 'raffle_prizes',
  winners: 'raffle_winners',
  theme: 'raffle_theme',
}

const themeMap = computed(() => {
  const map = new Map()
  themeOptions.forEach((theme) => map.set(theme.id, theme))
  return map
})

const availableParticipants = computed(() => {
  const winnerIds = new Set(winners.value.map((item) => item.participantId))
  return participants.value.filter((item) => !winnerIds.has(item.id))
})

const prizeStats = computed(() => {
  const result = prizes.value.map((prize) => {
    const used = winners.value.filter((winner) => winner.prizeId === prize.id).length
    return {
      ...prize,
      used,
      remaining: Math.max(prize.count - used, 0),
    }
  })
  return result
})

const selectedPrize = computed(() => prizeStats.value.find((item) => item.id === selectedPrizeId.value))

const maxDrawCount = computed(() => {
  if (!selectedPrize.value) return 1
  const remaining = selectedPrize.value.remaining
  const available = availableParticipants.value.length
  return Math.max(1, Math.min(remaining, available))
})

const homeFeatures = [
  { title: '零网络请求', detail: '全程本地运行，数据仅保存在浏览器中。' },
  { title: '多格式导入导出', detail: '支持 Excel、CSV、JSON，一键批量导入。' },
  { title: '多主题切换', detail: '8 套主题即刻切换，适配不同活动场景。' },
  { title: '抽奖过程透明', detail: '实时展示奖品与中奖名单，过程清晰。' },
]

const formatTime = (time) => new Date(time).toLocaleString('zh-CN')

const generateId = () => `${Date.now()}-${Math.random().toString(16).slice(2)}`

const createAudioContext = () => {
  if (!window.AudioContext && !window.webkitAudioContext) return null
  return new (window.AudioContext || window.webkitAudioContext)()
}

const playBeep = () => {
  const context = createAudioContext()
  if (!context) return
  const oscillator = context.createOscillator()
  const gain = context.createGain()
  oscillator.connect(gain)
  gain.connect(context.destination)
  oscillator.frequency.value = 880
  gain.gain.value = 0.06
  oscillator.start()
  setTimeout(() => {
    oscillator.stop()
    context.close()
  }, 180)
}

const normalizeParticipant = (item, index) => {
  if (typeof item === 'string') {
    return { id: generateId(), name: item.trim(), group: '' }
  }
  const name = String(item.name ?? item.姓名 ?? item.nickname ?? item.名称 ?? '').trim()
  const group = String(item.group ?? item.分组 ?? item.department ?? item.部门 ?? '').trim()
  const idValue = item.id ?? item.ID ?? item.编号 ?? ''
  const id = String(idValue).trim() || `${generateId()}-${index}`
  if (!name) return null
  return { id, name, group }
}

const normalizePrize = (item, index) => {
  if (typeof item === 'string') {
    return { id: generateId(), name: item.trim(), count: 1 }
  }
  const name = String(item.name ?? item.奖品 ?? item.名称 ?? item.title ?? '').trim()
  const countValue = Number(item.count ?? item.数量 ?? item.total ?? 1)
  const count = Number.isFinite(countValue) && countValue > 0 ? Math.floor(countValue) : 1
  const idValue = item.id ?? item.ID ?? item.编号 ?? ''
  const id = String(idValue).trim() || `${generateId()}-${index}`
  if (!name) return null
  return { id, name, count }
}

const parseCsv = (text) => {
  const rows = text
    .split(/\r?\n/)
    .map((row) => row.trim())
    .filter((row) => row.length > 0)
  if (rows.length === 0) return []
  const splitRow = (row) => {
    const values = []
    let current = ''
    let inQuotes = false
    for (let i = 0; i < row.length; i += 1) {
      const char = row[i]
      if (char === '"') {
        if (inQuotes && row[i + 1] === '"') {
          current += '"'
          i += 1
        } else {
          inQuotes = !inQuotes
        }
      } else if (char === ',' && !inQuotes) {
        values.push(current)
        current = ''
      } else {
        current += char
      }
    }
    values.push(current)
    return values.map((value) => value.trim())
  }
  const header = splitRow(rows[0])
  const hasHeader = header.some((item) => /name|姓名|奖品|名称|count|数量|group|分组|id|编号/i.test(item))
  if (!hasHeader) {
    return rows.map((row) => ({ name: row }))
  }
  const keys = header
  return rows.slice(1).map((row) => {
    const values = splitRow(row)
    const record = {}
    keys.forEach((key, index) => {
      record[key] = values[index] ?? ''
    })
    return record
  })
}

const mergeUniqueById = (source, incoming) => {
  const map = new Map(source.map((item) => [item.id, item]))
  incoming.forEach((item) => {
    if (!map.has(item.id)) map.set(item.id, item)
  })
  return Array.from(map.values())
}

const handleImport = async (event, type) => {
  const file = event.target.files?.[0]
  if (!file) return
  importMessage.value = ''
  const extension = file.name.split('.').pop()?.toLowerCase() ?? ''
  try {
    let rawData = []
    if (extension === 'json') {
      const text = await file.text()
      const parsed = JSON.parse(text)
      rawData = Array.isArray(parsed) ? parsed : []
    } else if (extension === 'csv') {
      const text = await file.text()
      rawData = parseCsv(text)
    } else if (extension === 'xlsx' || extension === 'xls') {
      const buffer = await file.arrayBuffer()
      const workbook = XLSX.read(buffer)
      const sheet = workbook.Sheets[workbook.SheetNames[0]]
      rawData = XLSX.utils.sheet_to_json(sheet, { defval: '' })
    }

    if (type === 'participants') {
      const normalized = rawData
        .map((item, index) => normalizeParticipant(item, index))
        .filter(Boolean)
      participants.value = mergeUniqueById(participants.value, normalized)
      importMessage.value = `成功导入 ${normalized.length} 位抽奖人员。`
    } else {
      const normalized = rawData
        .map((item, index) => normalizePrize(item, index))
        .filter(Boolean)
      prizes.value = mergeUniqueById(prizes.value, normalized)
      importMessage.value = `成功导入 ${normalized.length} 个奖品。`
    }
  } catch (error) {
    importMessage.value = '导入失败，请检查文件格式。'
  }
  event.target.value = ''
}

const downloadBlob = (blob, filename) => {
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = filename
  link.click()
  URL.revokeObjectURL(url)
}

const exportJson = (data, filename) => {
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' })
  downloadBlob(blob, filename)
}

const exportCsv = (rows, headers, filename) => {
  const csvLines = [headers.join(',')]
  rows.forEach((row) => {
    const line = headers
      .map((key) => {
        const value = String(row[key] ?? '')
        return value.includes(',') || value.includes('"')
          ? `"${value.replace(/"/g, '""')}"`
          : value
      })
      .join(',')
    csvLines.push(line)
  })
  const blob = new Blob([csvLines.join('\n')], { type: 'text/csv' })
  downloadBlob(blob, filename)
}

const exportExcel = (rows, filename, sheetName) => {
  const worksheet = XLSX.utils.json_to_sheet(rows)
  const workbook = XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(workbook, worksheet, sheetName)
  XLSX.writeFile(workbook, filename)
}

const handleExport = (type, format) => {
  exportMessage.value = ''
  const data =
    type === 'participants' ? participants.value : type === 'prizes' ? prizes.value : winners.value
  if (data.length === 0) {
    exportMessage.value = '暂无数据可导出。'
    return
  }
  const filename = `${type}-${Date.now()}`
  if (format === 'json') {
    exportJson(data, `${filename}.json`)
  }
  if (format === 'csv') {
    const headers =
      type === 'participants'
        ? ['id', 'name', 'group']
        : type === 'prizes'
          ? ['id', 'name', 'count']
          : ['id', 'prizeName', 'participantName', 'participantGroup', 'time']
    exportCsv(data, headers, `${filename}.csv`)
  }
  if (format === 'xlsx') {
    const sheetName =
      type === 'participants' ? 'Participants' : type === 'prizes' ? 'Prizes' : 'Winners'
    exportExcel(data, `${filename}.xlsx`, sheetName)
  }
  exportMessage.value = '导出完成，请检查浏览器下载内容。'
}

const downloadTemplate = (type) => {
  const data =
    type === 'participants'
      ? [{ name: '张三', group: '技术部', id: '1001' }]
      : [{ name: '特等奖', count: 1, id: 'P001' }]
  const sheetName = type === 'participants' ? '人员模板' : '奖品模板'
  exportExcel(data, `${sheetName}.xlsx`, sheetName)
}

const addParticipant = () => {
  const name = participantName.value.trim()
  if (!name) return
  participants.value.push({
    id: generateId(),
    name,
    group: participantGroup.value.trim(),
  })
  participantName.value = ''
  participantGroup.value = ''
}

const addPrize = () => {
  const name = prizeName.value.trim()
  const countValue = Number(prizeCount.value)
  if (!name) return
  const count = Number.isFinite(countValue) && countValue > 0 ? Math.floor(countValue) : 1
  prizes.value.push({
    id: generateId(),
    name,
    count,
  })
  prizeName.value = ''
  prizeCount.value = 1
}

const removeParticipant = (id) => {
  participants.value = participants.value.filter((item) => item.id !== id)
  winners.value = winners.value.filter((item) => item.participantId !== id)
}

const removePrize = (id) => {
  prizes.value = prizes.value.filter((item) => item.id !== id)
  winners.value = winners.value.filter((item) => item.prizeId !== id)
  if (selectedPrizeId.value === id) selectedPrizeId.value = ''
}

const cleanDuplicateParticipants = () => {
  const map = new Map()
  const duplicated = []
  participants.value.forEach((person) => {
    const key = `${person.name}`.trim().toLowerCase() + `|${person.group}`.trim().toLowerCase()
    if (!map.has(key)) {
      map.set(key, person)
    } else {
      duplicated.push(person)
    }
  })
  if (duplicated.length === 0) {
    importMessage.value = '未发现重复人员。'
    return
  }
  participants.value = Array.from(map.values())
  const duplicatedIds = new Set(duplicated.map((item) => item.id))
  winners.value = winners.value.filter((item) => !duplicatedIds.has(item.participantId))
  importMessage.value = `已合并 ${duplicated.length} 位重复人员。`
}

const resetAllData = () => {
  participants.value = []
  prizes.value = []
  winners.value = []
  selectedPrizeId.value = ''
  localStorage.removeItem(storageKeys.participants)
  localStorage.removeItem(storageKeys.prizes)
  localStorage.removeItem(storageKeys.winners)
}

let drawAnimationFrame = null
let drawStartTime = null

const drawWinners = () => {
  importMessage.value = ''
  exportMessage.value = ''
  currentBatchWinners.value = [] // Reset batch winners before new draw
  if (isDrawing.value) return
  if (!selectedPrize.value) {
    importMessage.value = '请先选择奖品。'
    return
  }
  const remaining = selectedPrize.value.remaining
  if (remaining <= 0) {
    importMessage.value = '该奖品已抽完。'
    return
  }
  if (availableParticipants.value.length === 0) {
    importMessage.value = '暂无可抽奖人员。'
    return
  }
  const desired = Number(drawCount.value)
  const max = Math.min(remaining, availableParticipants.value.length)
  const count = Number.isFinite(desired) && desired > 0 ? Math.min(Math.floor(desired), max) : 1
  const pool = [...availableParticipants.value]
  
  // Shuffle pool
  for (let i = pool.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[pool[i], pool[j]] = [pool[j], pool[i]]
  }
  
  const chosen = pool.slice(0, Math.min(count, pool.length))
  const durationValue = Number(drawDuration.value)
  const duration = Number.isFinite(durationValue) ? Math.min(Math.max(durationValue, 600), 5000) : 1800
  
  isDrawing.value = true
  drawStartTime = performance.now()
  
  const animate = (time) => {
    const elapsed = time - drawStartTime
    
    if (elapsed < duration) {
      // Update rolling name every frame for smooth effect, or throttle if needed
      const person = pool[Math.floor(Math.random() * pool.length)]
      if (person) rollingName.value = person.name
      drawAnimationFrame = requestAnimationFrame(animate)
    } else {
      finishDraw(chosen)
    }
  }
  
  drawAnimationFrame = requestAnimationFrame(animate)
}

const finishDraw = (chosen) => {
  if (drawAnimationFrame) cancelAnimationFrame(drawAnimationFrame)
  isDrawing.value = false
  rollingName.value = ''
  const now = new Date().toISOString()
  
  chosen.forEach((person) => {
    winners.value.unshift({
      id: generateId(),
      prizeId: selectedPrize.value.id,
      prizeName: selectedPrize.value.name,
      participantId: person.id,
      participantName: person.name,
      participantGroup: person.group,
      time: now,
    })
  })
  playBeep()
}

const resetWinners = () => {
  winners.value = []
}

watch(currentThemeId, (value) => {
  document.documentElement.dataset.theme = value
  localStorage.setItem(storageKeys.theme, value)
})

let importMessageTimer = null
watch(importMessage, (newVal) => {
  if (importMessageTimer) clearTimeout(importMessageTimer)
  if (newVal) {
    importMessageTimer = setTimeout(() => {
      importMessage.value = ''
      importMessageTimer = null
    }, 3000)
  }
})

let exportMessageTimer = null
watch(exportMessage, (newVal) => {
  if (exportMessageTimer) clearTimeout(exportMessageTimer)
  if (newVal) {
    exportMessageTimer = setTimeout(() => {
      exportMessage.value = ''
      exportMessageTimer = null
    }, 3000)
  }
})

watch(
  participants,
  (value) => localStorage.setItem(storageKeys.participants, JSON.stringify(value)),
  { deep: true }
)

watch(
  prizes,
  (value) => localStorage.setItem(storageKeys.prizes, JSON.stringify(value)),
  { deep: true }
)

watch(
  winners,
  (value) => localStorage.setItem(storageKeys.winners, JSON.stringify(value)),
  { deep: true }
)

watch(
  prizeStats,
  (value) => {
    if (!selectedPrizeId.value && value.length > 0) {
      selectedPrizeId.value = value[0].id
    }
  },
  { immediate: true }
)

watch(maxDrawCount, (newMax) => {
  if (drawCount.value > newMax) {
    drawCount.value = newMax
  }
})

watch(drawCount, (newVal) => {
  if (newVal > maxDrawCount.value) {
    drawCount.value = maxDrawCount.value
  }
})

onMounted(() => {
  const storedTheme = localStorage.getItem(storageKeys.theme)
  if (storedTheme && themeMap.value.has(storedTheme)) currentThemeId.value = storedTheme

  try {
    const storedParticipants = localStorage.getItem(storageKeys.participants)
    if (storedParticipants) participants.value = JSON.parse(storedParticipants)
    const storedPrizes = localStorage.getItem(storageKeys.prizes)
    if (storedPrizes) prizes.value = JSON.parse(storedPrizes)
    const storedWinners = localStorage.getItem(storageKeys.winners)
    if (storedWinners) winners.value = JSON.parse(storedWinners)
  } catch (error) {
    participants.value = []
    prizes.value = []
    winners.value = []
  }

  document.documentElement.dataset.theme = currentThemeId.value
})
</script>

<template>
  <div class="app">
    <header class="topbar">
      <div class="topbar-inner">
        <div class="brand">
          <svg
            class="brand-logo"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5" />
          </svg>
          <div class="brand-text">
            <div class="brand-title">Raffle Studio</div>
            <div class="brand-sub">安全、离线、多主题抽奖平台</div>
          </div>
        </div>
        <nav class="nav">
          <button
            v-for="view in viewOptions"
            :key="view.id"
            class="nav-button"
            :class="{ active: currentView === view.id }"
            @click="currentView = view.id"
          >
            {{ view.name }}
          </button>
        </nav>
        <div class="topbar-actions" style="display: flex; align-items: center; gap: 16px;">
          <div class="theme-switch">
            <svg
              width="16"
              height="16"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <circle cx="12" cy="12" r="5" />
              <path d="M12 1v2M12 21v2M4.22 4.22l1.42 1.42M18.36 18.36l1.42 1.42M1 12h2M21 12h2M4.22 19.78l1.42-1.42M18.36 5.64l1.42-1.42" />
            </svg>
            <select v-model="currentThemeId">
              <option v-for="theme in themeOptions" :key="theme.id" :value="theme.id">
                {{ theme.name }}
              </option>
            </select>
          </div>
          <a href="https://github.com/radiumCN/Raffle" target="_blank" class="github-link" style="color: var(--text-muted); display: flex; align-items: center; transition: color 0.2s;">
            <svg height="24" viewBox="0 0 16 16" version="1.1" width="24" aria-hidden="true" fill="currentColor">
              <path fill-rule="evenodd" d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"></path>
            </svg>
          </a>
        </div>
      </div>
    </header>

    <main class="content">
      <section v-if="currentView === 'home'" class="home">
        <div class="hero">
          <div class="hero-text">
            <h1>一次搭建，覆盖官网与抽奖场景</h1>
            <p>
              无任何网络请求，数据只在本地浏览器内流转。支持 Excel/CSV/JSON 导入导出，
              多主题随场景切换，适配品牌活动、年会、发布会。
            </p>
            <div class="hero-actions">
              <button class="primary" @click="currentView = 'screen'">立即进入大屏</button>
              <button class="ghost" @click="showAdmin = true">进入后台配置</button>
            </div>
          </div>
          <div class="hero-card">
            <div class="stat">
              <div class="stat-value">{{ participants.length }}</div>
              <div class="stat-label">已导入人员</div>
            </div>
            <div class="stat">
              <div class="stat-value">{{ prizes.length }}</div>
              <div class="stat-label">奖品配置</div>
            </div>
            <div class="stat">
              <div class="stat-value">{{ winners.length }}</div>
              <div class="stat-label">已产生中奖</div>
            </div>
          </div>
        </div>

        <div class="section">
          <h2>核心优势</h2>
          <div class="feature-grid">
            <div v-for="feature in homeFeatures" :key="feature.title" class="feature-card">
              <h3>{{ feature.title }}</h3>
              <p>{{ feature.detail }}</p>
            </div>
          </div>
        </div>

        <div class="section">
          <h2>主题示例</h2>
          <div class="theme-grid">
            <div v-for="theme in themeOptions" :key="theme.id" class="theme-card">
              <div class="theme-swatch">
                <span v-for="color in theme.colors" :key="color" :style="{ background: color }"></span>
              </div>
              <div class="theme-info">
                <h3>{{ theme.name }}</h3>
                <p>{{ theme.style }}</p>
              </div>
              <button class="ghost" @click="currentThemeId = theme.id">应用主题</button>
            </div>
          </div>
        </div>
      </section>

      <section v-else class="screen">
        <div class="screen-actions">
           <button class="icon-btn" @click="showAdmin = true" title="设置">
             <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="3"></circle><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path></svg>
           </button>
        </div>
        <div class="screen-container">
          <header class="screen-header">
            <h1 class="screen-title">Raffle Studio 抽奖现场</h1>
            <div class="screen-stats">
              <div class="stat-item">
                <span class="stat-num">{{ availableParticipants.length }}</span>
                <span class="stat-desc">待抽奖</span>
              </div>
              <div class="stat-item">
                <span class="stat-num">{{ prizeStats.reduce((sum, item) => sum + item.remaining, 0) }}</span>
                <span class="stat-desc">剩余奖品</span>
              </div>
            </div>
          </header>
          
          <main class="screen-main">
             <div class="prize-selector">
                <div 
                  v-for="prize in prizeStats" 
                  :key="prize.id" 
                  class="prize-card"
                  :class="{ active: selectedPrizeId === prize.id, disabled: prize.remaining === 0 }"
                  @click="selectedPrizeId = prize.id"
                >
                  <div class="prize-name">{{ prize.name }}</div>
                  <div class="prize-count">剩余 {{ prize.remaining }} / {{ prize.count }}</div>
                </div>
             </div>
             
             <div class="draw-area" :class="{ 'is-drawing': isDrawing }">
                <div class="draw-content">
                   <div class="draw-display">
                      <div class="current-prize" v-if="selectedPrize">
                     {{ isDrawing ? '正在抽取：' : '当前奖品：' }}{{ selectedPrize.name }}
                  </div>
                  
                  <div v-if="isDrawing" class="rolling-text">
                     {{ rollingName }}
                  </div>
                  
                  <div v-else class="rolling-text">
                     Ready
                  </div>
                   </div>
                   
                   <div class="draw-controls">
                      <div class="control-group">
                         <label>抽取人数</label>
                         <input v-model.number="drawCount" type="number" min="1" :max="maxDrawCount" :disabled="isDrawing" />
                      </div>
                      <button class="draw-btn" @click="drawWinners" :disabled="isDrawing || !selectedPrize || selectedPrize.remaining === 0">
                         {{ isDrawing ? '抽奖中...' : '开始抽奖' }}
                      </button>
                   </div>
                </div>
             </div>
          </main>
          
          <aside class="screen-winners" :style="{ visibility: winners.length > 0 ? 'visible' : 'hidden' }">
             <h3>最新中奖</h3>
             <div class="winner-ticker">
                <div v-if="winners.length === 0" class="ticker-item">
                   <span class="w-name">&nbsp;</span>
                   <span class="w-prize">&nbsp;</span>
                </div>
                <div v-for="winner in winners.slice(0, 12)" :key="winner.id" class="ticker-item">
                   <span class="w-name">{{ winner.participantName }}</span>
                   <span class="w-prize">{{ winner.prizeName }}</span>
                </div>
             </div>
          </aside>
        </div>
        
        <div v-if="isDrawing" class="draw-overlay">
          <div class="draw-card">
            <div class="draw-title">正在抽取 {{ selectedPrize?.name }}</div>
            <div class="draw-name">{{ rollingName }}</div>
          </div>
        </div>
      </section>
      <div v-if="showAdmin" class="admin-modal">
        <div class="admin-modal-content">
          <button class="close-btn" @click="showAdmin = false">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
          </button>
          <div class="raffle-grid">
            <div class="panel">
              <h2>后台配置</h2>
              <div class="panel-section">
                <h3>人员管理</h3>
                <div class="form-row">
                  <input v-model="participantName" placeholder="姓名" />
                  <input v-model="participantGroup" placeholder="分组/部门" />
                  <button class="primary" @click="addParticipant">添加人员</button>
                </div>
                <div class="file-row">
                  <label class="file-label">
                    <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
                      <polyline points="17 8 12 3 7 8"/>
                      <line x1="12" y1="3" x2="12" y2="15"/>
                    </svg>
                    导入人员
                    <input
                      type="file"
                      accept=".xlsx,.xls,.csv,.json"
                      @change="(e) => handleImport(e, 'participants')"
                    />
                  </label>
                  <div class="export-group">
                    <button class="ghost" @click="downloadTemplate('participants')">下载模板</button>
                  </div>
                </div>
                <div class="action-row">
                  <button class="ghost" @click="cleanDuplicateParticipants">
                    去重人员
                  </button>
                  <button class="ghost" @click="resetAllData">清空全部数据</button>
                </div>
                <div class="list">
                  <div v-for="person in participants" :key="person.id" class="list-item">
                    <div>
                      <strong>{{ person.name }}</strong>
                      <span v-if="person.group">{{ person.group }}</span>
                    </div>
                    <button class="ghost" @click="removeParticipant(person.id)">移除</button>
                  </div>
                  <div v-if="participants.length === 0" class="empty">暂无人员数据</div>
                </div>
              </div>
  
              <div class="panel-section">
                <h3>奖品管理</h3>
                <div class="form-row">
                  <input v-model="prizeName" placeholder="奖品名称" />
                  <input v-model.number="prizeCount" type="number" min="1" placeholder="数量" />
                  <button class="primary" @click="addPrize">添加奖品</button>
                </div>
                <div class="file-row">
                  <label class="file-label">
                    <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
                      <polyline points="17 8 12 3 7 8"/>
                      <line x1="12" y1="3" x2="12" y2="15"/>
                    </svg>
                    导入奖品
                    <input
                      type="file"
                      accept=".xlsx,.xls,.csv,.json"
                      @change="(e) => handleImport(e, 'prizes')"
                    />
                  </label>
                  <div class="export-group">
                    <button class="ghost" @click="downloadTemplate('prizes')">下载模板</button>
                  </div>
                </div>
                <div class="list">
                  <div v-for="prize in prizeStats" :key="prize.id" class="list-item">
                    <div>
                      <strong>{{ prize.name }}</strong>
                      <span>总数 {{ prize.count }}，剩余 {{ prize.remaining }}</span>
                    </div>
                    <button class="ghost" @click="removePrize(prize.id)">移除</button>
                  </div>
                  <div v-if="prizeStats.length === 0" class="empty">暂无奖品数据</div>
                </div>
              </div>
            </div>
  
            <div class="panel">
              <h2>中奖数据管理</h2>
              <div class="panel-section">
                <h3>参数设置</h3>
                <div class="form-row">
                  <div class="form-group">
                    <label>默认抽奖人数</label>
                    <input v-model.number="drawCount" type="number" min="1" />
                  </div>
                  <div class="form-group">
                    <label>动画时长 (ms)</label>
                    <input v-model.number="drawDuration" type="number" min="600" max="5000" />
                  </div>
                  <div class="form-group">
                    <label>滚动速度 (ms)</label>
                    <input v-model.number="drawSpeed" type="number" min="50" max="200" />
                  </div>
                </div>
              </div>
              
              <div class="panel-section">
                <h3>中奖名单</h3>
                <div class="export-group">
                  <button class="ghost" @click="resetWinners">清空中奖结果</button>
                  <button class="ghost" @click="handleExport('winners', 'xlsx')">导出 Excel</button>
                  <button class="ghost" @click="handleExport('winners', 'csv')">导出 CSV</button>
                  <button class="ghost" @click="handleExport('winners', 'json')">导出 JSON</button>
                </div>
                <div class="list winners">
                  <div v-for="winner in winners" :key="winner.id" class="list-item">
                    <div>
                      <strong>{{ winner.participantName }}</strong>
                      <span>{{ winner.prizeName }}</span>
                      <span v-if="winner.participantGroup">{{ winner.participantGroup }}</span>
                    </div>
                    <div class="time">{{ formatTime(winner.time) }}</div>
                  </div>
                  <div v-if="winners.length === 0" class="empty">暂无中奖记录</div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div v-if="importMessage" class="toast">{{ importMessage }}</div>
      <div v-if="exportMessage" class="toast">{{ exportMessage }}</div>
    </main>
  </div>
</template>
