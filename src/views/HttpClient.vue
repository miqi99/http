<script setup>
import { computed, onMounted, ref, watch } from 'vue'

const props = defineProps({ persistConfig: Boolean })
const emit = defineEmits(['changeTab'])

const STORAGE_KEY = 'http-client-config'
const HISTORY_KEY = 'http-client-history'

const url = ref('')
const protocol = ref('https')
const method = ref('GET')
const accessMode = ref('direct')
const paramType = ref('none')
const headers = ref([])
const cookie = ref('')
const cookieEnabled = ref(false)
const formParams = ref([{ key: '', value: '' }])
const jsonBody = ref('')
const formatRequestJson = ref(false)
const response = ref('')
const responseInfo = ref({ status: '', statusText: '', time: '', size: '', contentType: '', url: '' })
const responseTab = ref('body')
const loading = ref(false)
const timeout = ref(false)
const formatResponseJson = ref(true)
const showCurlModal = ref(false)
const curlText = ref('')
const curlError = ref('')
const historyList = ref([])
const copyTip = ref('')
const showExpandModal = ref(false)
const initialized = ref(false)

watch(url, (newVal) => {
  if (/^https:\/\//i.test(newVal)) {
    protocol.value = 'https'
    url.value = newVal.replace(/^https:\/\//i, '')
  } else if (/^http:\/\//i.test(newVal)) {
    protocol.value = 'http'
    url.value = newVal.replace(/^http:\/\//i, '')
  }
})

const urlError = computed(() => {
  const raw = url.value.trim()
  if (!raw) return ''
  try { new URL(`${protocol.value}://${raw}`); return '' }
  catch { return '地址格式不正确，例如 api.example.com 或 api.example.com/path' }
})

const buttonText = computed(() => {
  if (loading.value) return '请求中...'
  if (timeout.value) return '已超时 请重试'
  return '发送请求'
})

const displayRequestJson = computed(() => {
  if (!formatRequestJson.value) return jsonBody.value
  try { return JSON.stringify(JSON.parse(jsonBody.value), null, 2) }
  catch { return jsonBody.value }
})

const displayResponse = computed(() => {
  if (!formatResponseJson.value) return response.value
  try { return JSON.stringify(JSON.parse(response.value), null, 2) }
  catch { return response.value }
})

function escapeHtml(text) {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
}

function highlightJson(json) {
  let html = ''
  let i = 0
  while (i < json.length) {
    const ch = json[i]
    if (ch === '"') {
      let j = i + 1
      const buf = [ch]
      while (j < json.length) {
        if (json[j] === '\\') { buf.push(json[j], json[j + 1] || ''); j += 2; continue }
        if (json[j] === '"') { buf.push(json[j]); j++; break }
        buf.push(json[j]); j++
      }
      const full = buf.join('')
      let k = j
      while (k < json.length && /\s/.test(json[k])) k++
      const isKey = k < json.length && json[k] === ':'
      html += `<span class="${isKey ? 'json-key' : 'json-string'}">${escapeHtml(full)}</span>`
      i = j
      continue
    }
    if ((ch === '-' && i + 1 < json.length && /\d/.test(json[i + 1])) || /\d/.test(ch)) {
      let j = i, num = ''
      while (j < json.length && /[-\d.eE+]/.test(json[j])) num += json[j++]
      html += `<span class="json-number">${num}</span>`
      i = j
      continue
    }
    if (json.startsWith('true', i)) { html += '<span class="json-boolean">true</span>'; i += 4; continue }
    if (json.startsWith('false', i)) { html += '<span class="json-boolean">false</span>'; i += 5; continue }
    if (json.startsWith('null', i)) { html += '<span class="json-null">null</span>'; i += 4; continue }
    html += escapeHtml(ch)
    i++
  }
  return html
}

const highlightedResponse = computed(() => {
  if (!response.value) return ''
  try {
    JSON.parse(response.value)
    return highlightJson(displayResponse.value)
  } catch {
    return escapeHtml(response.value)
  }
})

const statusColorClass = computed(() => {
  const status = String(responseInfo.value.status || '')
  if (status.startsWith('2')) return 'text-green-600'
  if (status.startsWith('3')) return 'text-blue-600'
  if (status.startsWith('4')) return 'text-orange-600'
  if (status.startsWith('5')) return 'text-red-600'
  if (status === '请求失败') return 'text-red-600'
  return 'text-slate-900'
})

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved) {
    try {
      const config = JSON.parse(saved)
      url.value = config.url || ''
      protocol.value = config.protocol || 'https'
      method.value = config.method || 'GET'
      accessMode.value = config.accessMode || 'direct'
      paramType.value = config.paramType || 'none'
      headers.value = Array.isArray(config.headers) && config.headers.length > 0 ? config.headers : []
      cookie.value = config.cookie || ''
      cookieEnabled.value = Boolean(config.cookieEnabled)
      formParams.value = Array.isArray(config.formParams) && config.formParams.length > 0
        ? config.formParams
        : [{ key: '', value: '' }]
      jsonBody.value = config.jsonBody || ''
      formatRequestJson.value = Boolean(config.formatRequestJson)
      formatResponseJson.value = config.formatResponseJson !== false
    } catch {
      localStorage.removeItem(STORAGE_KEY)
    }
  }

  const savedHistory = localStorage.getItem(HISTORY_KEY)
  if (savedHistory) {
    try {
      const list = JSON.parse(savedHistory)
      historyList.value = Array.isArray(list) ? list : []
    } catch {
      localStorage.removeItem(HISTORY_KEY)
    }
  }

  initialized.value = true
})

