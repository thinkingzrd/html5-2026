<script setup lang="ts">
/**
 * 我的（设置）
 * ------------------------------------------------------------------
 * 这里最重要的功能是大模型接口配置 —— 没有它整个应用跑不起来。
 * 设计原则：
 *   1. 密钥只写进本机 IndexedDB，不随代码提交、不经过中转服务器；
 *   2. 输入即自动保存（防抖），不需要用户找"保存"按钮；
 *   3. 提供"测试连接"，把"配错了"这件事在演示前就暴露出来。
 */
import { computed, ref, watch } from 'vue'
import AppIcon from '@/components/AppIcon.vue'
import { PROVIDER_PRESETS } from '@/api/types'
import { testChatEndpoint, type TestResult } from '@/api/llm'
import { useTheme, VISUAL_THEMES } from '@/composables/useTheme'
import { useToast } from '@/composables/useToast'
import { usePrivacyStore } from '@/stores/privacy'
import { useSettingsStore } from '@/stores/settings'

const settings = useSettingsStore()
const privacy = usePrivacyStore()
const { visualStyle, setVisualStyle } = useTheme()
const { show } = useToast()

const showKey = ref(false)
const saved = ref(false)
const chatTest = ref<TestResult | null>(null)
const testing = ref<'' | 'chat'>('')
const pinDraft = ref('')
const pinBusy = ref(false)
const pinError = ref('')

/** 当前厂商的说明文字 */
const selectedPreset = computed(() => PROVIDER_PRESETS.find((p) => p.id === settings.api.providerId))

let persistTimer: number | undefined
let savedTimer: number | undefined

watch(
  () => settings.api,
  () => {
    if (persistTimer !== undefined) window.clearTimeout(persistTimer)
    persistTimer = window.setTimeout(async () => {
      try {
        await settings.persist()
        saved.value = true
        if (savedTimer !== undefined) window.clearTimeout(savedTimer)
        savedTimer = window.setTimeout(() => {
          saved.value = false
        }, 1600)
      } catch (error) {
        saved.value = false
        show(`保存失败：${(error as Error).message}`, 'error')
      }
    }, 500)
  },
  { deep: true }
)

watch(
  () => [settings.api.chat.baseUrl, settings.api.chat.model, settings.api.chat.apiKey],
  () => {
    chatTest.value = null
  }
)

function pickProvider(id: string): void {
  settings.applyPreset(id)
  chatTest.value = null
}

async function runChatTest(): Promise<void> {
  testing.value = 'chat'
  chatTest.value = null
  const endpoint = { ...settings.api.chat }
  try {
    const result = await testChatEndpoint(endpoint)
    if (
      Object.keys(endpoint).every(
        (key) => endpoint[key as keyof typeof endpoint] === settings.api.chat[key as keyof typeof endpoint]
      )
    )
      chatTest.value = result
  } finally {
    testing.value = ''
  }
}

async function savePin(): Promise<void> {
  if (!/^\d{4,8}$/.test(pinDraft.value)) {
    pinError.value = '请输入 4～8 位数字'
    return
  }
  pinBusy.value = true
  pinError.value = ''
  const ok = await privacy.setPin(pinDraft.value)
  pinBusy.value = false
  if (ok) {
    pinDraft.value = ''
    show('隐私锁已启用', 'success')
  } else {
    pinError.value = privacy.errorText ?? 'PIN 设置失败'
  }
}

async function disablePin(): Promise<void> {
  const value = pinDraft.value
  if (!value) {
    pinError.value = '请输入当前 PIN 后再关闭'
    return
  }
  pinBusy.value = true
  pinError.value = ''
  const ok = await privacy.disablePin(value)
  pinBusy.value = false
  if (ok) {
    pinDraft.value = ''
    show('隐私锁已关闭', 'success')
  } else {
    pinError.value = privacy.errorText ?? 'PIN 不正确'
  }
}
</script>

