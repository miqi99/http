<script setup>
import { computed, onMounted, ref, watch } from 'vue'

// 本地持久化存储 key
const STORAGE_KEY = 'http-client-config'
const HISTORY_KEY = 'http-client-history'

// 是否启用持久配置
const persistConfig = ref(false)

// 请求地址
const url = ref('')

// 请求方式：GET / POST
const method = ref('GET')

// 访问模式：direct = 本地直连，proxy = 代理转发
const accessMode = ref('direct')

// 参数类型：none = 无参数，form = 表单，json = JSON
const paramType = ref('none')

// 请求头，最少保留一行，可不填写
const headers = ref([
  { key: '', value: '' }
])

// Cookie，单独输入，最终会作为 Cookie 请求头发送
const cookie = ref('')
const cookieEnabled = ref(false)

// 表单参数，最少保留一行，可不填写
const formParams = ref([
  { key: '', value: '' }
])

// JSON 参数：用户自己复制 JSON 内容进来
const jsonBody = ref('')

// 是否格式化 JSON 参数
const formatRequestJson = ref(false)

// 返回结果原始内容
const response = ref('')

// 请求响应信息
const responseInfo = ref({
  status: '',
  statusText: '',
  time: '',
  size: '',
  contentType: '',
  url: ''
})

// 底部卡片：body = 返回结果，info = 请求响应
const responseTab = ref('body')

// 请求状态
const loading = ref(false)

// 是否超时
const timeout = ref(false)

// 是否格式化返回 JSON
const formatResponseJson = ref(true)

// CURL 导入弹窗
const showCurlModal = ref(false)
const curlText = ref('')
const curlError = ref('')
const historyList = ref([])
const showHistoryModal = ref(false)
const copyTip = ref('')

// 是否完成初始化，避免刚打开页面时覆盖持久配置
const initialized = ref(false)

// URL 合法性校验
const urlError = computed(() => {
  const raw = url.value.trim()
  if (!raw) return ''
  try {
    new URL(raw)
    return ''
  } catch {
    return '地址格式不正确，需要包含协议，例如 https://api.example.com'
  }
})

// 发送按钮文字
const buttonText = computed(() => {
  if (loading.value) return '请求中...'
  if (timeout.value) return '已超时 请重试'
  return '发送请求'
})

// JSON 参数展示内容
const displayRequestJson = computed(() => {
  if (!formatRequestJson.value) {
    return jsonBody.value
  }

  try {
    return JSON.stringify(JSON.parse(jsonBody.value), null, 2)
  } catch {
    return jsonBody.value
  }
})

// 返回结果展示内容
const displayResponse = computed(() => {
  if (!formatResponseJson.value) {
    return response.value
  }

  try {
    return JSON.stringify(JSON.parse(response.value), null, 2)
  } catch {
    return response.value
  }
})

const statusColorClass = computed(() => {
  const status = String(responseInfo.value.status || '')

  if (status.startsWith('2')) {
    return 'text-green-600'
  }

  if (status.startsWith('3')) {
    return 'text-blue-600'
  }

  if (status.startsWith('4')) {
    return 'text-orange-600'
  }

  if (status.startsWith('5')) {
    return 'text-red-600'
  }

  if (status === '请求失败') {
    return 'text-red-600'
  }

  return 'text-slate-900'
})

