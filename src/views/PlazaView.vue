<script setup lang="ts">
import { computed, ref } from 'vue'
import { useRouter } from 'vue-router'
import AppIcon from '@/components/AppIcon.vue'
import { DEFAULT_PERSONA_ID } from '@/data/personas'
import { useToast } from '@/composables/useToast'
import { useTheme, VISUAL_THEMES, type VisualStyle } from '@/composables/useTheme'
import { useChatStore } from '@/stores/chat'
import { usePersonasStore } from '@/stores/personas'
import { useSettingsStore } from '@/stores/settings'

const router = useRouter()
const chat = useChatStore()
const personas = usePersonasStore()
const settings = useSettingsStore()
const { show } = useToast()
const { theme, visualStyle, setVisualStyle } = useTheme()
const showThemes = ref(false)
const wish = ref('')
const entering = ref(false)
const recent = computed(() => chat.sessions.slice(0, 3))

/** 按时段打招呼：首页第一行字随一天的时间变化，打开时更有温度 */
const greeting = (() => {
  const hour = new Date().getHours()
  if (hour < 5) return '夜深了'
  if (hour < 11) return '早上好'
  if (hour < 13) return '中午好'
  if (hour < 18) return '下午好'
  return '晚上好'
})()

/** 今天的会话只显示时分，更早的显示月日 */
function formatRecent(at: number): string {
  const date = new Date(at)
  return date.toDateString() === new Date().toDateString()
    ? date.toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })
    : date.toLocaleDateString('zh-CN', { month: 'numeric', day: 'numeric' })
}

function pickTheme(style: VisualStyle): void {
  setVisualStyle(style)
  showThemes.value = false
}

async function quickChat(): Promise<void> {
  if (entering.value) return
  entering.value = true
  try {
    await chat.createSession(DEFAULT_PERSONA_ID)
    chat.selectedDocumentId = ''
    await router.push('/chat')
  } finally {
    entering.value = false
  }
}

async function startWish(): Promise<void> {
  const text = wish.value.trim()
  if (!text || entering.value) return
  entering.value = true
  try {
    const persona = personas.addTemp(text)
    await chat.createSession(persona.id)
    chat.selectedDocumentId = ''
    wish.value = ''
    show(`已以「${persona.name}」的身份开场`, 'success')
    await router.push('/chat')
  } finally {
    entering.value = false
  }
}

async function openSession(id: string): Promise<void> {
  await chat.selectSession(id)
  await router.push('/chat')
}
</script>

