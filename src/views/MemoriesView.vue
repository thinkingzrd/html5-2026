<script setup lang="ts">
/**
 * 记忆可视化
 * ------------------------------------------------------------------
 * 抽象的"AI 记住了你"必须变成用户看得见、删得掉、关得上的卡片。
 */
import { ref } from 'vue'
import AppIcon from '@/components/AppIcon.vue'
import EmptyState from '@/components/EmptyState.vue'
import { useMemoryStore } from '@/stores/memory'
import { useToast } from '@/composables/useToast'

const memory = useMemoryStore()
const { show } = useToast()
const draft = ref('')
const topic = ref('偏好')
const adding = ref(false)

const topics = ['身份', '偏好', '目标', '经历', '其他']

/** 主题标签的点缀色，只影响展示；模型抽取出的其他主题统一用品牌色 */
const topicTones: Record<string, string> = {
  身份: 'var(--tone-1)',
  偏好: 'var(--tone-3)',
  目标: 'var(--tone-2)',
  经历: 'var(--tone-4)'
}

function toneOf(name: string): string {
  return topicTones[name] ?? 'var(--brand)'
}

async function addMemory(): Promise<void> {
  if (!draft.value.trim()) return
  adding.value = true
  try {
    await memory.add(draft.value, topic.value)
    draft.value = ''
    show('已记住', 'success')
  } finally {
    adding.value = false
  }
}

async function remove(id: string): Promise<void> {
  await memory.remove(id)
  show('已删除这条记忆', 'success')
}
</script>

<template>
  <div class="page scroll-y">
    <header class="head safe-x">
      <RouterLink to="/me" class="back" aria-label="返回我的">
        <AppIcon name="back" />
      </RouterLink>
      <div>
        <h1 class="title">记忆</h1>
        <p class="subtitle">AI 记住了什么，由你决定</p>
      </div>
    </header>

    <section class="privacy safe-x">
      <span class="privacy-icon" aria-hidden="true"><AppIcon name="shield" /></span>
      <div class="privacy-copy">
        <p class="privacy-title">启用长期记忆</p>
        <p class="privacy-desc">开启后，AI 会从对话中提取关于你的长期事实，只存本机。</p>
      </div>
      <button
        class="switch"
        :class="{ on: memory.enabled }"
        role="switch"
        :aria-checked="memory.enabled"
        @click="memory.setEnabled(!memory.enabled)"
      >
        <span />
      </button>
    </section>

    <section class="add-section safe-x">
      <h2 class="section-title">手动添加</h2>
      <div class="add-card">
        <textarea v-model="draft" class="input" rows="2" maxlength="120" placeholder="例如：我正在准备六级考试" />
        <div class="add-bottom">
          <div class="topic-list">
            <button
              v-for="item in topics"
              :key="item"
              class="topic"
              :class="{ selected: topic === item }"
              :style="{ '--tag': toneOf(item) }"
              @click="topic = item"
            >
              {{ item }}
            </button>
          </div>
          <button class="add-btn" :disabled="adding || !draft.trim()" aria-label="添加记忆" @click="addMemory">
            <AppIcon name="plus" />
          </button>
        </div>
      </div>
    </section>

    <section class="list-section safe-x">
      <div class="section-head">
        <h2 class="section-title">
          已记住 <span>{{ memory.memories.length }}</span>
        </h2>
        <span v-if="memory.extracting" class="extracting"><i aria-hidden="true" />正在整理…</span>
      </div>

      <EmptyState
        v-if="memory.ready && memory.memories.length === 0"
        icon="sparkle"
        title="还没有记忆"
        description="聊得越多，AI 越了解你。也可以先手动添加一条。"
      />

      <div v-else class="memory-list">
        <article
          v-for="item in memory.sortedMemories"
          :key="item.id"
          class="memory-card"
          :style="{ '--tag': toneOf(item.topic) }"
        >
          <div class="memory-top">
            <span class="memory-topic">{{ item.topic }}</span>
            <button class="delete-btn" aria-label="删除记忆" @click="remove(item.id)">
              <AppIcon name="trash" />
            </button>
          </div>
          <p class="memory-content">{{ item.content }}</p>
          <p class="memory-meta">
            重要度 {{ item.weight }} · {{ new Date(item.updatedAt).toLocaleDateString('zh-CN') }}
          </p>
        </article>
      </div>
    </section>

    <div class="tail" />
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
}

