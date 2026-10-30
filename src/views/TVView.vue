<script setup lang="ts">
import Hls from 'hls.js'
import { storeToRefs } from 'pinia'
import { computed, onBeforeUnmount, ref, watch } from 'vue'
import { useConfigStore } from '../stores/config'

interface Channel {
  name: string
  url: string
  logo: string
  group: string
}

const configStore = useConfigStore()
const { tvConfig } = storeToRefs(configStore)

const channels = ref<Channel[]>([])
const currentChannel = ref<Channel | null>(null)
const loadingPlaylist = ref(false)
const playlistError = ref('')
const playbackError = ref('')
const searchKeyword = ref('')
const listVersion = ref(0)

const videoRef = ref<HTMLVideoElement | null>(null)
let hls: Hls | null = null

const groups = computed(() => {
  // 依赖 listVersion 以便重新加载后强制重算
  void listVersion.value
  const map = new Map<string, Channel[]>()
  for (const ch of channels.value) {
    const key = ch.group || '未分组'
    if (!map.has(key)) map.set(key, [])
    map.get(key)!.push(ch)
  }
  return Array.from(map.entries())
})

const filteredChannels = computed(() => {
  const kw = searchKeyword.value.trim().toLowerCase()
  if (!kw) return channels.value
  return channels.value.filter(ch => ch.name.toLowerCase().includes(kw))
})

const filteredGroups = computed(() => {
  const set = new Set(filteredChannels.value.map(ch => ch.name))
  return groups.value
    .map(([group, list]) => [group, list.filter(ch => set.has(ch.name))] as [string, Channel[]])
    .filter(([, list]) => list.length > 0)
})

function parseM3U(text: string): Channel[] {
  const lines = text.split(/\r?\n/)
  const result: Channel[] = []
  let pending: Partial<Channel> | null = null
  for (const raw of lines) {
    const line = raw.trim()
    if (!line) continue
    if (line.startsWith('#EXTINF')) {
      const name = line.split(',').slice(1).join(',').trim()
      const logo = line.match(/tvg-logo="([^"]*)"/)?.[1] ?? ''
      const group = line.match(/group-title="([^"]*)"/)?.[1] ?? ''
      pending = { name, logo, group }
    }
    else if (!line.startsWith('#')) {
      if (pending && pending.name) {
        result.push({ name: pending.name, url: line, logo: pending.logo ?? '', group: pending.group ?? '' })
      }
      pending = null
    }
  }
  return result
}

async function loadPlaylist() {
  const url = tvConfig.value.m3uUrl.trim()
  if (!url) {
    playlistError.value = '请先在设置中填写 M3U 地址'
    return
  }
  loadingPlaylist.value = true
  playlistError.value = ''
  try {
    const res = await fetch(url)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    const text = await res.text()
    channels.value = parseM3U(text)
    listVersion.value++
    if (channels.value.length === 0) playlistError.value = '播放列表解析结果为空'
  }
  catch (e: any) {
    playlistError.value = `加载失败：${e?.message ?? e}（可能是 CORS 跨域限制，需源站允许访问）`
  }
  finally {
    loadingPlaylist.value = false
  }
}

function play(channel: Channel) {
  playbackError.value = ''
  currentChannel.value = channel
  const video = videoRef.value
  if (!video) return
  destroyHls()

  if (Hls.isSupported() && /\.m3u8(\?|$)/i.test(channel.url)) {
    hls = new Hls({ maxBufferLength: 10 })
    hls.loadSource(channel.url)
    hls.attachMedia(video)
    hls.on(Hls.Events.ERROR, (_evt: string, data: any) => {
      if (data.fatal) playbackError.value = '播放失败，线路可能已失效，请换一个频道试试'
    })
  }
  else {
    video.src = channel.url
  }
  video.play().catch(() => {
    // 浏览器自动播放策略：需要用户先点击页面
  })
}

function destroyHls() {
  if (hls) {
    hls.destroy()
    hls = null
  }
}

watch(() => tvConfig.value.m3uUrl, () => {
  // 地址变化后由设置保存触发重新加载
})

onBeforeUnmount(destroyHls)
</script>

<template>
  <div class="w-full h-full flex flex-col bg-black/60 text-white">
    <div class="flex-1 flex min-h-0">
      <!-- 播放器区域 -->
      <div class="flex-1 flex flex-col min-w-0 p-4 gap-3">
        <div class="relative flex-1 bg-black rounded-2xl overflow-hidden border border-white/10">
          <video ref="videoRef" class="w-full h-full object-contain" playsinline controls />
          <div
            v-if="!currentChannel"
            class="absolute inset-0 flex flex-col items-center justify-center gap-4 text-white/40"
          >
            <div class="text-6xl">📺</div>
            <div class="text-sm tracking-widest">选择右侧频道开始观看</div>
          </div>
          <div
            v-if="playbackError"
            class="absolute inset-x-0 bottom-0 bg-black/80 text-red-300 text-sm px-4 py-3"
          >
            {{ playbackError }}
          </div>
        </div>
        <div v-if="currentChannel" class="flex items-center gap-3 px-1">
          <img v-if="currentChannel.logo" :src="currentChannel.logo" class="h-8 rounded" @error="($event.target as HTMLImageElement).style.display = 'none'">
          <span class="font-medium">{{ currentChannel.name }}</span>
          <span class="text-white/30 text-sm">{{ currentChannel.group }}</span>
        </div>
      </div>

      <!-- 频道列表 -->
      <div class="w-80 lg:w-96 flex flex-col border-l border-white/10 min-h-0">
        <div class="p-3 space-y-2 border-b border-white/10">
          <div class="flex space-x-2">
            <input
              v-model="searchKeyword"
              type="text"
              placeholder="搜索频道..."
              class="settings-input !py-2 text-sm"
            >
            <button
              class="settings-secondary-btn !px-3 whitespace-nowrap"
              :disabled="loadingPlaylist"
              @click="loadPlaylist"
            >
              {{ loadingPlaylist ? '加载中' : '刷新' }}
            </button>
          </div>
          <div v-if="playlistError" class="text-xs text-red-300/80">{{ playlistError }}</div>
          <div v-else-if="channels.length" class="text-xs text-white/30">
            {{ channels.length }} 个频道 · {{ groups.length }} 个分组
          </div>
        </div>
        <div class="flex-1 overflow-y-auto p-3 space-y-4">
          <div v-for="[group, list] in filteredGroups" :key="group">
            <div class="text-xs text-white/40 uppercase tracking-widest mb-2 sticky top-0 backdrop-blur py-1">
              {{ group }}
            </div>
            <div class="space-y-1">
              <button
                v-for="ch in list"
                :key="ch.url"
                class="w-full flex items-center gap-2 px-3 py-2 rounded-xl text-left transition-all border"
                :class="currentChannel?.url === ch.url
                  ? 'bg-white/15 border-white/40 text-white'
                  : 'bg-white/[0.04] border-white/5 text-white/60 hover:bg-white/10 hover:text-white'"
                @click="play(ch)"
              >
                <img
                  v-if="ch.logo"
                  :src="ch.logo"
                  class="h-6 w-9 object-contain rounded flex-shrink-0"
                  @error="($event.target as HTMLImageElement).style.visibility = 'hidden'"
                >
                <span class="text-sm truncate">{{ ch.name }}</span>
              </button>
            </div>
          </div>
          <div v-if="!channels.length && !loadingPlaylist" class="text-center text-white/30 text-sm py-10">
            暂无频道，请在设置的「智能家居」下方填写 M3U 地址
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