<template>
  <div class="page scroll-y">
    <div class="content">
      <header class="hero">
        <span class="brand-mark" aria-hidden="true"><AppIcon name="sparkle" /></span>
        <div class="hero-copy">
          <p class="eyebrow">{{ greeting }} · AI 智囊团</p>
          <h1>灵感广场</h1>
        </div>
        <button
          class="theme-toggle"
          aria-label="选择主题"
          :aria-expanded="showThemes"
          @click="showThemes = !showThemes"
        >
          <AppIcon :name="theme === 'dark' ? 'sun' : 'moon'" />
        </button>
      </header>

      <Transition name="drop">
        <div v-if="showThemes" class="theme-picker" role="group" aria-label="主题选择">
          <button
            v-for="item in VISUAL_THEMES"
            :key="item.id"
            :aria-pressed="visualStyle === item.id"
            :class="{ selected: visualStyle === item.id }"
            @click="pickTheme(item.id)"
          >
            <i :style="{ background: `linear-gradient(135deg, ${item.background} 50%, ${item.accent} 50%)` }" />{{
              item.name
            }}
          </button>
        </div>
      </Transition>

      <div v-if="settings.ready && !settings.chatConfigured" class="notice">
        <span class="notice-icon"><AppIcon name="key" /></span>
        <span class="notice-text">配置大模型接口后即可开始对话。</span>
        <RouterLink to="/me">去配置<AppIcon name="chevron-right" /></RouterLink>
      </div>

      <section class="ask-panel" aria-label="开始对话">
        <div class="hero-orbit" aria-hidden="true"><i /></div>

        <div class="ask-top">
          <h2>今天想聊些什么？</h2>
        </div>
        <button class="ask-button" :disabled="entering" @click="quickChat">
          <span class="ask-button-icon" aria-hidden="true"><AppIcon name="chat" /></span>
          <span class="ask-button-text">开启新的对话</span>
          <AppIcon name="arrow-up-right" class="ask-button-arrow" />
        </button>
        <div class="wish">
          <AppIcon name="sparkle" class="wish-icon" />
          <label class="sr-only" for="wish-input">描述想聊的角色</label>
          <input
            id="wish-input"
            v-model="wish"
            maxlength="40"
            placeholder="描述一个角色，如：数学老师"
            @keyup.enter="startWish"
          />
          <button :disabled="!wish.trim() || entering" aria-label="开始角色对话" @click="startWish">
            <AppIcon name="send" />
          </button>
        </div>
      </section>

      <section class="section" aria-labelledby="explore-title">
        <div class="section-heading">
          <h2 id="explore-title">探索功能</h2>
        </div>
        <div class="features">
          <RouterLink to="/guess" class="feature" style="--tile: var(--tone-1)">
            <span class="feature-icon"><AppIcon name="sparkle" /></span>
            <span class="feature-copy"><strong>猜心空间</strong><small>心中想一个事物，让 AI 来猜</small></span>
            <AppIcon name="arrow-up-right" class="arrow" />
          </RouterLink>
          <RouterLink to="/predict" class="feature" style="--tile: var(--tone-2)">
            <span class="feature-icon"><AppIcon name="trend" /></span>
            <span class="feature-copy"><strong>趋势预测</strong><small>让数据中的趋势清晰可见</small></span>
            <AppIcon name="arrow-up-right" class="arrow" />
          </RouterLink>
          <RouterLink to="/personas" class="feature" style="--tile: var(--tone-3)">
            <span class="feature-icon"><AppIcon name="chat" /></span>
            <span class="feature-copy"><strong>角色对话</strong><small>换个视角，展开新的交流</small></span>
            <AppIcon name="arrow-up-right" class="arrow" />
          </RouterLink>
          <RouterLink to="/library" class="feature" style="--tile: var(--tone-4)">
            <span class="feature-icon"><AppIcon name="library" /></span>
            <span class="feature-copy"><strong>问知知识库</strong><small>根据资料提问，回答附出处</small></span>
            <AppIcon name="arrow-up-right" class="arrow" />
          </RouterLink>
        </div>
      </section>

      <section v-if="recent.length" class="section" aria-labelledby="recent-title">
        <div class="section-heading">
          <div>
            <p class="eyebrow">接着上次的话题</p>
            <h2 id="recent-title">最近对话</h2>
          </div>
        </div>
        <div class="recent-list">
          <button v-for="session in recent" :key="session.id" class="recent-row" @click="openSession(session.id)">
            <span class="recent-initial" aria-hidden="true">{{ session.title.slice(0, 1) || '聊' }}</span>
            <span class="recent-body">
              <span class="recent-title">{{ session.title }}</span>
              <small class="recent-time">{{ formatRecent(session.updatedAt) }}</small>
            </span>
            <AppIcon name="chevron-right" class="recent-arrow" />
          </button>
        </div>
      </section>

      <section class="section extras" aria-labelledby="extras-title">
        <div class="section-heading">
          <div>
            <p class="eyebrow">换一种思路</p>
            <h2 id="extras-title">灵感与探索</h2>
          </div>
        </div>
        <div class="extra-grid">
          <RouterLink to="/debate" class="extra-card">
            <span class="extra-symbol" aria-hidden="true"><AppIcon name="swap" /></span>
            <strong>观点对辩</strong>
            <small>听见两种不同的声音</small>
          </RouterLink>
          <RouterLink to="/memories" class="extra-card">
            <span class="extra-symbol" aria-hidden="true"><AppIcon name="history" /></span>
            <strong>长期记忆</strong>
            <small>管理对话中记住的内容</small>
          </RouterLink>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
}
.content {
  max-width: 520px;
  margin: auto;
  padding: 20px max(20px, var(--safe-right)) 36px max(20px, var(--safe-left));
}

