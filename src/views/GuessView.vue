<script setup lang="ts">
/**
 * 猜心 · 选做玩法（§2.2-D）
 * ------------------------------------------------------------------
 * 心里想好一个东西不说，App 每轮只问一个「是 / 否 / 不确定」的问题，
 * 直到猜出来。这一页只做展示与交互，规则与状态机分别在
 * prompt/guess.ts 与 stores/guess.ts 里，方便单独测试。
 *
 * 四个界面共用一套状态机：开局 → 对局 → 结算 → 历史回看。
 */
import { computed, onMounted, ref } from 'vue'
import AppIcon from '@/components/AppIcon.vue'
import EmptyState from '@/components/EmptyState.vue'
import { answerLabel, MAX_ROUNDS, type GuessAnswer } from '@/prompt/guess'
import { useGuessStore, type GuessRecord, type GuessTurn } from '@/stores/guess'
import { useSettingsStore } from '@/stores/settings'

const guess = useGuessStore()
const settings = useSettingsStore()

/** 展开的历史对局 id */
const expandedId = ref<string | null>(null)
/** 已问过的问题折叠列表 */
const showAsked = ref(false)

onMounted(() => {
  void guess.loadHistory().then(() => guess.restoreGame())
})

/** 只展示校验通过的问题，避免闪出 JSON 和随后被判重复的题。 */
const asking = computed(() => guess.phase === 'asking')
const waitingText = computed(() => {
  if (guess.requestStage === 'checking') return '正在核对候选与已有线索…'
  if (guess.requestStage === 'replanning') return '正在重新整理线索…'
  return '正在分析你的回答…'
})

/** 每轮都显示轮数与模型自报的把握 */
const confidencePercent = computed(() => Math.round(guess.confidence * 100))
const confidenceHint = computed(() => {
  if (!guess.guess) return ''
  return `模型当前猜的是「${guess.guess}」，把握约 ${confidencePercent.value}%`
})

/**
 * 候选答案的文案。
 * 两种来路语气不同：用户刚答了「是」，就说「那我猜是……」；
 * 模型自己报出高置信度，就说「我觉得答案就是……」。
 */
const candidateTitle = computed(() => {
  if (!guess.candidateAnswer) return ''
  return guess.phase === 'confirming'
    ? `那我猜是「${guess.candidateAnswer}」——对吗？`
    : `我觉得答案就是「${guess.candidateAnswer}」`
})

const candidateHint = computed(() =>
  guess.phase === 'confirming' ? '点「猜对了」结束这一局，点「不是」我接着问。' : '也可以先回答下面的问题。'
)

const resultTitle = computed(() => {
  if (guess.result === 'guessed') return '猜中了！'
  if (guess.result === 'gaveup') return '这局先到这里'
  return '这局还没有猜出来'
})

const resultDesc = computed(() => {
  if (guess.result === 'guessed') return `第 ${guess.turns.length} 轮猜中，用时 ${guess.elapsedSeconds} 秒。`
  if (guess.result === 'gaveup')
    return `你主动结束了这一局，共用 ${guess.turns.length} 轮、${guess.elapsedSeconds} 秒。`
  return `用时 ${guess.elapsedSeconds} 秒。可以再来一局。`
})

async function start(): Promise<void> {
  expandedId.value = null
  showAsked.value = false
  await guess.start()
}

function answer(value: GuessAnswer): void {
  // 请求进行中会忽略点击；这里守卫是为了防止连点产生第二轮提交
  if (guess.busy || guess.phase !== 'waiting') return
  void guess.submitAnswer(value)
}

function toggleAsked(): void {
  showAsked.value = !showAsked.value
}

function toggleRecord(record: GuessRecord): void {
  expandedId.value = expandedId.value === record.id ? null : record.id
}

/**
 * 历史记录里的轮次。
 * 走 store.replay 拿一份克隆，而不是直接渲染 record.turns ——
 * IndexedDB 读出来的对象虽然今天是普通对象，但一旦有人把历史改成响应式数组，
 * 回看时就会直接改到存档本身。
 */
function recordTurns(record: GuessRecord): GuessTurn[] {
  return guess.replay(record)
}

