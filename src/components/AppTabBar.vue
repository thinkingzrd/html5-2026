<script setup lang="ts">
/**
 * 底部导航
 * ------------------------------------------------------------------
 * 用 router-link 的 active class 做高亮；底部留出安全区内边距，
 * 避免被 iPhone 的手势条压住。
 */
import { useRoute } from 'vue-router'
import AppIcon from './AppIcon.vue'
import type { IconName } from './icons'

interface TabItem {
  to: string
  label: string
  icon: IconName
}

const tabs: TabItem[] = [
  { to: '/', label: '广场', icon: 'plaza' },
  { to: '/guess', label: '猜心', icon: 'sparkle' },
  { to: '/predict', label: '预测', icon: 'trend' },
  { to: '/library', label: '问知', icon: 'library' },
  { to: '/me', label: '我的', icon: 'me' }
]

const route = useRoute()
</script>

<template>
  <nav class="tabbar safe-x" aria-label="主导航">
    <RouterLink v-for="tab in tabs" :key="tab.to" :to="tab.to" class="tab" :class="{ active: route.path === tab.to }">
      <span class="tab-pill"><AppIcon :name="tab.icon" class="tab-icon" /></span>
      <span class="tab-label">{{ tab.label }}</span>
    </RouterLink>
  </nav>
</template>

<style scoped>
.tabbar {
  position: relative;
  z-index: 2;
  flex: none;
  display: flex;
  align-items: stretch;
  min-height: calc(var(--tabbar-h) + var(--safe-bottom));
  padding-bottom: var(--safe-bottom);
  padding-inline: max(6px, var(--safe-left)) max(6px, var(--safe-right));
  background: var(--glass-bg);
  -webkit-backdrop-filter: blur(18px) saturate(150%);
  backdrop-filter: blur(18px) saturate(150%);
  border-top: 1px solid var(--border);
  box-shadow: 0 -10px 30px rgba(0, 0, 0, 0.06);
}

.tab {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 3px;
  min-width: 0;
  color: var(--text-2);
  transition: color var(--dur) var(--ease);
}

/* 选中项的图标落在一枚胶囊里，比单纯变色更容易一眼认出当前页 */
.tab-pill {
  display: grid;
  place-items: center;
  width: 54px;
  height: 30px;
  border-radius: var(--r-full);
  transition:
    background var(--dur) var(--ease),
    transform var(--dur-fast) var(--ease);
}

.tab:active .tab-pill {
  transform: scale(0.9);
}

.tab-icon {
  font-size: 1.3125rem;
  transition: transform var(--dur-slow) var(--ease-spring);
}

.tab.active {
  color: var(--brand);
}

.tab.active .tab-pill {
  background: color-mix(in srgb, var(--brand) 17%, transparent);
}

.tab.active .tab-icon {
  transform: scale(1.08);
}

.tab-label {
  font-size: 0.6875rem;
  font-weight: 500;
  letter-spacing: 0.02em;
}

.tab.active .tab-label {
  font-weight: 700;
}
</style>