/* ---- 顶部：品牌标 + 问候 + 主题入口 ---- */
.hero {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 18px;
}
.brand-mark {
  display: grid;
  place-items: center;
  flex: none;
  width: 46px;
  height: 46px;
  border-radius: 15px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 1.375rem;
  box-shadow: var(--shadow-brand);
}
.hero-copy {
  flex: 1;
  min-width: 0;
}
.eyebrow {
  margin: 0 0 3px;
  color: var(--brand);
  font-size: 0.72rem;
  font-weight: 700;
  letter-spacing: 0.08em;
}
.hero h1 {
  margin: 0;
  color: var(--text-1);
  font-size: clamp(1.5rem, 6vw, 1.875rem);
  font-weight: 800;
  letter-spacing: -0.03em;
  line-height: 1.2;
}
.theme-toggle {
  display: grid;
  place-items: center;
  width: 44px;
  height: 44px;
  flex: none;
  border-radius: 14px;
  color: var(--brand);
  font-size: 1.125rem;
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  transition:
    transform var(--dur-fast) var(--ease),
    background var(--dur) var(--ease);
}
.theme-toggle:active {
  transform: scale(0.92);
}
.theme-toggle[aria-expanded='true'] {
  background: var(--brand-soft);
  border-color: var(--brand-line);
}

.theme-picker {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 8px;
  margin-bottom: 16px;
  padding: 10px;
  border: 1px solid var(--border);
  border-radius: var(--r-lg);
  background: var(--surface-1);
  box-shadow: var(--tech-shadow);
}
.theme-picker button {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  min-height: 44px;
  border: 1px solid var(--border);
  border-radius: var(--r-sm);
  font-size: 0.75rem;
  color: var(--text-2);
  background: var(--surface-2);
  transition:
    border-color var(--dur) var(--ease),
    background var(--dur) var(--ease);
}
.theme-picker button.selected {
  border-color: var(--brand);
  color: var(--brand);
  font-weight: 700;
  background: var(--brand-soft);
}
.theme-picker i {
  flex: none;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 1px solid color-mix(in srgb, var(--text-1) 16%, transparent);
}
.drop-enter-active,
.drop-leave-active {
  transition:
    opacity var(--dur) var(--ease),
    transform var(--dur) var(--ease);
}
.drop-enter-from,
.drop-leave-to {
  opacity: 0;
  transform: translateY(-6px) scale(0.98);
}

/* ---- 未配置接口的提示条 ---- */
.notice {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 16px;
  padding: 10px 10px 10px 12px;
  border: 1px solid var(--warning-line);
  border-radius: var(--r-md);
  background: var(--warning-soft);
  color: var(--text-1);
  font-size: var(--fs-xs);
}
.notice-icon {
  display: grid;
  place-items: center;
  flex: none;
  width: 30px;
  height: 30px;
  border-radius: 10px;
  color: var(--warning);
  background: color-mix(in srgb, var(--warning) 16%, transparent);
  font-size: 0.95rem;
}
.notice-text {
  flex: 1;
  min-width: 0;
  line-height: 1.5;
}
.notice a {
  flex: none;
  display: inline-flex;
  align-items: center;
  gap: 2px;
  padding: 6px 8px 6px 11px;
  border-radius: var(--r-full);
  color: var(--brand);
  background: var(--surface-1);
  font-weight: 700;
}

