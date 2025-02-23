<script setup lang="ts">
import type { Ref } from 'vue'
import { useRouteQuery } from '@vueuse/router'
import { Pane, Splitpanes } from 'splitpanes'
import { msToTime } from '../../logic/utils'
import { enableDiff, inspectSSR, lineWrapping, onRefetch, showOneColumn } from '../../logic'
import { rpc } from '../../logic/rpc'
import type { HMRData } from '../../../types'
import { getHot } from '../../logic/hot'

const id = useRouteQuery<string | undefined>('id')
const data = ref(id.value ? await rpc.getIdInfo(id.value, inspectSSR.value) : undefined)
const index = useRouteQuery('index') as Ref<string>
const currentIndex = computed(() => +index.value ?? (data.value?.transforms.length || 1) - 1 ?? 0)
const panelSize = useLocalStorage('vite-inspect-module-panel-size', '10')

async function refetch() {
  if (id.value)
    data.value = await rpc.getIdInfo(id.value, inspectSSR.value, true)
}

onRefetch.on(async () => {
  await refetch()
})

watch([id, inspectSSR], refetch)

const from = computed(() => data.value?.transforms[currentIndex.value - 1]?.result || '')
const to = computed(() => data.value?.transforms[currentIndex.value]?.result || '')

getHot().then((hot) => {
  if (hot) {
    hot.on('vite-plugin-inspect:update', ({ ids }: HMRData) => {
      if (id.value && ids.includes(id.value))
        refetch()
    })
  }
})
</script>

<template>
  <NavBar>
    <router-link class="icon-btn !outline-none my-auto" to="/">
      <carbon-arrow-left />
    </router-link>
    <ModuleId v-if="id" :id="id" />
    <Badge
      v-if="inspectSSR"
      class="bg-teal-400/10 text-teal-400 font-bold"
    >
      SSR
    </Badge>
    <div class="flex-auto" />

    <button class="icon-btn text-lg" title="Inspect SSR" @click="inspectSSR = !inspectSSR">
      <carbon:cloud-services :class="inspectSSR ? 'opacity-100' : 'opacity-25'" />
    </button>
    <button class="icon-btn text-lg" title="Line Wrapping" @click="lineWrapping = !lineWrapping">
      <carbon:text-wrap :class="lineWrapping ? 'opacity-100' : 'opacity-25'" />
    </button>
    <button class="icon-btn text-lg" title="Toggle one column" @click="showOneColumn = !showOneColumn">
      <carbon-side-panel-open v-if="showOneColumn" />
      <carbon-side-panel-close v-else />
    </button>
    <button class="icon-btn text-lg" title="Toggle Diff" @click="enableDiff = !enableDiff">
      <carbon:compare :class="enableDiff ? 'opacity-100' : 'opacity-25'" />
    </button>
  </NavBar>
  <Container
    v-if="data && data.transforms"
    class="flex overflow-hidden"
  >
    <Splitpanes class="h-full of-hidden" @resize="panelSize = $event[0].size">
      <Pane :size="panelSize" min-size="10" class="flex flex-col border-r border-main overflow-y-auto">
        <div
          class="border-b border-main px-3 py-2 text-center text-sm tracking-widest text-gray-400"
        >
          {{ inspectSSR ? 'SSR ' : '' }}TRANSFORM STACK
        </div>
        <template v-for="tr, idx of data.transforms" :key="tr.name">
          <button
            class="border-b border-main px-2 py-2 text-left font-mono text-xs !outline-none"
            flex="~ gap-1 wrap" items-center
            :class="
              currentIndex === idx
                ? 'bg-main bg-opacity-10'
                : tr.result === data.transforms[idx - 1]?.result
                  ? 'op50'
                  : ''
            "
            @click="index = idx.toString()"
          >
            <span :class="currentIndex === idx ? 'font-bold' : ''">
              <PluginName :name="tr.name" />
            </span>
            <Badge
              v-if="tr.result === data.transforms[idx - 1]?.result"
              class="bg-orange-400/10 text-orange-400"
              v-text="'no change'"
            />
            <Badge
              v-if="idx === 0"
              class="bg-light-blue-400/10 text-light-blue-400"
              v-text="'load'"
            />
            <Badge
              v-if="tr.order && tr.order !== 'normal'"
              class="bg-rose-400/10 text-rose-400"
              :title="tr.order.includes('-') ? `Using object hooks ${tr.order}` : tr.order"
              v-text="tr.order"
            />
            <span text-xs opacity-50 flex-auto text-right>{{ msToTime(tr.end - tr.start) }}</span>
          </button>
        </template>
      </Pane>
      <Pane min-size="5">
        <div h-full of-auto>
          <DiffEditor :from="from" :to="to" h-unset />
        </div>
      </Pane>
    </Splitpanes>
  </Container>
</template>