function formatDate(at: number): string {
  return new Date(at).toLocaleString('zh-CN', { month: 'numeric', day: 'numeric', hour: '2-digit', minute: '2-digit' })
}

function formatDuration(ms: number): string {
  const seconds = Math.max(0, Math.round(ms / 1000))
  if (seconds < 60) return `${seconds} 秒`
  return `${Math.floor(seconds / 60)} 分 ${seconds % 60} 秒`
}

function resultLabel(value: GuessRecord['result']): string {
  if (value === 'guessed') return '猜中'
  if (value === 'gaveup') return '放弃'
  return '未猜中'
}
</script>

<template>
  <div class="page scroll-y guess-page">
    <header class="head safe-x">
      <RouterLink to="/" class="back" aria-label="返回广场"><AppIcon name="back" /></RouterLink>
      <div>
        <h1 class="title">猜心空间</h1>
        <p class="subtitle">心中藏一个答案，让 AI 循着线索来猜</p>
      </div>
    </header>

    <!-- 未配置接口：没有 Key 一局都跑不起来 -->
    <section v-if="settings.ready && !settings.chatConfigured" class="section safe-x">
      <RouterLink to="/me" class="warn">
        <AppIcon name="key" class="warn-icon" />
        <span>尚未配置 API Key，点此前往设置</span>
        <AppIcon name="chevron-right" />
      </RouterLink>
    </section>

    <!-- 开局屏（FR-G1） -->
    <section v-if="guess.restoring" class="section safe-x restoring">正在恢复对局…</section>
    <template v-else-if="guess.phase === 'idle'">
      <section class="section safe-x">
        <div class="rules">
          <div class="mind-orb" aria-hidden="true">
            <span class="orb-ring" />
            <span class="orb-ring late" />
            <span class="orb-core"><AppIcon name="sparkle" /></span>
          </div>
          <p class="mind-label">大模型驱动 · 最多 {{ MAX_ROUNDS }} 轮</p>
          <p class="rules-lead">心里想好一个东西，先别告诉我。</p>
          <ul class="rules-list">
            <li><span class="rule-no">1</span>选一个人物、动物、物品或概念。</li>
            <li><span class="rule-no">2</span>根据心中的答案，回答「是」「否」或「不确定」。</li>
          </ul>
          <details class="game-details">
            <summary>了解猜心原理与进度保存 <AppIcon name="chevron" class="details-chevron" /></summary>
            <p class="rules-note">进度自动保存在本机，刷新后可以继续。不会写进长期记忆或对话列表。</p>
            <p class="rules-note">模型根据回答逐步缩小范围，并非读取想法；把握数值是模型自评，最终由你确认是否猜中。</p>
          </details>
        </div>
        <button class="start-btn" :disabled="!settings.chatConfigured" @click="start">
          <AppIcon name="sparkle" /> 开始猜心
        </button>
      </section>
    </template>

    <!-- 对局屏（FR-G2 / FR-G3）：请求中 / 等回答 / 等确认候选 / 请求失败待重试 -->
    <template v-else-if="guess.phase !== 'guessed' && guess.phase !== 'finished'">
      <section class="section safe-x">
        <div class="round-row">
          <span class="round"
            >第 <strong>{{ guess.round }}</strong> 轮</span
          >
          <span class="round-max">最多 {{ MAX_ROUNDS }} 轮</span>
        </div>
        <div
          class="round-progress"
          role="progressbar"
          aria-label="已使用轮数"
          :aria-valuenow="guess.round"
          :aria-valuemax="MAX_ROUNDS"
          aria-valuemin="0"
        >
          <span :style="{ width: `${Math.min(100, (guess.round / MAX_ROUNDS) * 100)}%` }" />
        </div>

        <!-- 请求失败：整局进度保留，只把这一轮标成失败 -->
        <div v-if="guess.phase === 'error'" class="failure">
          <span class="failure-icon" aria-hidden="true"><AppIcon name="alert" /></span>
          <p class="failure-title">线索已保留，稍后继续</p>
          <p class="failure-desc">{{ guess.failureText }}</p>
          <p class="failure-note">已经问过的 {{ guess.turns.length }} 轮都还在，点下面的按钮接着玩，不用从头开始。</p>
          <button class="start-btn" :disabled="guess.busy" @click="guess.retryRound()">
            <AppIcon name="sparkle" /> 重试这一轮
          </button>
        </div>

        <template v-else>
          <!-- :key 绑在轮数上：换一轮就重建卡片，入场动画才会重新播一次 -->
          <div :key="guess.round" class="question-card" :class="{ processing: asking }" aria-live="polite">
            <div class="mind-orb compact" aria-hidden="true">
              <span class="orb-ring" />
              <span class="orb-ring late" />
              <span class="orb-core"><AppIcon name="sparkle" /></span>
            </div>
            <p class="mind-label">
              {{ guess.candidateAnswer ? '等待你确认答案' : asking ? '正在整理你的线索' : '根据心中的答案选择' }}
            </p>
            <p v-if="asking" class="question thinking" role="status"><i class="caret" />{{ waitingText }}</p>
            <p v-else class="question">{{ guess.currentTurn?.question ?? '' }}</p>
          </div>

          <!-- 候选答案：只有用户点了「猜对了」才算猜中 -->
          <div v-if="guess.candidateAnswer" class="candidate">
            <p class="reveal-label"><AppIcon name="bolt" /> 我的猜测</p>
            <p class="candidate-name">{{ guess.candidateAnswer }}</p>
            <p class="candidate-text">{{ candidateTitle }}</p>
            <p class="candidate-hint">{{ candidateHint }}</p>
            <div class="candidate-actions">
              <button class="answer yes" @click="guess.confirmCandidate(true)"><AppIcon name="check" />猜对了</button>
              <button class="answer" @click="guess.confirmCandidate(false)"><AppIcon name="close" />不是</button>
            </div>
          </div>

          <p v-if="asking" class="status">这一轮最多等待 30 秒，你也可以暂停后重试。</p>
          <p v-else-if="confidenceHint && !guess.candidateAnswer" class="status">{{ confidenceHint }}</p>

          <div v-if="!guess.awaitingConfirm" class="answers">
            <button class="answer yes" :disabled="asking" @click="answer('yes')"><AppIcon name="check" />是的</button>
            <button class="answer" :disabled="asking" @click="answer('no')"><AppIcon name="close" />不是</button>
            <button class="answer" :disabled="asking" @click="answer('unsure')"><AppIcon name="help" />不确定</button>
          </div>
        </template>

        <div class="game-actions">
          <button v-if="asking" class="giveup" @click="guess.cancelRound()"><AppIcon name="stop" />暂停等待</button>
          <button v-if="guess.canUndo" class="giveup" @click="guess.undoAnswer()">
            <AppIcon name="history" />撤回上个回答
          </button>
          <button class="giveup danger" @click="guess.abandon()"><AppIcon name="close" />放弃这一局</button>
        </div>
      </section>

      <!-- 已问过的问题：默认折叠，需要回看时再展开 -->
      <section v-if="guess.turns.length" class="section safe-x">
        <button class="asked-toggle" @click="toggleAsked">
          <span
            >已问过的问题<em>{{ guess.turns.length }}</em></span
          >
          <AppIcon name="chevron" :class="['chevron', { open: showAsked }]" />
        </button>
        <ol v-if="showAsked" class="asked-list">
          <li v-for="(turn, index) in guess.turns" :key="index" class="asked-item">
            <span class="asked-no">{{ index + 1 }}</span>
            <span class="asked-q">{{ turn.question }}</span>
            <span class="asked-a" :class="[turn.answer || 'pending']">
              {{ turn.answer ? answerLabel(turn.answer) : '待回答' }}
            </span>
          </li>
        </ol>
      </section>
    </template>

    <!-- 结算屏（FR-G4） -->
    <template v-else>
      <section class="section safe-x">
        <div class="result" :class="guess.result === 'guessed' ? 'win' : 'lose'">
          <span class="result-icon" aria-hidden="true"
            ><AppIcon :name="guess.result === 'guessed' ? 'sparkle' : 'history'"
          /></span>
          <p class="result-title">{{ resultTitle }}</p>
          <p v-if="guess.result === 'guessed'" class="result-answer">{{ guess.guess }}</p>
          <p class="result-desc">{{ resultDesc }}</p>
          <div class="stats">
            <div>
              <strong>{{ guess.turns.length }}</strong
              ><small>轮数</small>
            </div>
            <div>
              <strong>{{ formatDuration(guess.durationMs) }}</strong
              ><small>用时</small>
            </div>
          </div>
        </div>

        <button class="start-btn" :disabled="!settings.chatConfigured" @click="start">
          <AppIcon name="sparkle" /> 再来一局
        </button>

        <button class="asked-toggle" @click="toggleAsked">
          <span
            >这一局的问答<em>{{ guess.turns.length }}</em></span
          >
          <AppIcon name="chevron" :class="['chevron', { open: showAsked }]" />
        </button>
        <ol v-if="showAsked" class="asked-list">
          <li v-for="(turn, index) in guess.turns" :key="index" class="asked-item">
            <span class="asked-no">{{ index + 1 }}</span>
            <span class="asked-q">{{ turn.question }}</span>
            <span class="asked-a" :class="[turn.answer || 'pending']">{{
              turn.answer ? answerLabel(turn.answer) : '未回答'
            }}</span>
          </li>
        </ol>
      </section>
    </template>

    <!-- 历史对局（FR-G5） -->
    <section class="section safe-x">
      <div class="section-head">
        <h2 class="section-title"><AppIcon name="history" /> 历史对局</h2>
        <button v-if="guess.hasHistory" class="clear" @click="guess.clearHistory()">清空</button>
      </div>

      <EmptyState v-if="!guess.hasHistory" icon="history" title="还没有对局记录" description="完成一局后可回看问答。" />

      <div v-else class="records">
        <article v-for="record in guess.history" :key="record.id" class="record">
          <button class="record-main" @click="toggleRecord(record)">
            <span class="record-result" :class="record.result">{{ resultLabel(record.result) }}</span>
            <span class="record-body">
              <span class="record-title">
                {{
                  record.result === 'guessed'
                    ? `猜中：${record.answer}`
                    : `${record.rounds} 轮 · ${formatDuration(record.durationMs)}`
                }}
              </span>
              <span class="record-meta">{{ formatDate(record.startedAt) }} · {{ record.rounds }} 轮</span>
            </span>
            <AppIcon name="chevron" :class="['chevron', { open: expandedId === record.id }]" />
          </button>

          <ol v-if="expandedId === record.id" class="asked-list">
            <li v-for="(turn, index) in recordTurns(record)" :key="index" class="asked-item">
              <span class="asked-no">{{ index + 1 }}</span>
              <span class="asked-q">{{ turn.question }}</span>
              <span class="asked-a" :class="[turn.answer || 'pending']">{{
                turn.answer ? answerLabel(turn.answer) : '未回答'
              }}</span>
            </li>
          </ol>
        </article>
      </div>
    </section>

    <p v-if="guess.storageWarning" class="warning safe-x">{{ guess.storageWarning }}</p>
    <p v-if="guess.errorText" class="game-hint safe-x">{{ guess.errorText }}</p>

    <div class="tail" />
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
}