watch(
  [() => props.persistConfig, url, protocol, method, accessMode, paramType, headers, cookie, cookieEnabled, formParams, jsonBody, formatRequestJson, formatResponseJson],
  () => {
    if (!initialized.value) return
    if (!props.persistConfig) { localStorage.removeItem(STORAGE_KEY); return }
    localStorage.setItem(STORAGE_KEY, JSON.stringify({
      persistConfig: props.persistConfig,
      url: url.value, protocol: protocol.value, method: method.value,
      accessMode: accessMode.value, paramType: paramType.value,
      headers: headers.value, cookie: cookie.value, cookieEnabled: cookieEnabled.value,
      formParams: formParams.value, jsonBody: jsonBody.value,
      formatRequestJson: formatRequestJson.value, formatResponseJson: formatResponseJson.value,
    }))
  },
  { deep: true }
)

function addHeader() { headers.value.push({ key: '', value: '' }) }
function removeHeader(index) { headers.value.splice(index, 1) }
function addFormParam() { formParams.value.push({ key: '', value: '' }) }
function removeFormParam(index) { if (formParams.value.length > 1) formParams.value.splice(index, 1) }

function compressJson() {
  try { jsonBody.value = JSON.stringify(JSON.parse(jsonBody.value)) } catch {}
}

function hasHeader(headersObject, headerName) {
  return Object.keys(headersObject).some(key => key.toLowerCase() === headerName.toLowerCase())
}

function buildHeaders() {
  const requestHeaders = {}
  headers.value.forEach(item => { const k = item.key.trim(); if (k) requestHeaders[k] = item.value })
  if (cookieEnabled.value && cookie.value.trim()) requestHeaders.Cookie = cookie.value.trim()
  if (!hasHeader(requestHeaders, 'Content-Type') && method.value === 'POST') {
    if (paramType.value === 'json') requestHeaders['Content-Type'] = 'application/json; charset=utf-8'
    if (paramType.value === 'form') requestHeaders['Content-Type'] = 'application/x-www-form-urlencoded; charset=utf-8'
  }
  return requestHeaders
}

function buildBody() {
  if (method.value !== 'POST') return undefined
  if (paramType.value === 'json') return jsonBody.value
  if (paramType.value === 'form') {
    const formData = new URLSearchParams()
    formParams.value.forEach(item => { const k = item.key.trim(); if (k) formData.append(k, item.value) })
    return formData
  }
  return undefined
}

function formatSize(text) {
  const bytes = new Blob([text]).size
  return bytes < 1024 ? `${bytes} B` : `${(bytes / 1024).toFixed(2)} KB`
}

function validateJsonBeforeSend() {
  if (method.value !== 'POST' || paramType.value !== 'json') return true
  if (!jsonBody.value.trim()) { response.value = 'JSON 参数不能为空'; responseTab.value = 'body'; return false }
  try { JSON.parse(jsonBody.value); return true }
  catch (err) { response.value = `JSON 格式错误，已阻止发送：\n${err.message}`; responseTab.value = 'body'; return false }
}

function saveHistoryEntry(ok) {
  const entry = {
    time: new Date().toLocaleString(), url: url.value, protocol: protocol.value, method: method.value,
    accessMode: accessMode.value, paramType: paramType.value,
    headers: JSON.parse(JSON.stringify(headers.value)), cookie: cookie.value,
    formParams: JSON.parse(JSON.stringify(formParams.value)), jsonBody: jsonBody.value,
    response: response.value, responseInfo: JSON.parse(JSON.stringify(responseInfo.value)),
    ok,
  }
  historyList.value = [
    entry,
    ...historyList.value.filter(item => !(item.url === entry.url && item.method === entry.method)),
  ].slice(0, 20)
  localStorage.setItem(HISTORY_KEY, JSON.stringify(historyList.value))
}

