<script setup lang="ts">
import { nextTick, onMounted, ref } from 'vue'
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller'

type Msg = { id: string; text: string; mine: boolean; ts: number }

const listRef = ref<InstanceType<typeof DynamicScroller> | null>(null)
const messages = ref<Msg[]>(build(100))

function prependHistory() {
  const anchorId = messages.value[0]?.id
  const history = build(25, Date.now() - 9_000_000)
  messages.value = [...history, ...messages.value]

  // 渲染后回到原锚点附近（简化版）
  nextTick(() => {
    const idx = messages.value.findIndex((m) => m.id === anchorId)
    if (idx >= 0) listRef.value?.scrollToItem?.(idx)
  })
}

function jumpToId(id: string) {
  const idx = messages.value.findIndex((m) => m.id === id)
  if (idx >= 0) listRef.value?.scrollToItem?.(idx)
}

function appendIncoming() {
  messages.value.push({
    id: crypto.randomUUID(),
    text: `new incoming `.repeat(2 + Math.floor(Math.random() * 5)),
    mine: Math.random() > 0.5,
    ts: Date.now(),
  })
}

onMounted(() => {
  nextTick(() => {
    listRef.value?.scrollToBottom?.()
  })
})

function build(n: number, base = Date.now() - 1_200_000): Msg[] {
  return Array.from({ length: n }, (_, i) => ({
    id: `${base}-${i}-${Math.random().toString(36).slice(2, 7)}`,
    text: `消息 ${i} ` + '文本 '.repeat(1 + Math.floor(Math.random() * 10)),
    mine: i % 2 === 0,
    ts: base + i * 18_000,
  }))
}
</script>

<template>
  <section class="page">
    <header class="ops">
      <button @click="prependHistory">加载历史</button>
      <button @click="appendIncoming">新增消息</button>
      <button @click="jumpToId(messages[Math.floor(messages.length / 3)]?.id)">跳转到 1/3 位置</button>
    </header>

    <DynamicScroller
      ref="listRef"
      class="scroller"
      :items="messages"
      key-field="id"
      :min-item-size="54"
      :buffer="400"
      v-slot="{ item, index, active }"
    >
      <DynamicScrollerItem
        :item="item"
        :active="active"
        :size-dependencies="[item.text]"
        :data-index="index"
      >
        <article class="bubble" :class="item.mine ? 'mine' : 'other'">
          <p>{{ item.text }}</p>
          <small>{{ new Date(item.ts).toLocaleString() }}</small>
        </article>
      </DynamicScrollerItem>
    </DynamicScroller>
  </section>
</template>

<style scoped>
.page { padding: 12px; }
.ops { display: flex; gap: 8px; margin-bottom: 10px; }
.scroller { height: 70vh; border: 1px solid #ddd; border-radius: 8px; padding: 8px; }
.bubble { max-width: 70%; margin: 6px 0; padding: 8px 10px; border-radius: 8px; background: #f4f4f5; }
.mine { margin-left: auto; background: #dcfce7; }
.other { margin-right: auto; }
</style>