.section {
  padding: 0 var(--sp-5);
  margin-bottom: var(--sp-5);
}

.section-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--sp-3);
}

.section-title {
  display: flex;
  align-items: center;
  gap: 7px;
  margin: 0;
  font-size: var(--fs-md);
  font-weight: 700;
  color: var(--text-1);
}

.section-title .icon {
  color: var(--brand);
}

.clear {
  padding: 5px 11px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  color: var(--text-2);
  background: var(--surface-1);
  border: 1px solid var(--border);
}

.restoring {
  color: var(--text-2);
  font-size: var(--fs-sm);
}

.warn {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 14px;
  border-radius: var(--r-md);
  font-size: var(--fs-sm);
  color: var(--text-1);
  background: var(--warning-soft);
  border: 1px solid var(--warning-line);
}

.warn span {
  flex: 1;
}

.warn-icon {
  color: var(--warning);
  font-size: 1.05rem;
}

/* ---- 心灵球：渐变核心 + 两圈向外扩散的光环 ---- */
.mind-orb {
  position: relative;
  display: grid;
  place-items: center;
  width: 124px;
  height: 124px;
  margin: 0 auto 14px;
}

.orb-core {
  position: relative;
  z-index: 1;
  display: grid;
  place-items: center;
  width: 76px;
  height: 76px;
  border-radius: 26px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 34px;
  box-shadow:
    var(--shadow-brand),
    inset 0 1px 0 rgba(255, 255, 255, 0.28);
}