<template>
  <div class="page scroll-y settings-page">
    <header class="head safe-x">
      <div>
        <h1 class="title">我的</h1>
      </div>
      <Transition name="fade">
        <span v-if="saved" class="saved"><AppIcon name="check" />已保存</span>
      </Transition>
    </header>

    <!-- 大模型接口 -->
    <section class="section safe-x">
      <h2 class="section-title">
        <span class="title-icon"><AppIcon name="key" /></span>APIkey
        <span class="conn-state" :class="{ ready: settings.chatConfigured }">{{
          settings.chatConfigured ? '已配置' : '待配置'
        }}</span>
      </h2>

      <div class="chips">
        <button
          v-for="preset in PROVIDER_PRESETS"
          :key="preset.id"
          class="chip"
          :class="{ on: settings.api.providerId === preset.id }"
          @click="pickProvider(preset.id)"
        >
          {{ preset.name }}
        </button>
      </div>

      <label class="field">
        <span class="field-label">API Key</span>
        <span class="field-with-btn">
          <input
            v-model="settings.api.chat.apiKey"
            class="input"
            :type="showKey ? 'text' : 'password'"
            autocomplete="off"
            autocapitalize="off"
            spellcheck="false"
            placeholder="sk-..."
          />
          <button class="ghost-btn" @click="showKey = !showKey">
            {{ showKey ? '隐藏' : '显示' }}
          </button>
        </span>
      </label>

      <details class="advanced-config" :open="settings.api.providerId === 'custom'">
        <summary>
          <span class="summary-main">接口地址与模型</span>
          <span class="summary-state">{{ settings.api.chat.model || '待填写' }}</span>
          <AppIcon name="chevron" class="fold-chevron" />
        </summary>
        <p class="hint">常用服务商自动填写；自定义地址填到 /v1，不需要加 /chat/completions。</p>
        <label class="field">
          <span class="field-label">Base URL</span>
          <input
            v-model="settings.api.chat.baseUrl"
            class="input"
            type="url"
            inputmode="url"
            autocomplete="off"
            autocapitalize="off"
            spellcheck="false"
            placeholder="https://api.example.com/v1"
          />
        </label>

        <label class="field">
          <span class="field-label">模型名</span>
          <input
            v-model="settings.api.chat.model"
            class="input"
            autocomplete="off"
            autocapitalize="off"
            spellcheck="false"
            placeholder="deepseek-flash"
          />
        </label>
      </details>
      <a
        v-if="selectedPreset?.docsUrl"
        class="docs-link"
        :href="selectedPreset.docsUrl"
        target="_blank"
        rel="noopener noreferrer"
        >获取 {{ selectedPreset.name }} API Key <AppIcon name="arrow-up-right"
      /></a>
      <button class="primary-btn" :disabled="testing === 'chat' || !settings.chatConfigured" @click="runChatTest">
        <AppIcon name="bolt" />{{ testing === 'chat' ? '测试中…' : '测试连接' }}
      </button>

      <Transition name="fade">
        <p v-if="chatTest" class="result" :class="chatTest.ok ? 'ok' : 'bad'">
          <AppIcon :name="chatTest.ok ? 'check' : 'alert'" />
          {{ chatTest.message }}
        </p>
      </Transition>
    </section>

    <!-- 外观 -->
    <section class="section safe-x">
      <h2 class="section-title">
        <span class="title-icon"><AppIcon name="palette" /></span>外观
      </h2>
      <div class="style-choice" role="group" aria-label="界面风格">
        <button
          v-for="item in VISUAL_THEMES"
          :key="item.id"
          class="style-option"
          :class="{ selected: visualStyle === item.id }"
          :aria-pressed="visualStyle === item.id"
          @click="setVisualStyle(item.id)"
        >
          <!-- 迷你界面预览：主题底色上画一枚强调色图标和两条文字线 -->
          <span
            class="style-swatch"
            :style="{ '--sw-bg': item.background, '--sw-accent': item.accent }"
            aria-hidden="true"
            ><i /><i /><i
          /></span>
          <strong>{{ item.name }}</strong>
          <span v-if="visualStyle === item.id" class="style-check" aria-hidden="true"><AppIcon name="check" /></span>
        </button>
      </div>
    </section>

    <!-- 数据与记忆 -->
    <section class="section safe-x">
      <h2 class="section-title">
        <span class="title-icon"><AppIcon name="sparkle" /></span>我的管理
      </h2>
      <RouterLink to="/memories" class="row-btn">
        <span class="row-icon"><AppIcon name="history" /></span>
        <span class="row-body">
          <span class="row-title">长期记忆</span>
        </span>
        <span class="row-action">管理<AppIcon name="chevron-right" /></span>
      </RouterLink>
      <details class="privacy-box" :open="privacy.enabled">
        <summary>
          <span class="summary-main"><AppIcon name="lock" />高级设置</span>
          <span class="summary-state">{{ privacy.enabled ? '已启用' : '可选' }}</span>
          <AppIcon name="chevron" class="fold-chevron" />
        </summary>
        <div class="privacy-box-head">
          <div>
            <p class="row-title">隐私锁</p>
          </div>
          <span class="lock-status" :class="{ on: privacy.enabled }">{{ privacy.enabled ? '已启用' : '未启用' }}</span>
        </div>
        <input
          v-model="pinDraft"
          class="input pin-input"
          type="password"
          inputmode="numeric"
          maxlength="8"
          autocomplete="off"
          placeholder="4～8 位数字 PIN"
        />
        <div class="privacy-actions">
          <button class="primary-small" :disabled="pinBusy" @click="savePin">启用 / 修改</button>
          <button v-if="privacy.enabled" class="ghost-small" :disabled="pinBusy" @click="disablePin">关闭隐私锁</button>
        </div>
        <p v-if="pinError" class="pin-error">{{ pinError }}</p>
        <p v-if="privacy.storageWarning" class="pin-error">{{ privacy.storageWarning }}</p>
      </details>
    </section>

    <div class="tail" />
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
}

