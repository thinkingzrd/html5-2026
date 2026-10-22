<script setup lang="ts">
import AppIcon from './AppIcon.vue'
import type { IconName } from './icons'

withDefaults(
  defineProps<{
    icon?: IconName
    title: string
    description?: string
  }>(),
  { icon: 'sparkle', description: '' }
)
</script>

<template>
  <div class="empty">
    <div class="empty-icon">
      <AppIcon :name="icon" />
    </div>
    <p class="empty-title">{{ title }}</p>
    <p v-if="description" class="empty-desc">{{ description }}</p>
    <slot />
  </div>
</template>

<style scoped>
.empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  padding: var(--sp-6) var(--sp-5);
  gap: var(--sp-2);
  border-radius: var(--r-xl);
  border: 1px dashed var(--border-strong);
  background: color-mix(in srgb, var(--surface-1) 55%, transparent);
}

/* 图标外再套一圈淡淡的光环，空状态不至于显得冷清 */
.empty-icon {
  width: 62px;
  height: 62px;
  border-radius: 21px;
  display: grid;
  place-items: center;
  font-size: 1.625rem;
  color: var(--brand);
  background: var(--brand-soft);
  border: 1px solid var(--brand-line);
  box-shadow: 0 0 0 7px color-mix(in srgb, var(--brand) 7%, transparent);
  margin-bottom: var(--sp-3);
  animation: pop var(--dur-slow) var(--ease-spring);
}

@keyframes pop {
  from {
    opacity: 0;
    transform: scale(0.8);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.empty-title {
  margin: 0;
  font-size: var(--fs-md);
  font-weight: 700;
}

.empty-desc {
  margin: 0;
  font-size: var(--fs-sm);
  color: var(--text-2);
  max-width: 30ch;
  line-height: 1.7;
  /* 优先在标点处折行，两行尽量等长，避免末行只剩一两个字 */
  text-wrap: balance;
  word-break: keep-all;
  overflow-wrap: anywhere;
}
</style>