.orb-ring {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  border: 1.5px solid color-mix(in srgb, var(--brand) 50%, transparent);
  animation: orb-pulse 2.8s var(--ease) infinite;
}

.orb-ring.late {
  animation-delay: 1.4s;
}

@keyframes orb-pulse {
  from {
    transform: scale(0.6);
    opacity: 0.9;
  }
  to {
    transform: scale(1.12);
    opacity: 0;
  }
}

.mind-orb.compact {
  width: 84px;
  height: 84px;
  margin-bottom: 12px;
}

.mind-orb.compact .orb-core {
  width: 52px;
  height: 52px;
  border-radius: 18px;
  font-size: 24px;
}

/* 对局中只有在模型思考时光环才扩散，等用户回答时保持安静 */
.question-card:not(.processing) .orb-ring {
  animation: none;
  opacity: 0;
}

.mind-label {
  margin: 0 0 12px;
  color: var(--brand);
  font-size: var(--fs-xs);
  font-weight: 600;
  letter-spacing: 0.04em;
  text-align: center;
}

/* ---- 开局屏 ---- */
.rules {
  position: relative;
  overflow: hidden;
  padding: 26px 20px 20px;
  border-radius: var(--r-xl);
  border: 1px solid var(--brand-line);
  background:
    radial-gradient(100% 70% at 50% 0%, color-mix(in srgb, var(--brand) 18%, transparent) 0%, transparent 70%),
    var(--surface-1);
  box-shadow: var(--tech-shadow);
}