.head {
  align-items: center;
  justify-content: space-between;
}

.saved {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 10px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  font-weight: 600;
  color: var(--success);
  background: var(--success-soft);
}

/* 每组设置一张卡片 */
.section {
  margin: 0 20px 16px;
  padding: 18px;
  border: 1px solid var(--border);
  border-radius: var(--r-xl);
  background: var(--surface-1);
  box-shadow: var(--card-shadow);
}

.section-title {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 0 0 16px;
  font-size: 0.9375rem;
  font-weight: 700;
  color: var(--text-1);
}

.title-icon {
  display: grid;
  place-items: center;
  flex: none;
  width: 32px;
  height: 32px;
  border-radius: 11px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 1rem;
  box-shadow: var(--shadow-brand);
}

.conn-state {
  margin-left: auto;
  padding: 3px 9px;
  border-radius: var(--r-full);
  font-size: 0.6875rem;
  font-weight: 600;
  color: var(--warning);
  background: var(--warning-soft);
}

.conn-state.ready {
  color: var(--success);
  background: var(--success-soft);
}

.chips {
  display: flex;
  flex-wrap: wrap;
  gap: var(--sp-2);
  margin-bottom: var(--sp-4);
}

.chip {
  padding: 7px 13px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  color: var(--text-2);
  background: var(--surface-2);
  border: 1px solid var(--border);
  transition:
    color var(--dur) var(--ease),
    background var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}

.chip.on {
  color: var(--brand-on);
  font-weight: 700;
  background: var(--grad-brand);
  border-color: transparent;
  box-shadow: var(--shadow-brand);
}

.hint {
  margin: 0 0 var(--sp-3);
  font-size: var(--fs-xs);
  line-height: 1.65;
  color: var(--text-2);
}

