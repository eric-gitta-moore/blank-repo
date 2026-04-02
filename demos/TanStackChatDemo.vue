<script setup lang="ts">
import { computed, nextTick, onMounted, ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

type Msg = { id: string; text: string; mine: boolean; ts: number }

const viewportRef = ref<HTMLElement | null>(null)
const messages = ref<Msg[]>(seed(80))

const idToIndex = computed(() => {
  const map = new Map<string, number>()
  messages.value.forEach((m, i) => map.set(m.id, i))
  return map
})

const virtualizer = useVirtualizer({
  count: computed(() => messages.value.length),
  getScrollElement: () => viewportRef.value,
  estimateSize: () => 72,
  overscan: 10,
  // 关键：动态高度测量后，如有尺寸变化自动调整，减少 prepend 抖动
  shouldAdjustScrollPositionOnItemSizeChange: true,
})

function prependHistory() {
  const firstBefore = messages.value[0]?.id
  const oldOffset = virtualizer.value.scrollOffset ?? 0
  const history = seed(30, Date.now() - 10_000_000)
  messages.value = [...history, ...messages.value]

  // 保持原视口锚点稳定：新数据渲染后按总高度差补偿
  nextTick(() => {
    const firstAfter = messages.value.findIndex((m) => m.id === firstBefore)
    if (firstAfter < 0) return
    const delta = history.length * 72 // 保守估计补偿
    virtualizer.value.scrollToOffset(oldOffset + delta)
  })
}

function appendIncoming() {
  messages.value.push({
    id: crypto.randomUUID(),
    text: `incoming ${Math.random().toString(36).slice(2)} `.repeat(1 + Math.floor(Math.random() * 4)),
    mine: Math.random() > 0.5,
    ts: Date.now(),
  })
}

function jumpToMessage(id: string) {
  const idx = idToIndex.value.get(id)
  if (idx == null) return
  virtualizer.value.scrollToIndex(idx, { align: 'center' })
}

onMounted(() => {
  // 初始定位到底部
  nextTick(() => {
    virtualizer.value.scrollToIndex(messages.value.length - 1, { align: 'end' })
  })
})

function seed(n: number, base = Date.now() - 1_000_000): Msg[] {
  return Array.from({ length: n }).map((_, i) => ({
    id: `${base}-${i}-${Math.random().toString(36).slice(2, 8)}`,
    text: `message #${i} ` + '内容 '.repeat(1 + Math.floor(Math.random() * 8)),
    mine: i % 3 === 0,
    ts: base + i * 12_000,
  }))
}
</script>

<template>
  <section class="chat-page">
    <header class="ops">
      <button @click="prependHistory">加载更早消息（prepend）</button>
      <button @click="appendIncoming">模拟新消息</button>
      <button @click="jumpToMessage(messages[Math.floor(messages.length / 2)]?.id)">跳到中间消息</button>
    </header>

    <div ref="viewportRef" class="viewport">
      <div :style="{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }">
        <div
          v-for="item in virtualizer.getVirtualItems()"
          :key="messages[item.index].id"
          :data-index="item.index"
          :ref="(el) => el && virtualizer.measureElement(el as Element)"
          class="row"
          :style="{
            position: 'absolute',
            top: 0,
            left: 0,
            width: '100%',
            transform: `translateY(${item.start}px)`,
          }"
        >
          <article class="bubble" :class="messages[item.index].mine ? 'mine' : 'other'">
            <p>{{ messages[item.index].text }}</p>
            <small>{{ new Date(messages[item.index].ts).toLocaleTimeString() }}</small>
          </article>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.chat-page { padding: 12px; }
.ops { display: flex; gap: 8px; margin-bottom: 8px; }
.viewport { height: 68vh; overflow: auto; border: 1px solid #ddd; border-radius: 8px; padding: 8px; }
.row { padding: 4px 0; }
.bubble { max-width: 70%; padding: 8px 10px; border-radius: 8px; background: #f4f4f5; }
.mine { margin-left: auto; background: #dbeafe; }
.other { margin-right: auto; }
</style>