.rules-lead {
  margin: 0;
  font-size: var(--fs-lg);
  font-weight: 750;
  text-align: center;
  letter-spacing: -0.01em;
}

.rules-list {
  display: grid;
  gap: 10px;
  margin: 18px 0 0;
  padding: 0;
  list-style: none;
  font-size: var(--fs-sm);
  line-height: 1.6;
  color: var(--text-2);
}

.rules-list li {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 10px 12px;
  border-radius: var(--r-sm);
  background: color-mix(in srgb, var(--surface-2) 70%, transparent);
}

.rule-no {
  display: grid;
  place-items: center;
  flex: none;
  width: 22px;
  height: 22px;
  margin-top: 1px;
  border-radius: 50%;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 0.75rem;
  font-weight: 700;
}

.game-details {
  margin-top: 14px;
  color: var(--text-3);
  font-size: var(--fs-xs);
}

.game-details summary {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 8px 0 2px;
  list-style: none;
  cursor: pointer;
}

.game-details summary::-webkit-details-marker {
  display: none;
}

.details-chevron {
  transition: transform var(--dur) var(--ease);
}

.game-details[open] .details-chevron {
  transform: rotate(180deg);
}

.rules-note {
  margin: var(--sp-2) 0 0;
  font-size: var(--fs-xs);
  line-height: 1.7;
  color: var(--text-3);
}

/* ---- 对局屏 ---- */
.round-row {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-bottom: var(--sp-2);
}

.round {
  font-size: var(--fs-md);
  font-weight: 650;
  color: var(--text-2);
}

.round strong {
  color: var(--text-1);
  font-size: 1.375rem;
  font-weight: 800;
  font-variant-numeric: tabular-nums;
}

.round-max {
  padding: 3px 9px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  color: var(--text-2);
  background: var(--surface-1);
  border: 1px solid var(--border);
}

.round-progress {
  height: 6px;
  margin: 10px 0 18px;
  overflow: hidden;
  border-radius: var(--r-full);
  background: var(--surface-2);
}

