<script setup lang="ts">
/**
 * 问理 · 多智能体辩论（P5）
 * ------------------------------------------------------------------
 * 选两位先贤、一个辩题，双方轮流流式发言、逐轮回应。你当裁判。
 */
import { ref } from 'vue'
import AppIcon from '@/components/AppIcon.vue'
import MarkdownView from '@/components/MarkdownView.vue'
import EmptyState from '@/components/EmptyState.vue'
import { SAGES } from '@/data/personas'
import { useDebateStore } from '@/stores/debate'

const debate = useDebateStore()
const draftTopic = ref('')

const sampleTopics = ['成大事者是否应当不拘小节', '人工智能能否真正理解人类情感', '读书贵精还是贵博']

function start(): void {
  debate.topic = draftTopic.value.trim()
  void debate.run()
}

function useSample(topic: string): void {
  draftTopic.value = topic
}
</script>

<template>
  <div class="page scroll-y">
    <header class="head safe-x">
      <RouterLink to="/" class="back" aria-label="返回广场"><AppIcon name="back" /></RouterLink>
      <div>
        <h1 class="title">问理</h1>
        <p class="subtitle">两位思想家同屏对辩，你来裁判</p>
      </div>
    </header>

    <section class="section safe-x">
      <h2 class="section-title">辩手</h2>
      <div class="speakers">
        <div class="speaker positive">
          <span class="side"><i aria-hidden="true" />正方</span>
          <select
            v-model="debate.aId"
            class="select"
            aria-label="正方辩手"
            :disabled="debate.running || debate.canResume"
          >
            <option v-for="sage in SAGES" :key="sage.id" :value="sage.id" :disabled="sage.id === debate.bId">
              {{ sage.avatar }} {{ sage.name }}
            </option>
          </select>
        </div>
        <span class="vs" aria-hidden="true">VS</span>
        <div class="speaker negative">
          <span class="side"><i aria-hidden="true" />反方</span>
          <select
            v-model="debate.bId"
            class="select"
            aria-label="反方辩手"
            :disabled="debate.running || debate.canResume"
          >
            <option v-for="sage in SAGES" :key="sage.id" :value="sage.id" :disabled="sage.id === debate.aId">
              {{ sage.avatar }} {{ sage.name }}
            </option>
          </select>
        </div>
      </div>
    </section>

    <section class="section safe-x">
      <h2 class="section-title">辩题</h2>
      <div class="topic-card">
        <textarea
          v-model="draftTopic"
          class="topic-input"
          rows="2"
          :disabled="debate.running || debate.canResume"
          maxlength="300"
          placeholder="输入一个辩题，例如：人工智能能否真正理解人类情感？"
        />
        <div class="samples">
          <button
            v-for="topic in sampleTopics"
            :key="topic"
            class="sample"
            :disabled="debate.running || debate.canResume"
            @click="useSample(topic)"
          >
            {{ topic }}
          </button>
        </div>
        <div class="rounds">
          <span>轮数</span>
          <div class="round-pills" role="group" aria-label="辩论轮数">
            <button
              v-for="n in [1, 2, 3]"
              :key="n"
              class="round"
              :class="{ on: debate.rounds === n }"
              :disabled="debate.running || debate.canResume"
              :aria-label="`${n} 轮`"
              :aria-pressed="debate.rounds === n"
              @click="debate.rounds = n"
            >
              {{ n }}
            </button>
          </div>
        </div>
      </div>
    </section>

    <section class="section safe-x">
      <button
        v-if="!debate.running && !debate.canResume"
        class="start-btn"
        :disabled="!draftTopic.trim() || debate.aId === debate.bId"
        @click="start"
      >
        <AppIcon name="swap" /> 开始辩论
      </button>
      <button v-if="debate.canResume" class="start-btn" @click="debate.run(true)">
        <AppIcon name="send" />继续当前轮
      </button>
      <button v-if="debate.canResume" class="restart-btn" @click="debate.reset()">重新设置辩论</button>
      <button v-if="debate.running" class="stop-btn" @click="debate.stop"><AppIcon name="stop" /> 停止</button>
      <p v-if="debate.errorText" class="error">{{ debate.errorText }}</p>
    </section>

    <section class="section safe-x">
      <h2 class="section-title">
        辩论现场 <small v-if="debate.turns.length">{{ debate.completedCount }} / {{ debate.rounds * 2 }} 次发言</small>
      </h2>
      <EmptyState
        v-if="!debate.turns.length && !debate.running"
        icon="chat"
        title="还没开始"
        description="选定辩手与辩题，点击「开始辩论」。"
      />
      <div v-else class="turns">
        <article
          v-for="turn in debate.turns"
          :key="turn.id"
          class="turn"
          :class="[turn.side, { speaking: debate.currentSpeakerId === turn.personaId && !turn.content.trim() }]"
        >
          <div class="turn-head">
            <span class="avatar">{{ turn.avatar }}</span>
            <div class="turn-meta">
              <strong>{{ turn.side === 'positive' ? '正方' : '反方' }} · {{ turn.personaName }}</strong>
              <small>第 {{ turn.round + 1 }} 轮 · {{ turn.role === 'opening' ? '开场' : '反驳' }}</small>
            </div>
            <span v-if="debate.currentSpeakerId === turn.personaId && debate.running" class="live">正在发言…</span>
          </div>
          <MarkdownView
            :content="turn.content || '…'"
            :streaming="debate.currentSpeakerId === turn.personaId && debate.running"
          />
        </article>
      </div>

      <div v-if="debate.finished" class="viewpoint-summary">
        <h3>双方观点回看</h3>
        <div v-for="side in ['positive', 'negative']" :key="side" :class="side">
          <strong>{{ side === 'positive' ? '正方 · 支持' : '反方 · 反对' }}</strong>
          <p>
            {{
              debate.turns
                .filter((turn) => turn.side === side)
                .at(-1)
                ?.content.slice(0, 180)
            }}{{ (debate.turns.filter((turn) => turn.side === side).at(-1)?.content.length ?? 0) > 180 ? '…' : '' }}
          </p>
        </div>
        <p class="verdict">以上摘自末轮发言，你觉得谁更有道理？</p>
      </div>
    </section>

    <div class="tail" />
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
  /* 正方用主题主色，反方用警示色，两种颜色贯穿选手卡、发言卡与回看 */
  --pro: var(--brand);
  --con: var(--warning);
}
.section {
  margin-bottom: var(--sp-5);
}
.section-title {
  display: flex;
  align-items: baseline;
  gap: 8px;
  margin: 0 0 var(--sp-3);
  font-size: var(--fs-md);
  font-weight: 700;
  color: var(--text-1);
}
.section-title small {
  color: var(--text-3);
  font-size: var(--fs-xs);
  font-weight: 500;
}