// 页面打开时恢复持久配置
onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)

  if (saved) {
    try {
      const config = JSON.parse(saved)

      persistConfig.value = Boolean(config.persistConfig)
      url.value = config.url || ''
      method.value = config.method || 'GET'
      accessMode.value = config.accessMode || 'direct'
      paramType.value = config.paramType || 'none'

      headers.value = Array.isArray(config.headers) && config.headers.length > 0
        ? config.headers
        : [{ key: '', value: '' }]

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

// 监听配置变化，勾选“持久配置”后自动保存
watch(
  [
    persistConfig,
    url,
    method,
    accessMode,
    paramType,
    headers,
    cookie,
    cookieEnabled,
    formParams,
    jsonBody,
    formatRequestJson,
    formatResponseJson
  ],
  () => {
    if (!initialized.value) {
      return
    }

    if (!persistConfig.value) {
      localStorage.removeItem(STORAGE_KEY)
      return
    }

    const config = {
      persistConfig: persistConfig.value,
      url: url.value,
      method: method.value,
      accessMode: accessMode.value,
      paramType: paramType.value,
      headers: headers.value,
      cookie: cookie.value,
      cookieEnabled: cookieEnabled.value,
      formParams: formParams.value,
      jsonBody: jsonBody.value,
      formatRequestJson: formatRequestJson.value,
      formatResponseJson: formatResponseJson.value
    }

    localStorage.setItem(STORAGE_KEY, JSON.stringify(config))
  },
  { deep: true }
)

// 添加请求头
function addHeader() {
  headers.value.push({
    key: '',
    value: ''
  })
}

function removeHeader(index) {
  headers.value.splice(index, 1)
}

// 添加表单参数
function addFormParam() {
  formParams.value.push({
    key: '',
    value: ''
  })
}

// 删除表单参数，最少保留一项
function removeFormParam(index) {
  if (formParams.value.length > 1) {
    formParams.value.splice(index, 1)
  }
}

// 手动格式化 JSON 参数
function applyRequestJsonFormat() {
  if (!formatRequestJson.value) {
    return
  }

  try {
    jsonBody.value = JSON.stringify(JSON.parse(jsonBody.value), null, 2)
  } catch {
    // JSON 不合法时不修改用户输入
  }
}

// 判断用户是否手动填写了某个请求头
function hasHeader(headersObject, headerName) {
  return Object.keys(headersObject).some(
    key => key.toLowerCase() === headerName.toLowerCase()
  )
}

// 生成请求头对象
function buildHeaders() {
  const requestHeaders = {}

  headers.value.forEach(item => {
    const key = item.key.trim()

    if (key) {
      requestHeaders[key] = item.value
    }
  })

  // Cookie 单独输入，最终转为 Cookie 请求头（仅开关开启时生效）
  if (cookieEnabled.value && cookie.value.trim()) {
    requestHeaders.Cookie = cookie.value.trim()
  }

  // 自动 Content-Type：
  // GET 不设置
  // POST + 无参数 不设置
  // POST + JSON 自动 application/json; charset=utf-8
  // POST + 表单 自动 application/x-www-form-urlencoded; charset=utf-8
  // 如果用户自己写了 Content-Type，则优先使用用户填写的
  if (!hasHeader(requestHeaders, 'Content-Type') && method.value === 'POST') {
    if (paramType.value === 'json') {
      requestHeaders['Content-Type'] = 'application/json; charset=utf-8'
    }

    if (paramType.value === 'form') {
      requestHeaders['Content-Type'] = 'application/x-www-form-urlencoded; charset=utf-8'
    }
  }

  return requestHeaders
}

// 生成请求 body
function buildBody() {
  if (method.value !== 'POST') {
    return undefined
  }

  if (paramType.value === 'json') {
    return jsonBody.value
  }

  if (paramType.value === 'form') {
    const formData = new URLSearchParams()

    formParams.value.forEach(item => {
      const key = item.key.trim()

      if (key) {
        formData.append(key, item.value)
      }
    })

    return formData
  }

  return undefined
}

// 计算响应大小
function formatSize(text) {
  const bytes = new Blob([text]).size

  if (bytes < 1024) {
    return `${bytes} B`
  }

  return `${(bytes / 1024).toFixed(2)} KB`
}

// 发送前校验 JSON
function validateJsonBeforeSend() {
  if (method.value !== 'POST') {
    return true
  }

  if (paramType.value !== 'json') {
    return true
  }

  if (!jsonBody.value.trim()) {
    response.value = 'JSON 参数不能为空'
    responseTab.value = 'body'
    return false
  }

  try {
    JSON.parse(jsonBody.value)
    return true
  } catch (err) {
    response.value = `JSON 格式错误，已阻止发送：\n${err.message}`
    responseTab.value = 'body'
    return false
  }
}

// 保存请求历史，最多保留最近 20 条
function saveHistoryEntry() {
  const entry = {
    time: new Date().toLocaleString(),
    url: url.value,
    method: method.value,
    accessMode: accessMode.value,
    paramType: paramType.value,
    headers: JSON.parse(JSON.stringify(headers.value)),
    cookie: cookie.value,
    formParams: JSON.parse(JSON.stringify(formParams.value)),
    jsonBody: jsonBody.value
  }

  historyList.value = [
    entry,
    ...historyList.value.filter(item => !(item.url === entry.url && item.method === entry.method))
  ].slice(0, 20)

  localStorage.setItem(HISTORY_KEY, JSON.stringify(historyList.value))
}

// 从历史记录恢复请求配置
function restoreHistory(index) {
  const entry = historyList.value[index]

  if (!entry) {
    return
  }

  url.value = entry.url || ''
  method.value = entry.method || 'GET'
  accessMode.value = entry.accessMode || 'direct'
  paramType.value = entry.paramType || 'none'
  headers.value = Array.isArray(entry.headers) && entry.headers.length > 0
    ? entry.headers
    : [{ key: '', value: '' }]
  cookie.value = entry.cookie || ''
  cookieEnabled.value = Boolean(entry.cookie)
  formParams.value = Array.isArray(entry.formParams) && entry.formParams.length > 0
    ? entry.formParams
    : [{ key: '', value: '' }]
  jsonBody.value = entry.jsonBody || '{\n  \n}'
  response.value = ''
  responseTab.value = 'body'
  showHistoryModal.value = false
}

// 清空历史记录
function clearHistory() {
  historyList.value = []
  localStorage.removeItem(HISTORY_KEY)
}

// 一键清空当前输入和返回结果
function clearAll() {
  url.value = ''
  method.value = 'GET'
  accessMode.value = 'direct'
  paramType.value = 'none'
  headers.value = [{ key: '', value: '' }]
  cookie.value = ''
  cookieEnabled.value = false
  formParams.value = [{ key: '', value: '' }]
  jsonBody.value = '{\n  \n}'
  formatRequestJson.value = false
  response.value = ''
  responseTab.value = 'body'
  responseInfo.value = {
    status: '',
    statusText: '',
    time: '',
    size: '',
    contentType: '',
    url: ''
  }
  timeout.value = false
}

// 复制返回结果
async function copyResponse() {
  if (!response.value) {
    copyTip.value = '暂无内容'
    setTimeout(() => {
      copyTip.value = ''
    }, 1200)
    return
  }

  await navigator.clipboard.writeText(response.value)
  copyTip.value = '已复制'

  setTimeout(() => {
    copyTip.value = ''
  }, 1200)
}

// 打开 CURL 导入弹窗
function openCurlModal() {
  curlText.value = ''
  curlError.value = ''
  showCurlModal.value = true
}

// 关闭 CURL 导入弹窗
function closeCurlModal() {
  showCurlModal.value = false
}

// 简单解析 shell 字符串，支持单引号、双引号和空格
function splitCurlCommand(command) {
  const result = []
  let current = ''
  let quote = ''
  let escaping = false

  for (const char of command) {
    if (escaping) {
      current += char
      escaping = false
      continue
    }

    if (char === '\\') {
      escaping = true
      continue
    }

    if (quote) {
      if (char === quote) {
        quote = ''
      } else {
        current += char
      }
      continue
    }

    if (char === '"' || char === "'") {
      quote = char
      continue
    }

    if (/\s/.test(char)) {
      if (current) {
        result.push(current)
        current = ''
      }
      continue
    }

    current += char
  }

  if (current) {
    result.push(current)
  }

  return result
}

// 把 a=b&c=d 转成表单参数
function parseFormString(text) {
  const params = new URLSearchParams(text)
  const rows = []

  params.forEach((value, key) => {
    rows.push({
      key,
      value
    })
  })

  return rows.length > 0 ? rows : [{ key: '', value: '' }]
}

// 导入 CURL
function importCurl() {
  curlError.value = ''

  const raw = curlText.value.trim()

  if (!raw) {
    curlError.value = '请先粘贴 CURL 内容'
    return
  }

  try {
    const normalized = raw
      .replace(/\\\r?\n/g, ' ')
      .replace(/\r?\n/g, ' ')

    const tokens = splitCurlCommand(normalized)

    if (!tokens.length || tokens[0] !== 'curl') {
      curlError.value = '请输入以 curl 开头的命令'
      return
    }

    const nextHeaders = []
    let nextCookie = ''
    let nextMethod = ''
    let nextBody = ''
    let nextUrl = ''

    for (let i = 1; i < tokens.length; i += 1) {
      const token = tokens[i]

      if (token === '-X' || token === '--request') {
        nextMethod = (tokens[i + 1] || '').toUpperCase()
        i += 1
        continue
      }

      if (token === '-H' || token === '--header') {
        const headerLine = tokens[i + 1] || ''
        const colonIndex = headerLine.indexOf(':')

        if (colonIndex > -1) {
          const key = headerLine.slice(0, colonIndex).trim()
          const value = headerLine.slice(colonIndex + 1).trim()

          if (key.toLowerCase() === 'cookie') {
            nextCookie = value
          } else {
            nextHeaders.push({
              key,
              value
            })
          }
        }

        i += 1
        continue
      }

      if (
        token === '-d' ||
        token === '--data' ||
        token === '--data-raw' ||
        token === '--data-binary' ||
        token === '--data-urlencode'
      ) {
        nextBody = tokens[i + 1] || ''
        i += 1
        continue
      }

      if (!token.startsWith('-') && !nextUrl) {
        nextUrl = token
      }
    }

    if (!nextUrl) {
      curlError.value = '没有解析到请求地址'
      return
    }

    url.value = nextUrl
    cookie.value = nextCookie
    headers.value = nextHeaders.length > 0 ? nextHeaders : [{ key: '', value: '' }]

    const contentTypeHeader = nextHeaders.find(
      item => item.key.toLowerCase() === 'content-type'
    )

    const contentType = contentTypeHeader?.value?.toLowerCase() || ''

    if (nextBody) {
      method.value = nextMethod || 'POST'

      if (contentType.includes('application/json') || nextBody.trim().startsWith('{') || nextBody.trim().startsWith('[')) {
        paramType.value = 'json'
        jsonBody.value = nextBody

        try {
          jsonBody.value = JSON.stringify(JSON.parse(nextBody), null, 2)
          formatRequestJson.value = true
        } catch {
          formatRequestJson.value = false
        }
      } else {
        paramType.value = 'form'
        formParams.value = parseFormString(nextBody)
      }
    } else {
      method.value = nextMethod || 'GET'
      paramType.value = 'none'
    }

    showCurlModal.value = false
  } catch (err) {
    curlError.value = err.message || String(err)
  }
}

// 发送请求
async function sendRequest() {
  if (loading.value) {
    return
  }

  if (!url.value.trim()) {
    response.value = '请先输入请求地址'
    responseTab.value = 'body'
    return
  }

  if (urlError.value) {
    response.value = urlError.value
    responseTab.value = 'body'
    return
  }

  if (!validateJsonBeforeSend()) {
    return
  }

  loading.value = true
  timeout.value = false
  response.value = '请求中...'
  responseTab.value = 'body'

  responseInfo.value = {
    status: '',
    statusText: '',
    time: '',
    size: '',
    contentType: '',
    url: ''
  }

  const startTime = Date.now()

  // 5 秒超时提示：按钮恢复可点击
  const timer = setTimeout(() => {
    loading.value = false
    timeout.value = true
    response.value = '请求超过 5 秒，请重试'
    responseInfo.value = {
      ...responseInfo.value,
      time: `${Date.now() - startTime} ms`,
      size: formatSize(response.value)
    }
  }, 5000)

  try {
    const requestHeaders = buildHeaders()
    const body = buildBody()

    let requestUrl = url.value.trim()

    if (method.value !== 'POST' && paramType.value === 'form') {
      const params = new URLSearchParams()
      formParams.value.forEach(item => {
        const key = item.key.trim()
        if (key) params.append(key, item.value)
      })
      const qs = params.toString()
      if (qs) {
        requestUrl += (requestUrl.includes('?') ? '&' : '?') + qs
      }
    }

    // 代理转发：下一步接 Node.js 的 http://localhost:3000/proxy
    if (accessMode.value === 'proxy') {
      clearTimeout(timer)
      loading.value = false
      timeout.value = false
      response.value = '代理转发功能还未接入。\n下一步需要启动本地 Node.js 代理服务：http://localhost:3000/proxy'
      responseInfo.value = {
        status: '未发送',
        statusText: '代理未接入',
        time: `${Date.now() - startTime} ms`,
        size: formatSize(response.value),
        contentType: '',
        url: 'http://localhost:3000/proxy'
      }
      return
    }

    const res = await fetch(requestUrl, {
      method: method.value,
      headers: requestHeaders,
      body
    })

    const text = await res.text()

    clearTimeout(timer)
    loading.value = false
    timeout.value = false
    response.value = text

    responseInfo.value = {
      status: String(res.status),
      statusText: res.statusText || '',
      time: `${Date.now() - startTime} ms`,
      size: formatSize(text),
      contentType: res.headers.get('content-type') || '',
      url: res.url || url.value.trim()
    }

    saveHistoryEntry()
  } catch (err) {
    clearTimeout(timer)
    loading.value = false
    timeout.value = false
    response.value = err.message || String(err)

    responseInfo.value = {
      status: '请求失败',
      statusText: err.name || '',
      time: `${Date.now() - startTime} ms`,
      size: formatSize(response.value),
      contentType: '',
      url: url.value.trim()
    }
  }
}
</script>

<template>
  <div class="flex h-screen flex-col bg-slate-100 text-slate-800">

    <!-- 顶部 Header -->
    <header class="shrink-0 border-b border-slate-200 bg-white px-6 py-3 shadow-sm">
      <div class="mx-auto flex max-w-[1440px] items-center justify-between">
        <div>
          <h1 class="text-lg font-bold tracking-tight">HTTP 请求测试工具</h1>
          <p class="text-xs text-slate-400">GET / POST · 请求头 · Cookie · 表单 · JSON · CURL 导入</p>
        </div>
        <label class="flex cursor-pointer items-center gap-2 rounded-full bg-slate-100 px-4 py-2 text-sm text-slate-600 ring-1 ring-slate-200 hover:bg-slate-200 transition">
          <input v-model="persistConfig" type="checkbox" class="h-4 w-4">
          持久配置
        </label>
      </div>
    </header>

    <!-- 主体双栏布局 -->
    <div class="flex-1 overflow-hidden">
      <div class="mx-auto flex h-full max-w-[1440px] gap-4 px-4 py-4">

        <!-- ===== 左栏：工具 + 响应结果 ===== -->
        <div class="flex h-full w-[420px] shrink-0 flex-col gap-3">

          <!-- 工具按钮 -->
          <div class="shrink-0 rounded-2xl bg-white p-4 shadow-sm ring-1 ring-slate-200">
            <p class="mb-3 text-xs font-semibold uppercase tracking-widest text-slate-400">工具</p>
            <div class="grid grid-cols-3 gap-2">
              <button
                type="button"
                @click="openCurlModal"
                class="rounded-xl bg-slate-900 px-3 py-2.5 text-xs font-semibold text-white hover:bg-slate-700 transition"
              >
                CURL 导入
              </button>
              <button
                type="button"
                @click="showHistoryModal = true"
                class="rounded-xl bg-white px-3 py-2.5 text-xs font-semibold text-slate-700 ring-1 ring-slate-200 hover:bg-slate-50 transition"
              >
                请求历史
              </button>
              <button
                type="button"
                @click="clearAll"
                class="rounded-xl bg-white px-3 py-2.5 text-xs font-semibold text-red-500 ring-1 ring-red-100 hover:bg-red-50 transition"
              >
                一键清空
              </button>
            </div>
          </div>

          <!-- 响应结果（撑满剩余高度） -->
          <div class="flex min-h-0 flex-1 flex-col rounded-2xl bg-white p-4 shadow-sm ring-1 ring-slate-200">

            <!-- Tab 切换 + 操作按钮 -->
            <div class="mb-3 flex shrink-0 items-center justify-between gap-2">
              <div class="grid grid-cols-2 gap-1 rounded-xl bg-slate-100 p-1">
                <button
                  type="button"
                  @click="responseTab = 'body'"
                  :class="responseTab === 'body' ? 'bg-white text-slate-900 shadow-sm' : 'text-slate-500 hover:text-slate-700'"
                  class="rounded-lg px-3 py-1.5 text-xs font-semibold transition"
                >
                  返回结果
                </button>
                <button
                  type="button"
                  @click="responseTab = 'info'"
                  :class="responseTab === 'info' ? 'bg-white text-slate-900 shadow-sm' : 'text-slate-500 hover:text-slate-700'"
                  class="rounded-lg px-3 py-1.5 text-xs font-semibold transition"
                >
                  请求响应
                </button>
              </div>

              <div v-if="responseTab === 'body'" class="flex shrink-0 items-center gap-2">
                <span v-if="copyTip" class="text-xs text-slate-500">{{ copyTip }}</span>
                <button
                  type="button"
                  @click="copyResponse"
                  class="rounded-lg bg-slate-900 px-3 py-1.5 text-xs font-semibold text-white hover:bg-slate-700 transition"
                >
                  复制
                </button>
                <label class="flex items-center gap-1.5 text-xs text-slate-600 cursor-pointer">
                  <input v-model="formatResponseJson" type="checkbox" class="h-3.5 w-3.5">
                  格式化
                </label>
              </div>
            </div>

            <!-- 返回 body -->
            <textarea
              v-if="responseTab === 'body'"
              :value="displayResponse"
              readonly
              class="min-h-0 flex-1 w-full resize-none rounded-xl border border-slate-300 bg-slate-950 p-4 font-mono text-sm text-slate-100 outline-none"
              placeholder="暂无返回结果"
            />

            <!-- 请求响应信息 -->
            <div
              v-else
              class="min-h-0 flex-1 overflow-y-auto rounded-xl border border-slate-300 bg-slate-50 p-3"
            >
              <div class="grid grid-cols-2 gap-2">
                <div class="rounded-lg bg-white p-3 ring-1 ring-slate-200">
                  <div class="text-xs text-slate-500">状态码</div>
                  <div :class="statusColorClass" class="mt-1 font-mono text-sm font-semibold">
                    {{ responseInfo.status || '-' }}
                  </div>
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

        <!-- ===== 右栏：请求构建器 ===== -->
        <div class="flex-1 overflow-y-auto">
          <div class="rounded-2xl bg-white p-6 shadow-sm ring-1 ring-slate-200">

            <!-- 请求地址 -->
            <div class="mb-5">
              <label class="mb-2 block text-sm font-medium">请求地址</label>
              <input
                v-model="url"
                :class="urlError
                  ? 'border-red-400 focus:border-red-400 focus:ring-red-100'
                  : 'border-slate-300 focus:border-blue-500 focus:ring-blue-100'"
                class="w-full rounded-xl border bg-white px-4 py-3 text-sm outline-none transition focus:ring-4"
                placeholder="https://api.example.com"
              >
              <p v-if="urlError" class="mt-1.5 text-xs text-red-500">{{ urlError }}</p>
            </div>

            <!-- 请求方式 + 访问模式 -->
            <div class="mb-5 grid gap-4 md:grid-cols-2">
              <div>
                <label class="mb-2 block text-sm font-medium">请求方式</label>
                <div class="grid grid-cols-2 gap-2">
                  <button
                    type="button"
                    @click="method = 'GET'"
                    :class="method === 'GET' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-2 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                    class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                  >GET</button>
                  <button
                    type="button"
                    @click="method = 'POST'"
                    :class="method === 'POST' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-2 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                    class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                  >POST</button>
                </div>
              </div>
              <div>
                <label class="mb-2 block text-sm font-medium">访问模式</label>
                <div class="grid grid-cols-2 gap-2">
                  <button
                    type="button"
                    @click="accessMode = 'direct'"
                    :class="accessMode === 'direct' ? 'border-emerald-600 bg-emerald-50 text-emerald-700 ring-2 ring-emerald-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                    class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                  >本地直连</button>
                  <button
                    type="button"
                    @click="accessMode = 'proxy'"
                    :class="accessMode === 'proxy' ? 'border-emerald-600 bg-emerald-50 text-emerald-700 ring-2 ring-emerald-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                    class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                  >代理转发</button>
                </div>
              </div>
            </div>

            <!-- 请求头 -->
            <div class="mb-5">
              <div class="mb-3 flex items-center justify-between gap-4">
                <div>
                  <h3 class="text-sm font-semibold">请求头</h3>
                  <p class="text-xs text-slate-500">可不填写 用户填写 Content-Type 时会优先生效。</p>
                </div>
                <button
                  type="button"
                  @click="addHeader"
                  class="shrink-0 rounded-lg bg-slate-900 px-3 py-2 text-sm text-white hover:bg-slate-700 transition"
                >添加请求头</button>
              </div>
              <div class="space-y-2">
                <div
                  v-for="(item, index) in headers"
                  :key="index"
                  class="grid grid-cols-[1fr_1fr_auto] gap-2"
                >
                  <input
                    v-model="item.key"
                    class="rounded-lg border border-slate-300 px-3 py-2 text-sm outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
                    placeholder="Header Key"
                  >
                  <input
                    v-model="item.value"
                    class="rounded-lg border border-slate-300 px-3 py-2 text-sm outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
                    placeholder="Header Value"
                  >
                  <button
                    type="button"
                    @click="removeHeader(index)"
                    class="rounded-lg border border-slate-300 px-3 text-sm text-slate-500 hover:bg-slate-100"
                  >删除</button>
                </div>
              </div>
            </div>

            <!-- Cookie -->
            <div class="mb-5">
              <div class="mb-2 flex items-center justify-between">
                <label class="text-sm font-medium">Cookie</label>
                <button
                  type="button"
                  @click="cookieEnabled = !cookieEnabled"
                  :class="cookieEnabled
                    ? 'bg-blue-600 text-white hover:bg-blue-500'
                    : 'bg-slate-100 text-slate-500 hover:bg-slate-200'"
                  class="rounded-full px-3 py-1 text-xs font-semibold transition"
                >
                  {{ cookieEnabled ? '已开启' : '已关闭' }}
                </button>
              </div>
              <textarea
                v-if="cookieEnabled"
                v-model="cookie"
                rows="3"
                class="w-full rounded-xl border border-slate-300 bg-white p-3 text-sm outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
                placeholder="token=123; uid=888"
              />
            </div>

            <!-- 参数类型 -->
            <div class="mb-5">
              <label class="mb-2 block text-sm font-medium">参数类型</label>
              <div class="grid grid-cols-3 gap-2">
                <button
                  type="button"
                  @click="paramType = 'none'"
                  :class="paramType === 'none' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-2 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                  class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                >无参数</button>
                <button
                  type="button"
                  @click="paramType = 'form'"
                  :class="paramType === 'form' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-2 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                  class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                >表单</button>
                <button
                  type="button"
                  @click="paramType = 'json'"
                  :class="paramType === 'json' ? 'border-blue-600 bg-blue-50 text-blue-700 ring-2 ring-blue-100' : 'border-slate-300 bg-white text-slate-600 hover:bg-slate-50'"
                  class="rounded-xl border px-4 py-3 text-sm font-semibold transition"
                >JSON</button>
              </div>
            </div>

            <!-- 表单参数 -->
            <div v-if="paramType === 'form'" class="mb-5">
              <div class="mb-3 flex items-center justify-between gap-4">
                <div>
                  <h3 class="text-sm font-semibold">表单参数</h3>
                  <p class="text-xs text-slate-500">按 key / value 添加，多行参数会自动转成表单提交</p>
                </div>
                <button
                  type="button"
                  @click="addFormParam"
                  class="shrink-0 rounded-lg bg-slate-900 px-3 py-2 text-sm text-white hover:bg-slate-700 transition"
                >添加参数</button>
              </div>
              <div class="space-y-2">
                <div
                  v-for="(item, index) in formParams"
                  :key="index"
                  class="grid grid-cols-[1fr_1fr_auto] gap-2"
                >
                  <input
                    v-model="item.key"
                    class="rounded-lg border border-slate-300 px-3 py-2 text-sm outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
                    placeholder="参数 Key"
                  >
                  <input
                    v-model="item.value"
                    class="rounded-lg border border-slate-300 px-3 py-2 text-sm outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
                    placeholder="参数 Value"
                  >
                  <button
                    type="button"
                    @click="removeFormParam(index)"
                    :disabled="formParams.length === 1"
                    :class="formParams.length === 1 ? 'cursor-not-allowed text-slate-300' : 'text-slate-500 hover:bg-slate-100'"
                    class="rounded-lg border border-slate-300 px-3 text-sm"
                  >删除</button>
                </div>
              </div>
            </div>

            <!-- JSON 参数 -->
            <div v-if="paramType === 'json'" class="mb-5">
              <div class="mb-2 flex items-center justify-between gap-4">
                <label class="block text-sm font-medium">JSON 参数</label>
                <label class="flex shrink-0 items-center gap-2 text-sm text-slate-600 cursor-pointer">
                  <input
                    v-model="formatRequestJson"
                    type="checkbox"
                    class="h-4 w-4"
                    @change="applyRequestJsonFormat"
                  >
                  格式化 JSON
                </label>
              </div>
              <textarea
                :value="displayRequestJson"
                rows="10"
                class="w-full rounded-xl border border-slate-300 bg-slate-950 p-4 font-mono text-sm text-slate-100 outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
                placeholder="{&#10;  &quot;name&quot;: &quot;test&quot;&#10;}"
                @input="jsonBody = $event.target.value"
              />
            </div>

            <!-- 发送请求 -->
            <button
              type="button"
              @click="sendRequest"
              :disabled="loading"
              :class="loading
                ? 'cursor-not-allowed bg-slate-400'
                : timeout
                  ? 'bg-orange-500 hover:bg-orange-400'
                  : 'bg-blue-600 hover:bg-blue-500'"
              class="w-full rounded-xl px-5 py-3 text-sm font-semibold text-white shadow-sm transition"
            >
              {{ buttonText }}
            </button>

          </div>
        </div>

      </div>
    </div>

    <!-- 请求历史弹窗 -->
    <div
      v-if="showHistoryModal"
      class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/50 px-4"
    >
      <div class="w-full max-w-4xl rounded-2xl bg-white p-6 shadow-xl">
        <div class="mb-4 flex items-center justify-between gap-4">
          <div>
            <h2 class="text-lg font-semibold">请求历史</h2>
            <p class="mt-1 text-sm text-slate-500">最近 20 条请求记录，点击恢复可填回当前页面</p>
          </div>
          <button
            type="button"
            @click="showHistoryModal = false"
            class="rounded-lg border border-slate-300 px-3 py-2 text-sm text-slate-600 hover:bg-slate-100"
          >关闭</button>
        </div>
        <div class="mb-4 flex justify-end">
          <button
            type="button"
            @click="clearHistory"
            class="rounded-lg bg-red-500 px-3 py-2 text-sm font-semibold text-white hover:bg-red-400"
          >清空历史</button>
        </div>
        <div
          v-if="historyList.length === 0"
          class="rounded-xl border border-dashed border-slate-300 p-8 text-center text-sm text-slate-500"
        >暂无请求历史</div>
        <div v-else class="max-h-[520px] space-y-2 overflow-y-auto">
          <div
            v-for="(item, index) in historyList"
            :key="index"
            class="rounded-xl border border-slate-200 bg-slate-50 p-4"
          >
            <div class="flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
              <div class="min-w-0">
                <div class="mb-1 flex items-center gap-2">
                  <span
                    :class="item.method === 'POST' ? 'bg-blue-100 text-blue-700' : 'bg-emerald-100 text-emerald-700'"
                    class="rounded-full px-2 py-1 text-xs font-semibold"
                  >{{ item.method }}</span>
                  <span class="rounded-full bg-slate-200 px-2 py-1 text-xs text-slate-600">{{ item.paramType }}</span>
                  <span class="text-xs text-slate-400">{{ item.time }}</span>
                </div>
                <div class="truncate font-mono text-sm text-slate-800">{{ item.url }}</div>
              </div>
              <button
                type="button"
                @click="restoreHistory(index)"
                class="shrink-0 rounded-lg bg-blue-600 px-3 py-2 text-sm font-semibold text-white hover:bg-blue-500"
              >恢复</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- CURL 导入弹窗 -->
    <div
      v-if="showCurlModal"
      class="fixed inset-0 z-50 flex items-center justify-center bg-slate-900/50 px-4"
    >
      <div class="w-full max-w-3xl rounded-2xl bg-white p-6 shadow-xl">
        <div class="mb-4 flex items-center justify-between gap-4">
          <div>
            <h2 class="text-lg font-semibold">CURL 导入</h2>
            <p class="mt-1 text-sm text-slate-500">粘贴 curl 命令后，会自动填充 URL、请求方式、请求头、Cookie 和参数</p>
          </div>
          <button
            type="button"
            @click="closeCurlModal"
            class="rounded-lg border border-slate-300 px-3 py-2 text-sm text-slate-600 hover:bg-slate-100"
          >关闭</button>
        </div>
        <textarea
          v-model="curlText"
          rows="12"
          class="w-full rounded-xl border border-slate-300 bg-slate-950 p-4 font-mono text-sm text-slate-100 outline-none focus:border-blue-500 focus:ring-4 focus:ring-blue-100"
          placeholder="curl 'https://api.example.com/user' -H 'Authorization: Bearer xxx' -H 'Content-Type: application/json' --data '{&quot;id&quot;:1}'"
        />
        <p v-if="curlError" class="mt-3 rounded-lg bg-red-50 px-3 py-2 text-sm text-red-600">{{ curlError }}</p>
        <div class="mt-5 flex justify-end gap-3">
          <button
            type="button"
            @click="closeCurlModal"
            class="rounded-xl border border-slate-300 px-5 py-2 text-sm font-semibold text-slate-600 hover:bg-slate-100"
          >取消</button>
          <button
            type="button"
            @click="importCurl"
            class="rounded-xl bg-blue-600 px-5 py-2 text-sm font-semibold text-white hover:bg-blue-500"
          >导入</button>
        </div>
      </div>
    </div>

  </div>
</template>