.round-progress span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: var(--grad-brand);
  box-shadow: 0 0 10px color-mix(in srgb, var(--brand) 50%, transparent);
  transition: width var(--dur-slow) var(--ease);
}

.question-card {
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: center;
  min-height: 216px;
  padding: 24px 20px 28px;
  text-align: center;
  border-radius: var(--r-xl);
  border: 1px solid var(--border);
  background:
    radial-gradient(90% 60% at 50% 0%, color-mix(in srgb, var(--brand) 14%, transparent) 0%, transparent 72%),
    var(--surface-1);
  box-shadow: var(--tech-shadow);
  overflow: hidden;
  animation: card-in var(--dur-slow) var(--ease);
}

/* 顶部居中一段渐变短条，避开大圆角，不会被裁成两头尖 */
.question-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 50%;
  width: 34%;
  height: 3px;
  border-radius: 0 0 3px 3px;
  transform: translateX(-50%);
  background: var(--grad-brand);
}

.question-card.processing {
  border-color: var(--brand-line);
}

@keyframes card-in {
  from {
    opacity: 0;
    transform: translateY(10px) scale(0.98);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

.question {
  margin: 0;
  min-height: 1.6em;
  font-size: clamp(21px, 5.6vw, 25px);
  font-weight: 700;
  line-height: 1.55;
  letter-spacing: -0.01em;
  overflow-wrap: anywhere;
  text-wrap: balance;
}

.question.thinking {
  font-size: var(--fs-md);
  font-weight: 500;
  color: var(--text-2);
}

.caret {
  display: inline-block;
  width: 2px;
  height: 1em;
  margin-right: 6px;
  vertical-align: -0.12em;
  background: var(--brand);
  animation: blink 1s steps(2, start) infinite;
}

@keyframes blink {
  to {
    opacity: 0;
  }
}

.status {
  margin: var(--sp-3) 0 0;
  font-size: var(--fs-xs);
  line-height: 1.6;
  color: var(--text-3);
  text-align: center;
}

/* ---- 候选答案确认 ---- */
.candidate {
  margin-top: var(--sp-3);
  padding: 22px 16px 16px;
  text-align: center;
  border-radius: var(--r-xl);
  border: 1px solid var(--brand-line);
  background:
    radial-gradient(100% 80% at 50% 0%, color-mix(in srgb, var(--brand) 22%, transparent) 0%, transparent 70%),
    var(--surface-1);
  box-shadow: var(--shadow-brand);
  animation: card-in var(--dur-slow) var(--ease-spring);
}

.reveal-label {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin: 0;
  padding: 4px 11px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  font-weight: 700;
  color: var(--brand);
  background: var(--brand-soft);
}

.candidate-name {
  margin: 12px 0 8px;
  font-size: clamp(26px, 8vw, 32px);
  font-weight: 800;
  letter-spacing: -0.02em;
  color: var(--text-1);
  overflow-wrap: anywhere;
}

.candidate-text {
  margin: 0;
  font-size: var(--fs-sm);
  line-height: 1.6;
  color: var(--text-2);
}

.candidate-hint {
  margin: 6px 0 0;
  font-size: var(--fs-xs);
  line-height: 1.6;
  color: var(--text-3);
}

.candidate-actions {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--sp-2);
  margin-top: var(--sp-4);
}

/* ---- 回答按钮 ---- */
.answers {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
  margin-top: var(--sp-4);
}

.answer {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  min-width: 0;
  padding: 15px 4px;
  border-radius: var(--r-md);
  font-size: var(--fs-md);
  font-weight: 650;
  color: var(--text-1);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  transition:
    transform var(--dur-fast) var(--ease),
    background var(--dur) var(--ease),
    opacity var(--dur) var(--ease);
}

.answer .icon {
  flex: none;
  font-size: 1.05rem;
  color: var(--text-3);
}

.answer:active {
  transform: scale(0.95);
  background: var(--surface-2);
}

.answer.yes {
  color: var(--brand-on);
  background: var(--grad-brand);
  border-color: transparent;
  box-shadow: var(--shadow-brand);
}

.answer.yes .icon {
  color: inherit;
}

.answer.yes:active {
  filter: brightness(0.95);
}

.answer:disabled {
  opacity: 0.4;
}

/* ---- 次要操作：暂停 / 撤回 / 放弃，并排成一行 ---- */
.game-actions {
  display: flex;
  flex-wrap: wrap;
  gap: var(--sp-2);
  margin-top: var(--sp-4);
}

.giveup {
  flex: 1 1 0;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  min-width: max-content;
  min-height: 40px;
  padding: 8px 12px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  color: var(--text-2);
  background: var(--surface-1);
  border: 1px solid var(--border);
  transition:
    color var(--dur) var(--ease),
    background var(--dur) var(--ease);
}

.giveup:active {
  background: var(--surface-2);
}

.giveup.danger:active {
  color: var(--danger);
  background: var(--danger-soft);
  border-color: var(--danger-line);
}

/* ---- 请求失败：进度保留，等用户重试 ---- */
.failure {
  padding: 22px 18px 18px;
  text-align: center;
  border-radius: var(--r-xl);
  background: var(--surface-1);
  border: 1px solid var(--warning-line);
  box-shadow: var(--card-shadow);
}

.failure-icon {
  display: inline-grid;
  place-items: center;
  width: 46px;
  height: 46px;
  border-radius: 15px;
  color: var(--warning);
  background: var(--warning-soft);
  font-size: 1.35rem;
}

.failure-title {
  margin: 12px 0 0;
  font-size: var(--fs-md);
  font-weight: 700;
}

.failure-desc {
  margin: var(--sp-2) 0 0;
  font-size: var(--fs-sm);
  line-height: 1.7;
  color: var(--text-2);
  word-break: break-word;
}

.failure-note {
  margin: var(--sp-2) 0 0;
  font-size: var(--fs-xs);
  line-height: 1.7;
  color: var(--text-3);
}

/* ---- 已问过的问题 ---- */
.asked-toggle {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  margin-top: var(--sp-3);
  padding: 13px var(--sp-4);
  border-radius: var(--r-md);
  font-size: var(--fs-sm);
  color: var(--text-1);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
}

.section > .asked-toggle:first-child {
  margin-top: 0;
}

.asked-toggle em {
  margin-left: 8px;
  padding: 1px 8px;
  border-radius: var(--r-full);
  font-style: normal;
  font-size: var(--fs-xs);
  font-weight: 700;
  color: var(--brand);
  background: var(--brand-soft);
}

.chevron {
  font-size: 1rem;
  color: var(--text-3);
  transition: transform var(--dur) var(--ease);
}

.chevron.open {
  transform: rotate(180deg);
}

.asked-list {
  margin: var(--sp-2) 0 0;
  padding: 4px var(--sp-3);
  list-style: none;
  border-radius: var(--r-md);
  background: var(--surface-1);
  border: 1px solid var(--border);
}

.asked-item {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  padding: 10px 0;
  border-bottom: 1px solid var(--border);
  font-size: var(--fs-xs);
  line-height: 1.6;
}

.asked-item:last-child {
  border-bottom: none;
}

.asked-no {
  display: grid;
  place-items: center;
  flex: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  font-size: 0.6875rem;
  font-weight: 700;
  color: var(--text-3);
  background: var(--surface-2);
}

.asked-q {
  flex: 1;
  min-width: 0;
  color: var(--text-2);
}

.asked-a {
  flex: none;
  padding: 2px 9px;
  border-radius: var(--r-full);
  font-weight: 600;
  color: var(--brand);
  background: var(--brand-soft);
}

.asked-a.yes {
  color: var(--success);
  background: var(--success-soft);
}

.asked-a.no {
  color: var(--danger);
  background: var(--danger-soft);
}

.asked-a.unsure {
  color: var(--warning);
  background: var(--warning-soft);
}

.asked-a.pending {
  color: var(--text-3);
  background: var(--surface-2);
}

/* ---- 结算屏 ---- */
.result {
  position: relative;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  padding: 30px 18px 24px;
  border-radius: var(--r-xl);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--tech-shadow);
  animation: card-in 300ms var(--ease);
}