function restoreHistory(index) {
  const entry = historyList.value[index]
  if (!entry) return
  url.value = entry.url || ''
  protocol.value = entry.protocol || 'https'
  method.value = entry.method || 'GET'
  accessMode.value = entry.accessMode || 'direct'
  paramType.value = entry.paramType || 'none'
  headers.value = Array.isArray(entry.headers) && entry.headers.length > 0 ? entry.headers : []
  cookie.value = entry.cookie || ''
  cookieEnabled.value = Boolean(entry.cookie)
  formParams.value = Array.isArray(entry.formParams) && entry.formParams.length > 0
    ? entry.formParams : [{ key: '', value: '' }]
  jsonBody.value = entry.jsonBody || ''
  response.value = entry.response || ''
  responseInfo.value = entry.responseInfo || { status: '', statusText: '', time: '', size: '', contentType: '', url: '' }
  responseTab.value = 'body'
}

function clearHistory() {
  historyList.value = []
  localStorage.removeItem(HISTORY_KEY)
}

function clearAll() {
  url.value = ''; protocol.value = 'https'; method.value = 'GET'; accessMode.value = 'direct'
  paramType.value = 'none'; headers.value = []; cookie.value = ''; cookieEnabled.value = false
  formParams.value = [{ key: '', value: '' }]; jsonBody.value = ''; formatRequestJson.value = false
  response.value = ''; responseTab.value = 'body'
  responseInfo.value = { status: '', statusText: '', time: '', size: '', contentType: '', url: '' }
  timeout.value = false
}

async function copyResponse() {
  if (!response.value) {
    copyTip.value = '暂无内容'; setTimeout(() => { copyTip.value = '' }, 1200); return
  }
  await navigator.clipboard.writeText(response.value)
  copyTip.value = '已复制'; setTimeout(() => { copyTip.value = '' }, 1200)
}

function openCurlModal() { curlText.value = ''; curlError.value = ''; showCurlModal.value = true }
function closeCurlModal() { showCurlModal.value = false }

function splitCurlCommand(command) {
  const result = []; let current = ''; let quote = ''; let escaping = false
  for (const char of command) {
    if (escaping) { current += char; escaping = false; continue }
    if (char === '\\') { escaping = true; continue }
    if (quote) { if (char === quote) quote = ''; else current += char; continue }
    if (char === '"' || char === "'") { quote = char; continue }
    if (/\s/.test(char)) { if (current) { result.push(current); current = '' }; continue }
    current += char
  }
  if (current) result.push(current)
  return result
}

function parseFormString(text) {
  const params = new URLSearchParams(text)
  const rows = []
  params.forEach((value, key) => rows.push({ key, value }))
  return rows.length > 0 ? rows : [{ key: '', value: '' }]
}

function importCurl() {
  curlError.value = ''
  const raw = curlText.value.trim()
  if (!raw) { curlError.value = '请先粘贴 CURL 内容'; return }
  try {
    const normalized = raw.replace(/\\\r?\n/g, ' ').replace(/\r?\n/g, ' ')
    const tokens = splitCurlCommand(normalized)
    if (!tokens.length || tokens[0] !== 'curl') { curlError.value = '请输入以 curl 开头的命令'; return }
    const nextHeaders = []; let nextCookie = ''; let nextMethod = ''; let nextBody = ''; let nextUrl = ''
    for (let i = 1; i < tokens.length; i += 1) {
      const token = tokens[i]
      if (token === '-X' || token === '--request') { nextMethod = (tokens[i + 1] || '').toUpperCase(); i += 1; continue }
      if (token === '-H' || token === '--header') {
        const headerLine = tokens[i + 1] || ''
        const colonIndex = headerLine.indexOf(':')
        if (colonIndex > -1) {
          const key = headerLine.slice(0, colonIndex).trim()
          const value = headerLine.slice(colonIndex + 1).trim()
          if (key.toLowerCase() === 'cookie') nextCookie = value
          else nextHeaders.push({ key, value })
        }
        i += 1; continue
      }
      if (token === '-d' || token === '--data' || token === '--data-raw' || token === '--data-binary' || token === '--data-urlencode') {
        nextBody = tokens[i + 1] || ''; i += 1; continue
      }
      if (!token.startsWith('-') && !nextUrl) nextUrl = token
    }
    if (!nextUrl) { curlError.value = '没有解析到请求地址'; return }
    if (nextUrl.startsWith('http://')) protocol.value = 'http'
    else if (nextUrl.startsWith('https://')) protocol.value = 'https'
    url.value = nextUrl; cookie.value = nextCookie
    headers.value = nextHeaders.length > 0 ? nextHeaders : [{ key: '', value: '' }]
    const contentTypeHeader = nextHeaders.find(item => item.key.toLowerCase() === 'content-type')
    const contentType = contentTypeHeader?.value?.toLowerCase() || ''
    if (nextBody) {
      method.value = nextMethod || 'POST'
      if (contentType.includes('application/json') || nextBody.trim().startsWith('{') || nextBody.trim().startsWith('[')) {
        paramType.value = 'json'; jsonBody.value = nextBody
        try { jsonBody.value = JSON.stringify(JSON.parse(nextBody), null, 2); formatRequestJson.value = true }
        catch { formatRequestJson.value = false }
      } else {
        paramType.value = 'form'; formParams.value = parseFormString(nextBody)
      }
    } else {
      method.value = nextMethod || 'GET'; paramType.value = 'none'
    }
    showCurlModal.value = false
  } catch (err) {
    curlError.value = err.message || String(err)
  }
}

