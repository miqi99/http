<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'

const props = defineProps({ persistConfig: Boolean })
const emit = defineEmits(['changeTab'])

const WS_STORAGE_KEY = 'ws-client-config'

const wsUrl = ref('')
const wsProtocol = ref('wss')
const wsStatus = ref('disconnected')
const wsMessages = ref([])
const wsInput = ref('')
const wsInstance = ref(null)
const wsLogRef = ref(null)
const initialized = ref(false)

const WS_STATUS = {
  disconnected: { text: '未连接',   textClass: 'text-slate-400',   dotClass: 'bg-slate-300' },
  connecting:   { text: '连接中',   textClass: 'text-amber-500',   dotClass: 'bg-amber-400 animate-pulse' },
  connected:    { text: '已连接',   textClass: 'text-emerald-600', dotClass: 'bg-emerald-400' },
  error:        { text: '连接失败', textClass: 'text-red-500',     dotClass: 'bg-red-400' },
}

const wsSt = computed(() => WS_STATUS[wsStatus.value])
const wsIsConnected = computed(() => wsStatus.value === 'connected')
const wsCanConnect = computed(() => wsStatus.value === 'disconnected' || wsStatus.value === 'error')

function normalizeWsUrl(raw) {
  const trimmed = raw.trim()
  if (trimmed.startsWith('wss://')) return trimmed.slice(6)
  if (trimmed.startsWith('ws://')) return trimmed.slice(5)
  return trimmed
}

watch(wsUrl, (newVal) => {
  const cleaned = normalizeWsUrl(newVal)
  if (cleaned !== newVal) wsUrl.value = cleaned
})

onMounted(() => {
  const saved = localStorage.getItem(WS_STORAGE_KEY)
  if (saved) {
    try {
      const config = JSON.parse(saved)
      wsUrl.value = config.wsUrl || ''
      wsProtocol.value = config.wsProtocol || 'wss'
    } catch {
      localStorage.removeItem(WS_STORAGE_KEY)
    }
  }
  initialized.value = true
})

watch(
  [() => props.persistConfig, wsUrl, wsProtocol],
  () => {
    if (!initialized.value) return
    if (!props.persistConfig) { localStorage.removeItem(WS_STORAGE_KEY); return }
    localStorage.setItem(WS_STORAGE_KEY, JSON.stringify({
      wsUrl: wsUrl.value,
      wsProtocol: wsProtocol.value,
    }))
  }
)

function wsAddMsg(type, content) {
  wsMessages.value.push({ type, content, time: new Date().toLocaleTimeString() })
  nextTick(() => {
    if (wsLogRef.value) wsLogRef.value.scrollTop = wsLogRef.value.scrollHeight
  })
}

function wsConnect() {
  const raw = wsUrl.value.trim()
  if (!raw) return
  wsStatus.value = 'connecting'
  const fullUrl = `${wsProtocol.value}://${raw}`
  wsAddMsg('system', `正在连接 ${fullUrl}`)
  try {
    const ws = new WebSocket(fullUrl)
    wsInstance.value = ws
    ws.onopen    = ()  => { wsStatus.value = 'connected'; wsAddMsg('system', '连接成功') }
    ws.onmessage = (e) => wsAddMsg('received', e.data)
    ws.onerror   = ()  => { wsStatus.value = 'error'; wsAddMsg('system', '连接出错') }
    ws.onclose   = (e) => {
      if (wsStatus.value !== 'error') wsStatus.value = 'disconnected'
      wsAddMsg('system', `连接已关闭 (${e.code})`)
      wsInstance.value = null
    }
  } catch (err) {
    wsStatus.value = 'error'
    wsAddMsg('system', err.message)
    wsInstance.value = null
  }
}

function wsDisconnect() { wsInstance.value?.close() }

function wsSend() {
  const msg = wsInput.value.trim()
  if (!msg || !wsIsConnected.value) return
  wsInstance.value.send(msg)
  wsAddMsg('sent', msg)
  wsInput.value = ''
}

function wsHandleKeydown(e) {
  if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); wsSend() }
}

function clearWs() {
  wsInstance.value?.close()
  wsUrl.value = ''
  wsProtocol.value = 'wss'
  wsStatus.value = 'disconnected'
  wsInput.value = ''
  wsMessages.value = []
  wsInstance.value = null
}
</script>