.result.win {
  border-color: var(--brand-line);
  background:
    radial-gradient(110% 70% at 50% 0%, color-mix(in srgb, var(--brand) 24%, transparent) 0%, transparent 70%),
    var(--surface-1);
}

.result-icon {
  display: grid;
  place-items: center;
  width: 72px;
  height: 72px;
  border-radius: 24px;
  font-size: 2rem;
  line-height: 1;
  color: var(--brand);
  background: var(--brand-soft);
}

.result.win .result-icon {
  color: var(--brand-on);
  background: var(--grad-brand);
  box-shadow: var(--shadow-brand);
  animation: answer-in var(--dur-slow) var(--ease-spring);
}

.result-title {
  margin: var(--sp-4) 0 0;
  font-size: var(--fs-lg);
  font-weight: 750;
}

.result-answer {
  margin: var(--sp-2) 0 0;
  max-width: 100%;
  font-size: clamp(28px, 8vw, 36px);
  font-weight: 800;
  letter-spacing: -0.02em;
  background: var(--grad-brand);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  overflow-wrap: anywhere;
  animation: answer-in var(--dur-slow) var(--ease-spring);
}

@keyframes answer-in {
  from {
    opacity: 0;
    transform: scale(0.85);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

.result-desc {
  margin: var(--sp-2) 0 0;
  max-width: 30ch;
  font-size: var(--fs-sm);
  line-height: 1.7;
  color: var(--text-2);
}

.stats {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--sp-2);
  width: 100%;
  max-width: 300px;
  margin-top: var(--sp-4);
}

.stats div {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  padding: 10px 6px;
  border-radius: var(--r-sm);
  background: color-mix(in srgb, var(--surface-2) 80%, transparent);
}

.stats strong {
  font-size: var(--fs-lg);
  font-weight: 750;
  font-variant-numeric: tabular-nums;
}

.stats small {
  font-size: 0.6875rem;
  color: var(--text-3);
  letter-spacing: 0.06em;
}

.start-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: var(--sp-2);
  width: 100%;
  margin-top: var(--sp-4);
  padding: 15px;
  border-radius: var(--r-md);
  font-size: var(--fs-md);
  font-weight: 700;
  color: var(--brand-on);
  background: var(--grad-brand);
  box-shadow: var(--shadow-brand);
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}