/* ---- 开始对话面板 ---- */
.ask-panel {
  position: relative;
  isolation: isolate;
  overflow: hidden;
  padding: 20px;
  border: 1px solid var(--brand-line);
  border-radius: var(--r-xl);
  background:
    radial-gradient(120% 90% at 100% 0%, color-mix(in srgb, var(--brand) 20%, transparent) 0%, transparent 58%),
    radial-gradient(90% 80% at 0% 110%, color-mix(in srgb, var(--brand-2) 15%, transparent) 0%, transparent 60%),
    var(--surface-1);
  box-shadow: var(--tech-shadow);
}
.ask-top h2 {
  margin: 0;
  font-size: 1.3rem;
  font-weight: 750;
  line-height: 1.3;
  letter-spacing: -0.02em;
}
.ask-button {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  min-height: 56px;
  margin-top: 16px;
  padding: 0 16px 0 9px;
  border-radius: 17px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 0.9375rem;
  font-weight: 700;
  box-shadow: var(--shadow-brand);
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}
.ask-button:active {
  transform: scale(0.98);
}
.ask-button:disabled {
  opacity: 0.6;
}
.ask-button-icon {
  display: grid;
  place-items: center;
  flex: none;
  width: 38px;
  height: 38px;
  border-radius: 12px;
  font-size: 1.125rem;
  background: color-mix(in srgb, var(--brand-on) 13%, transparent);
}
.ask-button-text {
  flex: 1;
  text-align: left;
}
.ask-button-arrow {
  font-size: 1.2rem;
}
.wish {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 12px;
  padding: 5px 5px 5px 13px;
  border: 1px solid var(--border);
  border-radius: 15px;
  background: color-mix(in srgb, var(--surface-2) 72%, transparent);
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}
.wish:focus-within {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}
.wish-icon {
  flex: none;
  color: var(--brand);
  font-size: 0.95rem;
}
.wish input {
  min-width: 0;
  flex: 1;
  padding: 8px 0;
  border: 0;
  outline: 0;
  background: transparent;
  font-size: var(--fs-sm);
}
.wish button {
  display: grid;
  place-items: center;
  flex: none;
  width: 36px;
  height: 36px;
  border-radius: 11px;
  color: var(--brand-on);
  background: var(--grad-brand);
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}
.wish button:active {
  transform: scale(0.9);
}
.wish button:disabled {
  color: var(--text-3);
  background: var(--surface-3);
}

/* 右上角的轨道装饰：外圈静止，内圈虚线缓慢旋转，中心一团品牌色光晕 */
.hero-orbit {
  position: absolute;
  z-index: -1;
  right: -48px;
  top: -52px;
  width: 196px;
  height: 196px;
  border-radius: 50%;
  border: 1px solid color-mix(in srgb, var(--brand) 26%, transparent);
}
.hero-orbit::before {
  content: '';
  position: absolute;
  inset: 30px;
  border-radius: 50%;
  border: 1px dashed color-mix(in srgb, var(--brand) 40%, transparent);
  animation: orbit-spin 36s linear infinite;
}
.hero-orbit::after {
  content: '';
  position: absolute;
  inset: 64px;
  border-radius: 50%;
  background: radial-gradient(circle, color-mix(in srgb, var(--brand) 42%, transparent), transparent 70%);
}
.hero-orbit i {
  position: absolute;
  inset: 0;
  animation: orbit-spin 14s linear infinite;
}
.hero-orbit i::before {
  content: '';
  position: absolute;
  left: 50%;
  bottom: 22px;
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--brand);
  box-shadow: 0 0 10px var(--brand);
}
@keyframes orbit-spin {
  to {
    transform: rotate(360deg);
  }
}

/* ---- 分区 ---- */
.section {
  margin-top: 26px;
}
.section-heading {
  display: flex;
  align-items: flex-end;
  justify-content: space-between;
  margin-bottom: 12px;
}
.section-heading .eyebrow {
  margin-bottom: 4px;
  font-size: 0.6875rem;
}
.section h2 {
  margin: 0;
  font-size: 1.0625rem;
  font-weight: 750;
  letter-spacing: -0.01em;
}