/* ---- 记忆开关 ---- */
.privacy {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  margin: 0 max(20px, var(--safe-right)) var(--sp-5) max(20px, var(--safe-left));
  padding: var(--sp-4);
  border-radius: var(--r-xl);
  border: 1px solid var(--brand-line);
  background:
    radial-gradient(120% 120% at 0% 0%, color-mix(in srgb, var(--brand) 20%, transparent) 0%, transparent 60%),
    var(--surface-1);
  box-shadow: var(--tech-shadow);
}
.privacy-icon {
  display: grid;
  place-items: center;
  flex: none;
  width: 42px;
  height: 42px;
  border-radius: 14px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 1.25rem;
  box-shadow: var(--shadow-brand);
}
.privacy-copy {
  flex: 1;
  min-width: 0;
}
.privacy-title {
  margin: 0;
  font-size: var(--fs-sm);
  font-weight: 700;
}
.privacy-desc {
  margin: 4px 0 0;
  font-size: var(--fs-xs);
  line-height: 1.6;
  color: var(--text-2);
}
.switch {
  flex: none;
  width: 50px;
  height: 30px;
  padding: 3px;
  border-radius: var(--r-full);
  background: var(--surface-3);
  border: 1px solid var(--border);
  transition: background var(--dur) var(--ease);
}
.switch span {
  display: block;
  width: 22px;
  height: 22px;
  border-radius: var(--r-full);
  background: var(--text-2);
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.25);
  transition:
    transform var(--dur) var(--ease-spring),
    background var(--dur) var(--ease);
}
.switch.on {
  background: var(--grad-brand);
  border-color: transparent;
}
.switch.on span {
  transform: translateX(20px);
  background: #fff;
}

.add-section,
.list-section {
  margin-bottom: var(--sp-5);
  padding-inline: max(20px, var(--safe-left)) max(20px, var(--safe-right));
}
.section-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0 0 var(--sp-3);
  font-size: var(--fs-md);
  font-weight: 700;
  color: var(--text-1);
}

/* ---- 手动添加 ---- */
.add-card {
  padding: var(--sp-3);
  border-radius: var(--r-lg);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
}
.input {
  width: 100%;
  padding: 11px 12px;
  border: 1px solid var(--border);
  border-radius: var(--r-sm);
  background: var(--surface-2);
  resize: none;
  outline: none;
  font-size: var(--fs-sm);
  line-height: 1.6;
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}
.input:focus {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}
.add-bottom {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  margin-top: 10px;
}
.topic-list {
  flex: 1;
  display: flex;
  gap: 6px;
  overflow-x: auto;
  scrollbar-width: none;
}
.topic-list::-webkit-scrollbar {
  display: none;
}
.topic {
  flex: none;
  padding: 5px 11px;
  border-radius: var(--r-full);
  font-size: 0.6875rem;
  font-weight: 600;
  color: var(--text-2);
  background: var(--surface-2);
  border: 1px solid transparent;
  transition:
    color var(--dur) var(--ease),
    background var(--dur) var(--ease);
}
.topic.selected {
  color: color-mix(in srgb, var(--tag) 70%, var(--text-1));
  background: color-mix(in srgb, var(--tag) 16%, transparent);
  border-color: color-mix(in srgb, var(--tag) 40%, transparent);
}
.add-btn {
  flex: none;
  width: 38px;
  height: 38px;
  display: grid;
  place-items: center;
  border-radius: 13px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 1.125rem;
  box-shadow: var(--shadow-brand);
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}
.add-btn:active {
  transform: scale(0.9);
}
.add-btn:disabled {
  color: var(--text-3);
  background: var(--surface-3);
  box-shadow: none;
}

/* ---- 已记住 ---- */
.section-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.section-head .section-title span {
  min-width: 24px;
  padding: 1px 8px;
  border-radius: var(--r-full);
  text-align: center;
  font-size: var(--fs-xs);
  color: var(--brand);
  background: var(--brand-soft);
}
.extracting {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-bottom: var(--sp-3);
  font-size: var(--fs-xs);
  color: var(--brand);
}
.extracting i {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--brand);
  animation: pulse 1.2s ease-in-out infinite;
}
@keyframes pulse {
  50% {
    opacity: 0.3;
    transform: scale(0.7);
  }
}
.memory-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.memory-card {
  position: relative;
  overflow: hidden;
  padding: 13px 14px 12px 18px;
  border-radius: var(--r-lg);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
}
.memory-card::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 4px;
  background: var(--tag);
  opacity: 0.85;
}
.memory-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.memory-topic {
  padding: 3px 9px;
  border-radius: var(--r-full);
  font-size: 0.6875rem;
  font-weight: 700;
  color: color-mix(in srgb, var(--tag) 70%, var(--text-1));
  background: color-mix(in srgb, var(--tag) 15%, transparent);
}
.delete-btn {
  display: grid;
  place-items: center;
  width: 32px;
  height: 32px;
  color: var(--text-3);
  font-size: 0.9375rem;
  border-radius: var(--r-full);
}
.delete-btn:active {
  color: var(--danger);
  background: var(--danger-soft);
}
.memory-content {
  margin: var(--sp-2) 0 0;
  font-size: var(--fs-sm);
  line-height: 1.7;
}
.memory-meta {
  margin: 8px 0 0;
  font-size: 0.6875rem;
  color: var(--text-3);
}
.tail {
  height: var(--sp-6);
}
</style>