.start-btn:active {
  transform: scale(0.98);
}

.start-btn:disabled {
  opacity: 0.4;
  box-shadow: none;
}

/* ---- 历史记录 ---- */
.records {
  display: flex;
  flex-direction: column;
  gap: var(--sp-2);
}

.record {
  border-radius: var(--r-md);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  overflow: hidden;
}

.record-main {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  width: 100%;
  padding: 13px var(--sp-4);
  text-align: left;
}

.record-result {
  flex: none;
  padding: 3px 9px;
  border-radius: var(--r-full);
  font-size: 0.6875rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: var(--text-3);
  background: var(--surface-2);
}

.record-result.guessed {
  color: var(--success);
  background: var(--success-soft);
}

.record-result.gaveup {
  color: var(--warning);
  background: var(--warning-soft);
}

.record-body {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.record-title {
  font-size: var(--fs-sm);
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.record-meta {
  font-size: 0.6875rem;
  color: var(--text-3);
}

.record .asked-list {
  margin: 0 var(--sp-3) var(--sp-3);
  background: var(--surface-2);
}

/* ---- 提示 ---- */
.warning,
.game-hint {
  padding-inline: max(20px, var(--safe-left)) max(20px, var(--safe-right));
  font-size: var(--fs-xs);
  line-height: 1.6;
}

.warning {
  margin: 0 0 var(--sp-2);
  color: var(--warning);
}

.game-hint {
  margin: 0;
  color: var(--text-3);
}

.tail {
  height: var(--sp-6);
}

@media (max-width: 350px) {
  .answers {
    gap: 7px;
  }
  .answer {
    gap: 4px;
    font-size: var(--fs-sm);
  }
  .answer .icon {
    font-size: 0.95rem;
  }
}

@media (prefers-reduced-motion: reduce) {
  .orb-ring,
  .result,
  .question-card,
  .candidate {
    animation: none;
  }
}
</style>
