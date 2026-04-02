<script setup lang="ts">
import { computed, nextTick, ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

type Msg = { id: string; text: string; ts: number }

const RECENT_COUNT = 200
const viewportRef = ref<HTMLElement | null>(null)
const all = ref(seed(5000))

const history = computed(() => all.value.slice(0, Math.max(0, all.value.length - RECENT_COUNT)))
const recent = computed(() => all.value.slice(-RECENT_COUNT))

const historyVirtual = useVirtualizer({
  count: computed(() => history.value.length),
  getScrollElement: () => viewportRef.value,
  estimateSize: () => 56,
  overscan: 8,
})

function jumpToMessage(id: string) {
  const idx = all.value.findIndex((m) => m.id === id)
  if (idx < 0) return

  // 在 recent 区：直接 querySelector 到锚点
  if (idx >= all.value.length - RECENT_COUNT) {
    nextTick(() => {
      document.getElementById(`recent-${id}`)?.scrollIntoView({ block: 'center' })
    })
    return
  }

  // 在 history 虚拟区：scrollToIndex
  historyVirtual.value.scrollToIndex(idx, { align: 'center' })
}

function seed(n: number): Msg[] {
  const base = Date.now() - n * 10_000
  return Array.from({ length: n }, (_, i) => ({
    id: `m-${i}`,
    text: `#${i} ` + '历史消息 '.repeat(1 + (i % 6)),
    ts: base + i * 10_000,
  }))
}
</script>

<template>
  <section class="page">
    <header class="ops">
      <button @click="jumpToMessage('m-50')">跳到 m-50（历史区）</button>
      <button @click="jumpToMessage(`m-${all.length - 20}`)">跳到最近区消息</button>
    </header>

    <div ref="viewportRef" class="viewport">
      <!-- 历史区（虚拟） -->
      <div :style="{ height: `${historyVirtual.getTotalSize()}px`, position: 'relative' }">
        <div
          v-for="row in historyVirtual.getVirtualItems()"
          :key="history[row.index].id"
          class="item"
          :style="{
            position: 'absolute',
            top: 0,
            width: '100%',
            transform: `translateY(${row.start}px)`,
          }"
        >
          {{ history[row.index].text }}
        </div>
      </div>

      <!-- 最近区（真实 DOM） -->
      <div class="recent-divider">最近 {{ RECENT_COUNT }} 条（非虚拟）</div>
      <div v-for="m in recent" :id="`recent-${m.id}`" :key="m.id" class="item recent">
        {{ m.text }}
      </div>
    </div>
  </section>
</template>

<style scoped>
.page { padding: 12px; }
.ops { display: flex; gap: 8px; margin-bottom: 8px; }
.viewport { height: 72vh; overflow: auto; border: 1px solid #ddd; border-radius: 8px; }
.item { padding: 8px 10px; border-bottom: 1px solid #f0f0f0; }
.recent-divider { position: sticky; top: 0; background: #fff7ed; padding: 8px 10px; font-weight: 600; }
.recent { background: #fafafa; }
</style>