.docs-link {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  margin: 0 0 var(--sp-3);
  color: var(--brand);
  font-size: var(--fs-xs);
  font-weight: 600;
  text-decoration: none;
}

.docs-link:focus-visible {
  outline: 2px solid var(--brand);
  outline-offset: 3px;
  border-radius: 3px;
}

.field {
  display: block;
  margin-bottom: var(--sp-3);
}

.field-label {
  display: block;
  margin-bottom: 6px;
  font-size: var(--fs-xs);
  font-weight: 600;
  color: var(--text-2);
}

.input {
  width: 100%;
  min-width: 0;
  padding: 11px 13px;
  border-radius: var(--r-sm);
  background: var(--surface-2);
  border: 1px solid var(--border);
  font-size: var(--fs-sm);
  outline: none;
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}

.input:focus {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}

.field-with-btn {
  display: flex;
  gap: var(--sp-2);
  align-items: center;
}

.ghost-btn {
  flex: none;
  padding: 10px 14px;
  border-radius: var(--r-sm);
  font-size: var(--fs-xs);
  font-weight: 600;
  color: var(--text-2);
  background: var(--surface-2);
  border: 1px solid var(--border);
}

.primary-btn {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 13px;
  border-radius: var(--r-md);
  font-size: var(--fs-sm);
  font-weight: 700;
  color: var(--brand-on);
  background: var(--grad-brand);
  box-shadow: var(--shadow-brand);
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}

.primary-btn:active {
  transform: scale(0.985);
}

.primary-btn:disabled {
  opacity: 0.5;
  box-shadow: none;
}

.result {
  display: flex;
  align-items: flex-start;
  gap: var(--sp-2);
  margin: var(--sp-3) 0 0;
  padding: 10px 12px;
  border-radius: var(--r-sm);
  font-size: var(--fs-xs);
  line-height: 1.6;
}

.result.ok {
  color: var(--success);
  background: var(--success-soft);
  border: 1px solid var(--success-line);
}

.result.bad {
  color: var(--danger);
  background: var(--danger-soft);
  border: 1px solid var(--danger-line);
}

/* 折叠项：接口地址与模型 / 高级设置 */
.advanced-config,
.privacy-box {
  border-radius: var(--r-md);
  background: var(--surface-2);
  border: 1px solid var(--border);
}

.advanced-config {
  margin: 0 0 14px;
  padding: 0 12px;
}

.advanced-config[open] {
  padding-bottom: 2px;
}

.advanced-config summary,
.privacy-box summary {
  display: flex;
  align-items: center;
  gap: 8px;
  min-height: 46px;
  list-style: none;
  font-size: var(--fs-xs);
  color: var(--text-1);
  cursor: pointer;
}

.advanced-config summary::-webkit-details-marker,
.privacy-box summary::-webkit-details-marker {
  display: none;
}

.summary-main {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-weight: 600;
}

.summary-main .icon {
  color: var(--brand);
}

.summary-state {
  margin-left: auto;
  max-width: 50%;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  color: var(--text-3);
  font-size: 0.6875rem;
}

.fold-chevron {
  flex: none;
  color: var(--text-3);
  transition: transform var(--dur) var(--ease);
}

details[open] > summary .fold-chevron {
  transform: rotate(180deg);
}

.advanced-config .input,
.privacy-box .input {
  background: var(--surface-1);
}

/* ---- 外观：迷你界面预览 ---- */
.style-choice {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 8px;
}

.style-option {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  min-width: 0;
  padding: 8px 8px 10px;
  border-radius: var(--r-md);
  background: var(--surface-2);
  border: 1px solid var(--border);
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease),
    transform var(--dur-fast) var(--ease);
}

.style-option:active {
  transform: scale(0.96);
}

.style-option.selected {
  border-color: var(--brand);
  box-shadow: 0 0 0 3px color-mix(in srgb, var(--brand) 20%, transparent);
}