<template>
  <div class="mx-auto flex h-full max-w-[1440px] gap-4 px-4 py-4">

    <!-- 左栏：消息记录 -->
    <div class="flex h-full w-[420px] shrink-0 flex-col">
      <div class="flex min-h-0 flex-1 flex-col rounded-2xl bg-white p-4 shadow-sm ring-1 ring-slate-200">

        <div class="mb-3 flex shrink-0 items-center justify-between gap-2">
          <div class="flex items-center gap-2">
            <span class="text-sm font-semibold">消息记录</span>
            <span class="flex items-center gap-1 text-xs" :class="wsSt.textClass">
              <span class="h-1.5 w-1.5 rounded-full" :class="wsSt.dotClass"></span>
              {{ wsSt.text }}
            </span>
          </div>
          <button
            type="button"
            @click="wsMessages = []"
            class="rounded-lg border border-slate-200 px-2.5 py-1 text-xs text-slate-500 hover:bg-slate-100 transition"
          >清空</button>
        </div>

        <div
          ref="wsLogRef"
          class="min-h-0 flex-1 overflow-y-auto rounded-xl border border-slate-200 bg-slate-950 p-3 space-y-2"
        >
          <div v-if="wsMessages.length === 0" class="p-2 text-xs text-slate-500">暂无消息</div>
          <div v-for="(msg, i) in wsMessages" :key="i">
            <div v-if="msg.type === 'system'" class="flex items-center gap-1.5 text-xs text-slate-500">
              <span>●</span>
              <span class="flex-1">{{ msg.content }}</span>
              <span class="shrink-0 text-slate-600">{{ msg.time }}</span>
            </div>
            <div v-else-if="msg.type === 'sent'" class="flex flex-col items-end gap-0.5">
              <div class="max-w-[90%] break-all rounded-lg rounded-tr-sm bg-blue-600 px-3 py-1.5 font-mono text-xs text-white">{{ msg.content }}</div>
              <span class="text-[10px] text-slate-500">{{ msg.time }} ↑</span>
            </div>
            <div v-else class="flex flex-col items-start gap-0.5">
              <div class="max-w-[90%] break-all rounded-lg rounded-tl-sm bg-slate-800 px-3 py-1.5 font-mono text-xs text-slate-100">{{ msg.content }}</div>
              <span class="text-[10px] text-slate-500">↓ {{ msg.time }}</span>
            </div>
          </div>
        </div>

      </div>
    </div>

    <!-- 右栏：连接配置 + 发送 -->
    <div class="flex-1 overflow-y-auto">
      <div class="rounded-2xl bg-white shadow-sm ring-1 ring-slate-200">

        <!-- Tab 行 -->
        <div class="flex items-center justify-between border-b border-slate-100 px-4 py-2.5">
          <div class="flex items-center gap-1">
            <button
              type="button"
              @click="emit('changeTab', 'http')"
              class="rounded-lg px-3 py-1.5 text-xs font-semibold text-slate-500 hover:text-slate-800 hover:bg-slate-50 transition"
            >HTTP</button>
            <button
              type="button"
              class="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white"
            >WebSocket</button>
          </div>
          <button
            type="button"
            @click="clearWs"
            class="rounded-lg bg-white px-3 py-1.5 text-xs font-semibold text-red-500 ring-1 ring-red-100 hover:bg-red-50 transition"
          >一键清空</button>
        </div>

        <!-- 表单内容 -->
        <div class="p-4">

          <!-- 地址 -->
          <div class="mb-4">
            <label class="mb-2 block text-sm font-medium">WebSocket 地址</label>
            <div class="flex gap-2">
              <select
                v-model="wsProtocol"
                :disabled="!wsCanConnect"
                class="shrink-0 rounded-xl border border-slate-300 bg-white px-3 py-3 text-sm outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100 disabled:opacity-50"
              >
                <option value="wss">wss://</option>
                <option value="ws">ws://</option>
              </select>
              <input
                v-model="wsUrl"
                :disabled="!wsCanConnect"
                class="flex-1 rounded-xl border border-slate-300 bg-white px-4 py-3 text-sm outline-none transition focus:border-blue-500 focus:ring-4 focus:ring-blue-100 disabled:opacity-50"
                placeholder="echo.websocket.org"
                @keydown.enter="wsConnect"
              >
            </div>
          </div>

          <!-- 连接 / 断开 -->
          <div class="mb-4">
            <button
              v-if="!wsIsConnected"
              type="button"
              @click="wsConnect"
              :disabled="wsStatus === 'connecting' || !wsUrl.trim()"
              :class="wsStatus === 'connecting' ? 'cursor-not-allowed bg-slate-400' : 'bg-emerald-600 hover:bg-emerald-500'"
              class="w-full rounded-xl px-5 py-3 text-sm font-semibold text-white shadow-sm transition disabled:opacity-60"
            >{{ wsStatus === 'connecting' ? '连接中...' : '连接' }}</button>
            <button
              v-else
              type="button"
              @click="wsDisconnect"
              class="w-full rounded-xl bg-red-500 px-5 py-3 text-sm font-semibold text-white shadow-sm hover:bg-red-400 transition"
            >断开连接</button>
          </div>

          <!-- 发送消息 -->
          <div class="mb-4">
            <div class="mb-2 flex items-center justify-between">
              <label class="text-sm font-medium">发送消息</label>
              <span class="text-xs text-slate-400">Enter 发送 · Shift+Enter 换行</span>
            </div>
            <textarea
              v-model="wsInput"
              rows="8"
              :disabled="!wsIsConnected"
              class="w-full rounded-xl border border-slate-300 bg-slate-950 p-4 font-mono text-sm text-slate-100 outline-none transition focus:border-blue-500 focus:ring-4 focus:ring-blue-100 disabled:opacity-40"
              placeholder='{"type": "ping"}'
              @keydown="wsHandleKeydown"
            />
          </div>

          <button
            type="button"
            @click="wsSend"
            :disabled="!wsIsConnected || !wsInput.trim()"
            :class="wsIsConnected && wsInput.trim() ? 'bg-blue-600 hover:bg-blue-500' : 'cursor-not-allowed bg-slate-400'"
            class="w-full rounded-xl px-5 py-3 text-sm font-semibold text-white shadow-sm transition"
          >发送</button>

        </div>
      </div>
    </div>

  </div>
</template>