async function sendRequest() {
  if (loading.value) return
  if (!url.value.trim()) { response.value = '请先输入请求地址'; responseTab.value = 'body'; return }
  if (urlError.value) { response.value = urlError.value; responseTab.value = 'body'; return }
  if (!validateJsonBeforeSend()) return
  if (location.protocol === 'https:' && protocol.value === 'http') {
    const host = url.value.trim().split('/')[0].split(':')[0]
    const isLocal = host === 'localhost' || host === '127.0.0.1' || host.endsWith('.local')
    response.value = isLocal
      ? '请求被浏览器拦截（混合内容）\n\n当前页面通过 HTTPS 访问，无法向本地 HTTP 服务发送请求。\n\n解决方法：在本地运行 npm run dev，通过 http://localhost:5173 使用本工具。'
      : '请求被浏览器拦截（混合内容）\n\n当前页面通过 HTTPS 访问，不允许向 HTTP 地址发送请求。\n\n请将目标接口改为 https://，或在本地运行本工具。'
    responseTab.value = 'body'
    return
  }

  loading.value = true; timeout.value = false; response.value = '请求中...'; responseTab.value = 'body'
  responseInfo.value = { status: '', statusText: '', time: '', size: '', contentType: '', url: '' }

  const startTime = Date.now()
  const timer = setTimeout(() => {
    loading.value = false; timeout.value = true
    response.value = '请求超过 5 秒，请重试'
    responseInfo.value = { ...responseInfo.value, time: `${Date.now() - startTime} ms`, size: formatSize(response.value) }
    saveHistoryEntry(false)
  }, 5000)

  try {
    const requestHeaders = buildHeaders()
    const body = buildBody()
    let requestUrl = `${protocol.value}://${url.value.trim()}`
    if (method.value !== 'POST' && paramType.value === 'form') {
      const params = new URLSearchParams()
      formParams.value.forEach(item => { const k = item.key.trim(); if (k) params.append(k, item.value) })
      const qs = params.toString()
      if (qs) requestUrl += (requestUrl.includes('?') ? '&' : '?') + qs
    }
    if (accessMode.value === 'proxy') {
      clearTimeout(timer); loading.value = false; timeout.value = false
      response.value = '代理转发功能还未接入。\n下一步需要启动本地 Node.js 代理服务：http://localhost:3000/proxy'
      responseInfo.value = { status: '未发送', statusText: '代理未接入', time: `${Date.now() - startTime} ms`, size: formatSize(response.value), contentType: '', url: 'http://localhost:3000/proxy' }
      return
    }
    const res = await fetch(requestUrl, { method: method.value, headers: requestHeaders, body })
    const text = await res.text()
    clearTimeout(timer); loading.value = false; timeout.value = false; response.value = text
    responseInfo.value = {
      status: String(res.status), statusText: res.statusText || '',
      time: `${Date.now() - startTime} ms`, size: formatSize(text),
      contentType: res.headers.get('content-type') || '', url: res.url || url.value.trim(),
    }
    saveHistoryEntry(res.ok)
  } catch (err) {
    clearTimeout(timer); loading.value = false; timeout.value = false
    response.value = err.message || String(err)
    responseInfo.value = {
      status: '请求失败', statusText: err.name || '',
      time: `${Date.now() - startTime} ms`, size: formatSize(response.value),
      contentType: '', url: url.value.trim(),
    }
    saveHistoryEntry(false)
  }
}
</script>

