<script setup>
import { onMounted, ref } from 'vue'
import HttpClient from './views/HttpClient.vue'
import WsClient from './views/WsClient.vue'

const STORAGE_KEY = 'http-client-config'

const activeTab = ref('http')
const persistConfig = ref(false)

onMounted(() => {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    if (saved) persistConfig.value = Boolean(JSON.parse(saved).persistConfig)
  } catch {}
})
</script>

<template>
  <div class="flex h-screen flex-col bg-slate-100 text-slate-800">

    <header class="shrink-0 border-b border-slate-200 bg-white px-6 py-3 shadow-sm">
      <div class="mx-auto flex max-w-[1440px] items-center justify-between">
        <div>
          <h1 class="text-lg font-bold tracking-tight">HTTP 请求测试工具</h1>
          <p class="text-xs text-slate-400">GET / POST · 请求头 · Cookie · 表单 · JSON · CURL 导入 · WebSocket</p>
        </div>
        <label class="flex cursor-pointer items-center gap-2 rounded-full bg-slate-100 px-4 py-1.5 text-sm text-slate-600 ring-1 ring-slate-200 hover:bg-slate-200 transition">
          <input v-model="persistConfig" type="checkbox" class="h-4 w-4">
          持久配置
        </label>
      </div>
    </header>

    <div class="flex-1 overflow-hidden">
      <HttpClient
        v-show="activeTab === 'http'"
        :persist-config="persistConfig"
        @change-tab="activeTab = $event"
        style="height: 100%"
      />
      <WsClient
        v-show="activeTab === 'ws'"
        :persist-config="persistConfig"
        @change-tab="activeTab = $event"
        style="height: 100%"
      />
    </div>

  </div>
</template>