/* ---- 辩手：左右对垒 ---- */
.speakers {
  position: relative;
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
  gap: 12px;
}
.speaker {
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 0;
  padding: 12px;
  border-radius: var(--r-lg);
  border: 1px solid color-mix(in srgb, var(--side) 40%, var(--border));
  background:
    radial-gradient(120% 90% at 0% 0%, color-mix(in srgb, var(--side) 16%, transparent) 0%, transparent 65%),
    var(--surface-1);
  box-shadow: var(--card-shadow);
}
.speaker.positive {
  --side: var(--pro);
}
.speaker.negative {
  --side: var(--con);
}
.side {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: var(--fs-xs);
  font-weight: 700;
  color: var(--side);
}
.side i {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--side);
  box-shadow: 0 0 0 3px color-mix(in srgb, var(--side) 22%, transparent);
}
.select {
  width: 100%;
  min-width: 0;
  padding: 10px 30px 10px 10px;
  border: 1px solid var(--border);
  border-radius: var(--r-sm);
  background-color: var(--surface-2);
  color: var(--text-1);
  font-size: var(--fs-sm);
}
.select:disabled {
  opacity: 0.7;
}
/* VS 徽标压在两张卡中间 */
.vs {
  position: absolute;
  left: 50%;
  top: 50%;
  z-index: 1;
  display: grid;
  place-items: center;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  font-size: 0.75rem;
  font-weight: 900;
  font-style: italic;
  color: var(--brand-on);
  background: var(--grad-brand);
  border: 3px solid var(--bg);
  box-shadow: var(--shadow-brand);
}

/* ---- 辩题 ---- */
.topic-card {
  padding: 14px;
  border-radius: var(--r-lg);
  border: 1px solid var(--border);
  background: var(--surface-1);
  box-shadow: var(--card-shadow);
}
.topic-input {
  width: 100%;
  padding: 12px;
  border: 1px solid var(--border);
  border-radius: var(--r-md);
  background: var(--surface-2);
  color: var(--text-1);
  font-size: var(--fs-sm);
  line-height: 1.6;
  resize: vertical;
  outline: none;
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}
.topic-input:focus {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}
.samples {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 10px;
}
.sample {
  padding: 6px 11px;
  border-radius: var(--r-full);
  font-size: 0.6875rem;
  font-weight: 600;
  color: var(--brand);
  background: var(--brand-soft);
  border: 1px solid var(--brand-line);
}
.sample:disabled {
  opacity: 0.5;
}
.rounds {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 14px;
  padding-top: 12px;
  border-top: 1px solid var(--border);
  font-size: var(--fs-xs);
  font-weight: 600;
  color: var(--text-2);
}
.round-pills {
  display: flex;
  gap: 6px;
}
.round {
  width: 36px;
  height: 36px;
  border-radius: 12px;
  border: 1px solid var(--border);
  color: var(--text-2);
  background: var(--surface-2);
  font-weight: 700;
  transition:
    background var(--dur) var(--ease),
    color var(--dur) var(--ease);
}
.round.on {
  color: var(--brand-on);
  background: var(--grad-brand);
  border-color: transparent;
  box-shadow: var(--shadow-brand);
}
.round:disabled {
  opacity: 0.5;
}

