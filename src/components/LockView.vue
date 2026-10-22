<script setup lang="ts">
import { ref } from 'vue'
import AppIcon from './AppIcon.vue'
import { usePrivacyStore } from '@/stores/privacy'

const privacy = usePrivacyStore()
const pin = ref('')

async function submit(): Promise<void> {
  if (!pin.value) return
  const ok = await privacy.verify(pin.value)
  if (ok) pin.value = ''
  else pin.value = ''
}
</script>

<template>
  <div class="lock" role="dialog" aria-modal="true" aria-labelledby="lock-title">
    <div class="lock-card">
      <div class="lock-icon"><AppIcon name="lock" /></div>
      <h1 id="lock-title">AI 智囊团已锁定</h1>
      <template v-if="privacy.loadFailed">
        <p role="alert">{{ privacy.storageWarning }}</p>
        <p>请关闭可能占用数据库的其他页面，再重试读取。你的数据不会被清除。</p>
        <button class="unlock recovery" :disabled="privacy.loading" @click="privacy.init()">
          {{ privacy.loading ? '正在读取…' : '重试读取' }}
        </button>
      </template>
      <p v-else>请输入本机隐私 PIN 解锁</p>
      <form v-if="!privacy.loadFailed" class="form" @submit.prevent="submit">
        <input
          v-model="pin"
          class="pin"
          type="password"
          inputmode="numeric"
          pattern="[0-9]*"
          maxlength="8"
          autofocus
          autocomplete="off"
          placeholder="PIN"
          aria-label="隐私 PIN"
        />
        <button class="unlock" :disabled="privacy.busy || pin.length < 4">解锁</button>
      </form>
      <p v-if="privacy.errorText" class="error">{{ privacy.errorText }}</p>
      <p class="hint"><AppIcon name="shield" />PIN 仅用于本机解锁，忘记后无法找回。</p>
    </div>
  </div>
</template>

<style scoped>
.lock {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: var(--sp-5);
  background: var(--bg-ambient), var(--bg);
  text-align: center;
}
.lock-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: min(340px, 100%);
  padding: 30px 22px 22px;
  border-radius: var(--r-xl);
  border: 1px solid var(--border);
  background: var(--surface-1);
  box-shadow: var(--shadow-lg);
}
.lock-icon {
  width: 68px;
  height: 68px;
  display: grid;
  place-items: center;
  border-radius: 23px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 1.875rem;
  box-shadow:
    var(--shadow-brand),
    0 0 0 8px color-mix(in srgb, var(--brand) 10%, transparent);
}
h1 {
  margin: var(--sp-5) 0 0;
  font-size: var(--fs-lg);
  font-weight: 750;
}
p {
  margin: 8px 0 0;
  color: var(--text-2);
  font-size: var(--fs-sm);
  line-height: 1.6;
}
.form {
  width: 100%;
  display: flex;
  gap: var(--sp-2);
  margin-top: var(--sp-5);
}
.pin {
  min-width: 0;
  flex: 1;
  padding: 12px;
  border-radius: var(--r-md);
  border: 1px solid var(--border);
  background: var(--surface-2);
  font-size: var(--fs-md);
  letter-spacing: 0.3em;
  text-align: center;
  outline: none;
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}
.pin:focus {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}
.unlock {
  flex: none;
  padding: 0 20px;
  border-radius: var(--r-md);
  color: var(--brand-on);
  background: var(--grad-brand);
  font-weight: 700;
  box-shadow: var(--shadow-brand);
}
.unlock:disabled {
  opacity: 0.4;
  box-shadow: none;
}
.recovery {
  margin-top: 20px;
  padding: 12px 20px;
}
.error {
  color: var(--danger);
}
.hint {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin-top: var(--sp-5);
  font-size: var(--fs-xs);
  color: var(--text-3);
}
</style>
