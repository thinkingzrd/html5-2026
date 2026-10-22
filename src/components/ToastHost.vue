<script setup lang="ts">
import { useToast } from '@/composables/useToast'
import AppIcon from './AppIcon.vue'

const { items } = useToast()
</script>

<template>
  <div class="toast-host" role="status" aria-live="polite">
    <TransitionGroup name="toast">
      <div v-for="item in items" :key="item.id" class="toast" :class="item.tone">
        <AppIcon :name="item.tone === 'error' ? 'alert' : 'check'" class="toast-icon" />
        <span>{{ item.text }}</span>
      </div>
    </TransitionGroup>
  </div>
</template>

<style scoped>
.toast-host {
  position: fixed;
  left: 0;
  right: 0;
  /* 浮在底部导航之上 */
  bottom: calc(var(--tabbar-h) + var(--safe-bottom) + 14px);
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: var(--sp-2);
  pointer-events: none;
  z-index: 90;
  padding: 0 var(--sp-4);
}

.toast {
  display: flex;
  align-items: center;
  gap: 10px;
  max-width: 100%;
  padding: 7px 16px 7px 7px;
  border-radius: var(--r-full);
  font-size: var(--fs-sm);
  line-height: 1.45;
  color: var(--text-1);
  background: color-mix(in srgb, var(--surface-1) 88%, transparent);
  backdrop-filter: blur(16px) saturate(160%);
  -webkit-backdrop-filter: blur(16px) saturate(160%);
  border: 1px solid var(--border-strong);
  box-shadow: var(--shadow-lg);
}

/* 图标放进一枚语义色小圆片：成功绿、失败红，其余用品牌色 */
.toast-icon {
  flex: none;
  box-sizing: content-box;
  padding: 6px;
  border-radius: 50%;
  font-size: 0.875rem;
  color: var(--brand);
  background: var(--brand-soft);
}

.toast.success .toast-icon {
  color: var(--success);
  background: var(--success-soft);
}

.toast.error .toast-icon {
  color: var(--danger);
  background: var(--danger-soft);
}

.toast-enter-active,
.toast-leave-active {
  transition:
    opacity var(--dur) var(--ease),
    transform var(--dur) var(--ease);
}

.toast-enter-from,
.toast-leave-to {
  opacity: 0;
  transform: translateY(12px) scale(0.94);
}
</style>