/* ---- 操作按钮 ---- */
.start-btn,
.stop-btn {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: var(--sp-2);
  padding: 15px;
  border-radius: var(--r-md);
  font-size: var(--fs-md);
  font-weight: 700;
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}
.start-btn:active,
.stop-btn:active {
  transform: scale(0.98);
}
.start-btn {
  color: var(--brand-on);
  background: var(--grad-brand);
  box-shadow: var(--shadow-brand);
}
.start-btn:disabled {
  opacity: 0.45;
  box-shadow: none;
}
.stop-btn {
  color: var(--danger);
  background: var(--danger-soft);
  border: 1px solid var(--danger-line);
}
.restart-btn {
  display: block;
  margin: 12px auto 0;
  min-height: 44px;
  padding: 0 16px;
  border-radius: var(--r-full);
  color: var(--text-2);
  font-size: var(--fs-sm);
}
.error {
  margin: var(--sp-2) 0 0;
  padding: 10px 12px;
  border-radius: var(--r-sm);
  font-size: var(--fs-xs);
  line-height: 1.6;
  color: var(--danger);
  background: var(--danger-soft);
  border: 1px solid var(--danger-line);
}

/* ---- 辩论现场 ---- */
.turns {
  display: flex;
  flex-direction: column;
  gap: var(--sp-3);
}
.turn {
  position: relative;
  overflow: hidden;
  padding: 14px 14px 14px 16px;
  border-radius: var(--r-lg);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  animation: turn-in var(--dur-slow) var(--ease);
}
.turn::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 4px;
  background: var(--side);
}
.turn.positive {
  --side: var(--pro);
}
.turn.negative {
  --side: var(--con);
}
.turn.speaking {
  border-color: color-mix(in srgb, var(--side) 55%, var(--border));
  box-shadow: 0 0 0 3px color-mix(in srgb, var(--side) 16%, transparent);
}
@keyframes turn-in {
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: none;
  }
}
.turn-head {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}
.avatar {
  flex: none;
  width: 38px;
  height: 38px;
  display: grid;
  place-items: center;
  border-radius: 13px;
  background: color-mix(in srgb, var(--side) 16%, var(--surface-1));
  border: 1px solid color-mix(in srgb, var(--side) 30%, transparent);
  font-size: 1.2rem;
}
.turn-meta {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 1px;
}
.turn-meta strong {
  font-size: var(--fs-sm);
  font-weight: 700;
}
.turn-meta small {
  font-size: 0.6875rem;
  color: var(--text-3);
}
.live {
  flex: none;
  padding: 3px 9px;
  border-radius: var(--r-full);
  font-size: 0.6875rem;
  font-weight: 600;
  color: var(--side);
  background: color-mix(in srgb, var(--side) 14%, transparent);
  animation: pulse 1.2s ease-in-out infinite;
}
@keyframes pulse {
  50% {
    opacity: 0.4;
  }
}
.viewpoint-summary {
  margin-top: 20px;
  padding: 18px;
  border-radius: var(--r-xl);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--tech-shadow);
}
.viewpoint-summary h3 {
  margin: 0 0 12px;
  font-size: var(--fs-md);
  font-weight: 750;
}
.viewpoint-summary > div {
  padding: 12px 14px;
  margin: 10px 0;
  border-radius: var(--r-md);
  background: color-mix(in srgb, var(--side) 8%, var(--surface-2));
  border-left: 3px solid var(--side);
}
.viewpoint-summary .positive {
  --side: var(--pro);
}
.viewpoint-summary .negative {
  --side: var(--con);
}
.viewpoint-summary strong {
  color: var(--side);
  font-size: var(--fs-sm);
}
.viewpoint-summary > div p {
  margin: 6px 0 0;
  font-size: var(--fs-xs);
  line-height: 1.7;
  color: var(--text-2);
  overflow-wrap: anywhere;
}
.verdict {
  margin: 14px 0 0;
  padding: 12px;
  border-radius: var(--r-md);
  background: var(--brand-soft);
  border: 1px solid var(--brand-line);
  color: var(--brand);
  font-size: var(--fs-sm);
  font-weight: 600;
  text-align: center;
}
.tail {
  height: var(--sp-6);
}
</style>