/* ---- 功能入口：每张卡一种点缀色 ---- */
.features {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}
.feature {
  position: relative;
  isolation: isolate;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 12px;
  min-height: 132px;
  padding: 15px 14px 14px;
  border: 1px solid var(--border);
  border-radius: var(--r-lg);
  background: var(--surface-1);
  color: var(--text-1);
  box-shadow: var(--card-shadow);
  transition:
    transform var(--dur-fast) var(--ease),
    border-color var(--dur) var(--ease);
}
.feature::before {
  content: '';
  position: absolute;
  z-index: -1;
  right: -38px;
  top: -38px;
  width: 120px;
  height: 120px;
  border-radius: 50%;
  background: radial-gradient(circle, color-mix(in srgb, var(--tile) 22%, transparent), transparent 70%);
  pointer-events: none;
}
.feature:active {
  transform: scale(0.97);
  border-color: color-mix(in srgb, var(--tile) 45%, var(--border));
}
.feature-icon {
  display: grid;
  place-items: center;
  flex: none;
  width: 40px;
  height: 40px;
  border-radius: 13px;
  color: var(--tile);
  background: color-mix(in srgb, var(--tile) 16%, transparent);
  border: 1px solid color-mix(in srgb, var(--tile) 22%, transparent);
  font-size: 1.2rem;
}
.feature-copy {
  display: grid;
  gap: 5px;
  min-width: 0;
}
.feature-copy strong {
  font-size: 0.9375rem;
  font-weight: 700;
}
.feature-copy small {
  color: var(--text-2);
  font-size: 0.75rem;
  line-height: 1.45;
  /* 只在标点处换行（如「…事物，/让 AI 来猜」），两行尽量等长；放不下时再任意断开兜底 */
  text-wrap: balance;
  word-break: keep-all;
  overflow-wrap: anywhere;
}
.feature .arrow {
  position: absolute;
  right: 13px;
  top: 15px;
  color: var(--text-3);
  font-size: 1.05rem;
}

/* ---- 最近对话 ---- */
.recent-list {
  overflow: hidden;
  border: 1px solid var(--border);
  border-radius: var(--r-lg);
  background: var(--surface-1);
  box-shadow: var(--card-shadow);
}
.recent-row {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  min-height: 62px;
  padding: 10px 14px;
  text-align: left;
  transition: background var(--dur-fast) var(--ease);
}
.recent-row + .recent-row {
  border-top: 1px solid var(--border);
}
.recent-row:active {
  background: var(--surface-2);
}
.recent-initial {
  display: grid;
  place-items: center;
  flex: none;
  width: 38px;
  height: 38px;
  border-radius: 12px;
  color: var(--brand);
  background: var(--brand-soft);
  font-weight: 700;
}
.recent-body {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}
.recent-title {
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  font-size: var(--fs-sm);
  font-weight: 600;
}
.recent-time {
  color: var(--text-3);
  font-size: 0.6875rem;
}
.recent-arrow {
  flex: none;
  color: var(--text-3);
  font-size: 1rem;
}

/* ---- 灵感与探索 ---- */
.extra-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}
.extra-card {
  display: flex;
  flex-direction: column;
  min-height: 122px;
  padding: 15px 14px;
  border: 1px solid var(--border);
  border-radius: var(--r-lg);
  background: var(--grad-card);
  color: var(--text-1);
  box-shadow: var(--card-shadow);
  transition: transform var(--dur-fast) var(--ease);
}
.extra-card:active {
  transform: scale(0.97);
}
.extra-symbol {
  display: grid;
  place-items: center;
  width: 36px;
  height: 36px;
  margin-bottom: auto;
  border-radius: 12px;
  color: var(--brand);
  background: var(--brand-soft);
  font-size: 1.05rem;
}
.extra-card strong {
  margin-top: 12px;
  font-size: var(--fs-sm);
  font-weight: 700;
}
.extra-card small {
  margin-top: 3px;
  color: var(--text-2);
  font-size: var(--fs-xs);
  line-height: 1.45;
  text-wrap: balance;
}

@media (max-width: 350px) {
  .content {
    padding-inline: 14px;
  }
  .hero {
    gap: 10px;
  }
  .brand-mark {
    width: 40px;
    height: 40px;
    border-radius: 13px;
  }
  .ask-panel {
    padding: 16px;
  }
  .features,
  .extra-grid {
    gap: 10px;
  }
  .feature {
    padding: 13px 12px 12px;
  }
}
</style>