.style-option strong {
  font-size: 0.75rem;
  font-weight: 600;
  white-space: nowrap;
}

.style-option.selected strong {
  color: var(--brand);
}

.style-swatch {
  position: relative;
  display: block;
  width: 100%;
  height: 44px;
  overflow: hidden;
  border-radius: 10px;
  background: var(--sw-bg);
  border: 1px solid color-mix(in srgb, var(--text-1) 12%, transparent);
}

.style-swatch i {
  position: absolute;
  border-radius: 3px;
  background: var(--sw-accent);
}

.style-swatch i:nth-child(1) {
  left: 8px;
  top: 8px;
  width: 16px;
  height: 16px;
  border-radius: 6px;
}

.style-swatch i:nth-child(2) {
  left: 30px;
  top: 10px;
  width: 42%;
  height: 4px;
  opacity: 0.6;
}

.style-swatch i:nth-child(3) {
  left: 8px;
  right: 8px;
  bottom: 7px;
  height: 9px;
  border-radius: 5px;
  opacity: 0.85;
}

.style-check {
  position: absolute;
  top: 4px;
  right: 4px;
  display: grid;
  place-items: center;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 0.6875rem;
}

/* ---- 我的管理 ---- */
.row-btn {
  width: 100%;
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  padding: 12px;
  border-radius: var(--r-md);
  background: var(--surface-2);
  border: 1px solid var(--border);
  color: var(--text-1);
  text-align: left;
  transition: background var(--dur-fast) var(--ease);
}

.row-btn:active {
  background: var(--surface-3);
}

.row-icon {
  display: grid;
  place-items: center;
  flex: none;
  width: 34px;
  height: 34px;
  border-radius: 11px;
  font-size: 1.0625rem;
  color: var(--brand);
  background: var(--brand-soft);
}

.row-body {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.row-title {
  margin: 0;
  font-size: var(--fs-sm);
  font-weight: 600;
}

.row-action {
  flex: none;
  display: inline-flex;
  align-items: center;
  gap: 2px;
  font-size: var(--fs-xs);
  color: var(--text-3);
}

.privacy-box {
  margin: 12px 0 0;
  padding: 0 12px;
}

.privacy-box[open] {
  padding-bottom: 12px;
}

.privacy-box[open] summary {
  margin-bottom: 4px;
}

.privacy-box-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-3);
  margin-bottom: var(--sp-3);
}

.lock-status {
  padding: 3px 9px;
  border-radius: var(--r-full);
  color: var(--text-3);
  background: var(--surface-1);
  font-size: 0.6875rem;
}

/* 只有真正启用时才显示为绿色。
   之前用 :has(.lock-status) 判断，而那个元素一直都在，
   结果「未启用」也是绿的 —— 状态色反而误导用户。 */
.lock-status.on {
  color: var(--success);
  background: var(--success-soft);
}

.pin-input {
  width: 100%;
}

.privacy-actions {
  display: flex;
  gap: var(--sp-2);
  margin-top: var(--sp-2);
}

.primary-small,
.ghost-small {
  flex: 1;
  padding: 10px 8px;
  border-radius: var(--r-sm);
  font-size: var(--fs-xs);
  font-weight: 600;
}

.primary-small {
  color: var(--brand-on);
  background: var(--grad-brand);
}

.ghost-small {
  color: var(--text-2);
  background: var(--surface-1);
  border: 1px solid var(--border);
}

.primary-small:disabled,
.ghost-small:disabled {
  opacity: 0.45;
}

.pin-error {
  margin: var(--sp-2) 0 0;
  font-size: var(--fs-xs);
  color: var(--danger);
}

.tail {
  height: var(--sp-6);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity var(--dur) var(--ease);
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 350px) {
  .section {
    margin-left: 14px;
    margin-right: 14px;
    padding: 14px;
  }
}
</style>
