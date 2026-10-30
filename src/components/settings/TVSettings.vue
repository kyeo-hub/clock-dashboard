<script setup lang="ts">
import { RefreshCw, Tv } from 'lucide-vue-next'
import { storeToRefs } from 'pinia'
import { ref } from 'vue'
import { useConfigStore } from '../../stores/config'

const configStore = useConfigStore()
const { tvConfig } = storeToRefs(configStore)

const draft = ref(tvConfig.value.m3uUrl)

function save() {
  tvConfig.value.m3uUrl = draft.value
}

function reset() {
  draft.value = tvConfig.value.m3uUrl
}

defineExpose({ save, reset })
</script>

<template>
  <div class="space-y-4 animate-fade-in">
    <section class="flex flex-col space-y-4">
      <div class="flex items-center justify-between">
        <h4 class="text-white/60 uppercase tracking-widest text-sm font-medium flex items-center">
          <Tv class="w-4 h-4 mr-2" /> 电视直播
        </h4>
        <span class="text-[10px] text-white/30">M3U 播放列表</span>
      </div>

      <div class="space-y-2">
        <label class="block text-sm opacity-50 mb-2 uppercase tracking-widest">M3U 地址</label>
        <div class="flex space-x-2">
          <input
            v-model="draft"
            type="text"
            placeholder="https://example.com/live.m3u"
            class="settings-input"
          >
        </div>
        <p class="text-xs text-white/30 leading-relaxed">
          填写 M3U/M3U8 订阅地址，电视页（最左滑）会自动加载频道列表。
          注意：源站需允许跨域访问（CORS），否则浏览器无法拉取列表。
        </p>
      </div>

      <button
        v-if="draft"
        class="settings-secondary-btn w-full justify-center"
        @click="() => { save(); }"
      >
        <RefreshCw class="w-4 h-4" /> 保存后到电视页点击「刷新」加载
      </button>
    </section>
  </div>
</template>