<template>
  <div class="h-full">
  <div class="mx-auto flex h-full max-w-[1600px] gap-3 px-3 py-3">

    <!-- 左栏：请求历史 -->
    <aside class="flex h-full w-[240px] shrink-0 flex-col rounded-2xl bg-white shadow-sm ring-1 ring-slate-200">
      <div class="flex shrink-0 items-center justify-between border-b border-slate-100 px-3 py-2.5">
        <h3 class="text-sm font-semibold">请求历史</h3>
        <button
          type="button"
          @click="clearHistory"
          :disabled="historyList.length === 0"
          :class="historyList.length === 0 ? 'cursor-not-allowed text-slate-300' : 'text-red-500 hover:bg-red-50'"
          class="rounded-md px-2 py-1 text-xs font-medium transition"
        >清空</button>
      </div>
      <div class="min-h-0 flex-1 overflow-y-auto p-2">
        <div v-if="historyList.length === 0" class="px-2 py-10 text-center text-xs text-slate-400">暂无请求历史<br>发送成功后会自动记录</div>
        <button
          v-for="(item, index) in historyList"
          :key="index"
          type="button"
          @click="restoreHistory(index)"
          class="mb-1.5 block w-full rounded-lg border border-slate-200 px-2.5 py-2 text-left transition hover:border-blue-300 hover:bg-blue-50/60"
        >
          <div class="flex items-center gap-1.5">
            <span :class="item.method === 'POST' ? 'bg-blue-100 text-blue-700' : 'bg-emerald-100 text-emerald-700'" class="inline-block rounded px-1.5 py-0.5 text-[11px] font-semibold">{{ item.method }}</span>
            <span class="rounded bg-slate-100 px-1.5 py-0.5 text-[11px] text-slate-500">{{ item.paramType }}</span>
            <span v-if="item.ok === true" class="ml-auto inline-flex items-center gap-1 rounded bg-emerald-100 px-1.5 py-0.5 text-[11px] font-semibold text-emerald-700"><span class="h-1.5 w-1.5 rounded-full bg-emerald-500"></span>成功</span>
            <span v-else-if="item.ok === false" class="ml-auto inline-flex items-center gap-1 rounded bg-red-100 px-1.5 py-0.5 text-[11px] font-semibold text-red-700"><span class="h-1.5 w-1.5 rounded-full bg-red-500"></span>失败</span>
          </div>
          <div class="mt-1 truncate font-mono text-xs text-slate-700" :title="item.url">{{ item.url }}</div>
          <div class="mt-0.5 text-[11px] text-slate-400">{{ item.time }}</div>
        </button>
      </div>
    </aside>

    <!-- 中栏：请求构建器 -->
    <div class="flex h-full min-w-0 flex-1 flex-col rounded-2xl bg-white shadow-sm ring-1 ring-slate-200">

      <!-- Tab 行 + 操作按钮 -->
      <div class="flex shrink-0 items-center justify-between border-b border-slate-100 px-3 py-2">
        <div class="flex items-center gap-1">
          <button
            type="button"
            class="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white"
          >HTTP</button>
          <button
            type="button"
            @click="emit('changeTab', 'ws')"
            class="rounded-lg px-3 py-1.5 text-xs font-semibold text-slate-500 hover:text-slate-800 hover:bg-slate-50 transition"
          >WebSocket</button>
        </div>
        <div class="flex items-center gap-2">
          <button
            type="button"
            @click="openCurlModal"
            class="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white hover:bg-slate-700 transition"
          >CURL 导入</button>
          <button
            type="button"
            @click="clearAll"
            class="rounded-lg bg-white px-3 py-1.5 text-xs font-semibold text-red-500 ring-1 ring-red-100 hover:bg-red-50 transition"
          >一键清空</button>
        </div>
      </div>

      <!-- 表单内容（可滚动） -->
      <div class="min-h-0 flex-1 overflow-y-auto p-3">

        <!-- 请求地址 -->
        <div class="mb-3">
          <label class="mb-1.5 block text-xs font-medium text-slate-600">请求地址</label>
          <div class="flex gap-2">
            <select
              v-model="protocol"
              class="shrink-0 rounded-lg border border-slate-300 bg-white px-2.5 py-2 text-sm outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100"
            >
              <option value="https">https://</option>
              <option value="http">http://</option>
            </select>
            <input
              v-model="url"
              :class="urlError
                ? 'border-red-400 focus:border-red-400 focus:ring-red-100'
                : 'border-slate-300 focus:border-blue-500 focus:ring-blue-100'"
              class="flex-1 rounded-lg border bg-white px-3 py-2 text-sm outline-none transition focus:ring-2"
              placeholder="api.example.com"
            >
          </div>
          <p v-if="urlError" class="mt-1 text-xs text-red-500">{{ urlError }}</p>
        </div>

        <!-- 请求方式 + 访问模式 -->
        <div class="mb-3 grid gap-3 md:grid-cols-2">
          <div>
            <label class="mb-1.5 block text-xs font-medium text-slate-600">请求方式</label>
            <div class="grid grid-cols-2 gap-2">
              <button type="button" @click="method = 'GET'" :class="method === 'GET' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-1 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">GET</button>
              <button type="button" @click="method = 'POST'" :class="method === 'POST' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-1 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">POST</button>
            </div>
          </div>
          <div>
            <label class="mb-1.5 block text-xs font-medium text-slate-600">访问模式</label>
            <div class="grid grid-cols-2 gap-2">
              <button type="button" @click="accessMode = 'direct'" :class="accessMode === 'direct' ? 'border-emerald-600 bg-emerald-50 text-emerald-700 ring-1 ring-emerald-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">本地直连</button>
              <button type="button" @click="accessMode = 'proxy'" :class="accessMode === 'proxy' ? 'border-emerald-600 bg-emerald-50 text-emerald-700 ring-1 ring-emerald-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">代理转发</button>
            </div>
          </div>
        </div>

        <!-- 请求头 -->
        <div class="mb-3">
          <div class="mb-2 flex items-center justify-between gap-3">
            <label class="text-xs font-medium text-slate-600">请求头<span class="ml-1 text-slate-400">可选，自填 Content-Type 优先生效</span></label>
            <button type="button" @click="addHeader" class="shrink-0 rounded-lg bg-slate-900 px-2.5 py-1.5 text-xs font-semibold text-white hover:bg-slate-700 transition">添加</button>
          </div>
          <div class="space-y-2">
            <div v-for="(item, index) in headers" :key="index" class="grid grid-cols-[1fr_1fr_auto] gap-2">
              <input v-model="item.key" class="rounded-lg border border-slate-300 px-2.5 py-2 text-sm outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100" placeholder="Header Key">
              <input v-model="item.value" class="rounded-lg border border-slate-300 px-2.5 py-2 text-sm outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100" placeholder="Header Value">
              <button type="button" @click="removeHeader(index)" class="rounded-lg border border-slate-300 px-2.5 text-sm text-slate-500 hover:bg-slate-100">删除</button>
            </div>
          </div>
        </div>

        <!-- Cookie -->
        <div class="mb-3">
          <div class="mb-1.5 flex items-center justify-between">
            <label class="text-xs font-medium text-slate-600">Cookie</label>
            <button type="button" @click="cookieEnabled = !cookieEnabled" :class="cookieEnabled ? 'bg-blue-600 text-white hover:bg-blue-500' : 'bg-slate-100 text-slate-500 hover:bg-slate-200'" class="rounded-full px-2.5 py-1 text-xs font-semibold transition">{{ cookieEnabled ? '已开启' : '已关闭' }}</button>
          </div>
          <textarea v-if="cookieEnabled" v-model="cookie" rows="2" class="w-full rounded-lg border border-slate-300 bg-white p-2.5 text-sm outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100" placeholder="token=123; uid=888" />
        </div>

        <!-- 参数类型 -->
        <div class="mb-3">
          <label class="mb-1.5 block text-xs font-medium text-slate-600">参数类型</label>
          <div class="grid grid-cols-3 gap-2">
            <button type="button" @click="paramType = 'none'" :class="paramType === 'none' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-1 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">无参数</button>
            <button type="button" @click="paramType = 'form'" :class="paramType === 'form' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-1 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">表单</button>
            <button type="button" @click="paramType = 'json'" :class="paramType === 'json' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-1 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'" class="rounded-lg border px-3 py-2 text-sm font-semibold transition">JSON</button>
          </div>
        </div>

        <!-- 表单参数 -->
        <div v-if="paramType === 'form'" class="mb-3">
          <div class="mb-2 flex items-center justify-between gap-3">
            <label class="text-xs font-medium text-slate-600">表单参数<span class="ml-1 text-slate-400">多行会自动转成表单提交</span></label>
            <button type="button" @click="addFormParam" class="shrink-0 rounded-lg bg-slate-900 px-2.5 py-1.5 text-xs font-semibold text-white hover:bg-slate-700 transition">添加</button>
          </div>
          <div class="space-y-2">
            <div v-for="(item, index) in formParams" :key="index" class="grid grid-cols-[1fr_1fr_auto] gap-2">
              <input v-model="item.key" class="rounded-lg border border-slate-300 px-2.5 py-2 text-sm outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100" placeholder="参数 Key">
              <input v-model="item.value" class="rounded-lg border border-slate-300 px-2.5 py-2 text-sm outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100" placeholder="参数 Value">
              <button type="button" @click="removeFormParam(index)" :disabled="formParams.length === 1" :class="formParams.length === 1 ? 'cursor-not-allowed text-slate-300' : 'text-slate-500 hover:bg-slate-100'" class="rounded-lg border border-slate-300 px-2.5 text-sm">删除</button>
            </div>
          </div>
        </div>

        <!-- JSON 参数 -->
        <div v-if="paramType === 'json'" class="mb-3">
          <div class="mb-1.5 flex items-center justify-between gap-3">
            <label class="text-xs font-medium text-slate-600">JSON 参数</label>
            <label class="flex shrink-0 items-center gap-1.5 text-xs text-slate-600 cursor-pointer">
              <input v-model="formatRequestJson" type="checkbox" class="h-3.5 w-3.5">
              格式化 JSON
            </label>
          </div>
          <textarea :value="displayRequestJson" rows="8" class="w-full rounded-lg border border-slate-300 bg-slate-950 p-3 font-mono text-sm text-slate-100 outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100" @input="jsonBody = $event.target.value" @blur="compressJson" />
        </div>

      </div>

      <!-- 发送（固定底部） -->
      <div class="shrink-0 border-t border-slate-100 p-3">
        <button
          type="button"
          @click="sendRequest"
          :disabled="loading"
          :class="loading ? 'cursor-not-allowed bg-slate-400' : timeout ? 'bg-orange-500 hover:bg-orange-400' : 'bg-blue-600 hover:bg-blue-500'"
          class="w-full rounded-lg px-5 py-2.5 text-sm font-semibold text-white shadow-sm transition"
        >{{ buttonText }}</button>
      </div>
    </div>

    <!-- 右栏：返回结果 -->
    <div class="flex h-full w-[560px] shrink-0 flex-col rounded-2xl bg-white p-3 shadow-sm ring-1 ring-slate-200">

      <div class="mb-3 flex shrink-0 items-center justify-between gap-2">
        <div class="grid grid-cols-2 gap-1 rounded-xl bg-slate-100 p-1">
          <button
            type="button"
            @click="responseTab = 'body'"
            :class="responseTab === 'body' ? 'bg-white text-slate-900 shadow-sm' : 'text-slate-500 hover:text-slate-700'"
            class="rounded-lg px-3 py-1.5 text-xs font-semibold transition"
          >返回结果</button>
          <button
            type="button"
            @click="responseTab = 'info'"
            :class="responseTab === 'info' ? 'bg-white text-slate-900 shadow-sm' : 'text-slate-500 hover:text-slate-700'"
            class="rounded-lg px-3 py-1.5 text-xs font-semibold transition"
          >请求响应</button>
        </div>
        <div class="flex shrink-0 items-center gap-2">
          <template v-if="responseTab === 'body'">
            <span v-if="copyTip" class="text-xs text-slate-500">{{ copyTip }}</span>
            <button
              type="button"
              @click="showExpandModal = true"
              :disabled="!response"
              :class="response ? 'hover:bg-slate-700' : 'cursor-not-allowed opacity-50'"
              class="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white transition"
            >展开</button>
            <button
              type="button"
              @click="copyResponse"
              class="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white hover:bg-slate-700 transition"
            >复制</button>
            <label class="flex items-center gap-1.5 text-xs text-slate-600 cursor-pointer">
              <input v-model="formatResponseJson" type="checkbox" class="h-3.5 w-3.5">
              格式化
            </label>
          </template>
          <button
            type="button"
            @click="response = ''; responseInfo = { status: '', statusText: '', time: '', size: '', contentType: '', url: '' }"
            class="rounded-lg border border-slate-200 px-2.5 py-1 text-xs text-slate-500 hover:bg-slate-100 transition"
          >清空</button>
        </div>
      </div>

      <pre
        v-if="responseTab === 'body' && response"
        v-html="highlightedResponse"
        class="min-h-0 flex-1 w-full resize-none rounded-xl border border-slate-300 bg-slate-950 p-3 font-mono text-sm text-slate-100 outline-none overflow-auto whitespace-pre-wrap break-all"
      />
      <div
        v-else-if="responseTab === 'body'"
        class="min-h-0 flex-1 w-full resize-none rounded-xl border border-slate-300 bg-slate-950 p-3 font-mono text-sm text-slate-500 select-none"
      >暂无返回结果</div>
      <div
        v-else
        class="min-h-0 flex-1 overflow-y-auto rounded-xl border border-slate-300 bg-slate-50 p-3"
      >
        <div class="grid grid-cols-2 gap-2">
          <div class="rounded-lg bg-white p-3 ring-1 ring-slate-200">
            <div class="text-xs text-slate-500">状态码</div>
            <div :class="statusColorClass" class="mt-1 font-mono text-sm font-semibold">{{ responseInfo.status || '-' }}</div>
          </div>
          <div class="rounded-lg bg-white p-3 ring-1 ring-slate-200">
            <div class="text-xs text-slate-500">状态文本</div>
            <div class="mt-1 font-mono text-sm text-slate-900">{{ responseInfo.statusText || '-' }}</div>
          </div>
          <div class="rounded-lg bg-white p-3 ring-1 ring-slate-200">
            <div class="text-xs text-slate-500">响应时间</div>
            <div class="mt-1 font-mono text-sm text-slate-900">{{ responseInfo.time || '-' }}</div>
          </div>
          <div class="rounded-lg bg-white p-3 ring-1 ring-slate-200">
            <div class="text-xs text-slate-500">响应大小</div>
            <div class="mt-1 font-mono text-sm text-slate-900">{{ responseInfo.size || '-' }}</div>
          </div>
          <div class="col-span-2 rounded-lg bg-white p-3 ring-1 ring-slate-200">
            <div class="text-xs text-slate-500">Content-Type</div>
            <div class="mt-1 break-all font-mono text-sm text-slate-900">{{ responseInfo.contentType || '-' }}</div>
          </div>
          <div class="col-span-2 rounded-lg bg-white p-3 ring-1 ring-slate-200">
            <div class="text-xs text-slate-500">最终请求地址</div>
            <div class="mt-1 break-all font-mono text-sm text-slate-900">{{ responseInfo.url || '-' }}</div>
          </div>
        </div>
      </div>

    </div>

  </div>

  <!-- 展开返回结果弹窗 -->
  <div
    :class="showExpandModal ? 'modal-show' : ''"
    class="modal-mask"
    @click.self="showExpandModal = false"
  >
    <div class="modal-expand" role="dialog" aria-modal="true">
      <div class="modal-head">
        <div>
          <h3>返回结果</h3>
          <p v-if="responseInfo.status" class="mt-1 text-xs text-slate-400">
            {{ responseInfo.status }} {{ responseInfo.statusText }} · {{ responseInfo.time }} · {{ responseInfo.size }}
          </p>
        </div>
        <button
          type="button"
          @click="showExpandModal = false"
          class="rounded-lg border border-slate-200 px-4 py-2 text-sm font-semibold text-slate-600 hover:bg-slate-100 transition"
        >关闭</button>
      </div>
      <div class="modal-body">
        <pre
          v-if="response"
          v-html="highlightedResponse"
          class="w-full rounded-xl bg-slate-950 p-5 font-mono text-sm text-slate-100 overflow-auto whitespace-pre-wrap break-all"
        />
        <div
          v-else
          class="flex items-center justify-center rounded-xl border border-dashed border-slate-200 bg-slate-50 py-24 text-sm text-slate-400"
        >暂无返回结果</div>
      </div>
    </div>
  </div>

  <!-- CURL 导入弹窗 -->
  <div
    :class="showCurlModal ? 'modal-show' : ''"
    class="modal-mask"
    @click.self="closeCurlModal"
  >
    <div class="modal-expand" role="dialog" aria-modal="true">
      <div class="modal-head">
        <div>
          <h3>CURL 导入</h3>
          <p class="mt-1 text-xs text-slate-400">粘贴 curl 命令后，会自动填充 URL、请求方式、请求头、Cookie 和参数</p>
        </div>
        <button
          type="button"
          @click="closeCurlModal"
          class="rounded-lg border border-slate-200 px-4 py-2 text-sm font-semibold text-slate-600 hover:bg-slate-100 transition"
        >关闭</button>
      </div>
      <div class="modal-body">
        <textarea
          v-model="curlText"
          rows="14"
          class="w-full rounded-xl border border-slate-300 bg-slate-950 p-4 font-mono text-sm text-slate-100 outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
          placeholder="curl 'https://api.example.com/user' -H 'Authorization: Bearer xxx' -H 'Content-Type: application/json' --data '{&quot;id&quot;:1}'"
        />
        <p v-if="curlError" class="mt-3 rounded-lg bg-red-50 px-3 py-2 text-sm text-red-600">{{ curlError }}</p>
        <div class="mt-5 flex justify-end gap-3">
          <button type="button" @click="closeCurlModal" class="rounded-xl border border-slate-300 px-5 py-2 text-sm font-semibold text-slate-600 hover:bg-slate-100">取消</button>
          <button type="button" @click="importCurl" class="rounded-xl bg-blue-600 px-5 py-2 text-sm font-semibold text-white hover:bg-blue-500">导入</button>
        </div>
      </div>
    </div>
  </div>
  </div>
</template>

<style>
.json-key        { color: #93c5fd; }  /* blue-300 */
.json-string     { color: #86efac; }  /* green-300 */
.json-number     { color: #fdba74; }  /* orange-300 */
.json-boolean    { color: #c4b5fd; }  /* violet-300 */
.json-null       { color: #94a3b8; }  /* slate-400 */

.modal-mask {
  position: fixed;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  background: rgba(15, 23, 42, 0);
  opacity: 0;
  visibility: hidden;
  pointer-events: none;
  z-index: 100;
  transition:
    opacity .26s cubic-bezier(.2, .8, .2, 1),
    visibility .26s cubic-bezier(.2, .8, .2, 1),
    background .26s cubic-bezier(.2, .8, .2, 1);
}

.modal-mask.modal-show {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
  background: rgba(15, 23, 42, .42);
}

.modal-expand {
  width: min(960px, 100%);
  max-height: min(760px, calc(100vh - 40px));
  display: flex;
  flex-direction: column;
  background: #fff;
  border-radius: 22px;
  box-shadow: 0 28px 90px rgba(15, 23, 42, .28);
  overflow: hidden;
  transform: scale(.72);
  opacity: 0;
}

.modal-mask.modal-show .modal-expand {
  opacity: 1;
  animation: iosAlertPop .42s cubic-bezier(.18, .89, .32, 1.28) both;
}

@keyframes iosAlertPop {
  0%   { transform: scale(.72); opacity: 0; }
  55%  { transform: scale(1.035); opacity: 1; }
  78%  { transform: scale(.985); opacity: 1; }
  100% { transform: scale(1); opacity: 1; }
}

.modal-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  padding: 18px 20px;
  border-bottom: 1px solid #e6eaf0;
}

.modal-head h3 {
  margin: 0;
  font-size: 18px;
}

.modal-body {
  padding: 16px 20px 20px;
  overflow: auto;
  min-height: 0;
  flex: 1;
}

@media (prefers-reduced-motion: reduce) {
  .modal-mask,
  .modal-expand {
    transition: none !important;
  }
}
</style>
