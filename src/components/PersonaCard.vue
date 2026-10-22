<script setup lang="ts">
/**
 * 先贤卡片
 * ------------------------------------------------------------------
 * 主色通过 CSS 自定义属性 --accent 注入，卡片内部所有强调色都读它。
 * 这样新增一位先贤只要在数据里给个颜色，不用再写一份样式。
 */
import type { Persona } from '@/data/personas'
import AppIcon from './AppIcon.vue'

defineProps<{
  persona: Persona
  /** 是否是当前对话正在使用的角色 */
  active?: boolean
  /**
   * 细分身份标签（如「各科老师」「律师」）。
   * 角色广场把若干细分分类并成了展示分组，靠这个标签把合并掉的信息补回来，
   * 否则「学习与学术」一组里分不清谁是任课老师、谁是论文导师。
   */
  subLabel?: string
}>()
</script>

<template>
  <button
    class="card"
    :class="{ active }"
    :style="{ '--accent': persona.accent }"
    :aria-pressed="active ? 'true' : 'false'"
  >
    <span class="avatar" aria-hidden="true">{{ persona.name.slice(0, 1) }}</span>
    <span class="body">
      <span class="name-row">
        <span class="name">{{ persona.name }}</span>
        <span v-if="active" class="badge">当前</span>
      </span>
      <span class="meta">
        <span class="era">{{ persona.era }}</span>
        <span v-if="subLabel" class="sub">{{ subLabel }}</span>
      </span>
      <span class="tagline">{{ persona.tagline }}</span>
    </span>
    <AppIcon name="chevron-right" class="go" />
  </button>
</template>

<style scoped>
.card {
  position: relative;
  display: flex;
  align-items: center;
  gap: 13px;
  padding: 12px 12px 12px 12px;
  border-radius: var(--r-md);
  text-align: left;
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  overflow: hidden;
  /* 角色主色混入主题文字色：既保留各人的辨识色，又在深浅主题下都看得清 */
  --accent-ink: color-mix(in srgb, var(--accent) 62%, var(--text-1));
  transition:
    transform var(--dur-fast) var(--ease),
    border-color var(--dur) var(--ease),
    background var(--dur) var(--ease);
}

/* 左缘一道角色色细条，列表里一眼就能分出不同角色 */
.card::before {
  content: '';
  position: absolute;
  left: 0;
  top: 14px;
  bottom: 14px;
  width: 3px;
  border-radius: 0 3px 3px 0;
  background: var(--accent);
  opacity: 0.75;
}

.card:active {
  transform: scale(0.975);
  background: var(--surface-2);
}

.card.active {
  border-color: var(--brand);
  box-shadow:
    var(--card-shadow),
    0 0 0 3px color-mix(in srgb, var(--brand) 16%, transparent);
}

.avatar {
  display: grid;
  place-items: center;
  flex: none;
  width: 46px;
  height: 46px;
  border-radius: 15px;
  color: var(--accent-ink);
  background: color-mix(in srgb, var(--accent) 20%, var(--surface-1));
  border: 1px solid color-mix(in srgb, var(--accent) 34%, transparent);
  font-size: 1.25rem;
  font-weight: 700;
  font-family: Georgia, 'Songti SC', serif;
}

.body {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.name-row {
  display: flex;
  align-items: center;
  gap: 6px;
  min-width: 0;
}

.name {
  font-size: var(--fs-sm);
  font-weight: 700;
}

/* 细分身份标签：和 era 同一行，太长时换行而不是把卡片撑破 */
.meta {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.era {
  font-size: 0.6875rem;
  font-weight: 600;
  letter-spacing: 0.04em;
  color: var(--accent-ink);
}

.sub {
  font-size: 0.625rem;
  padding: 1px 7px;
  border-radius: var(--r-full);
  color: var(--text-2);
  background: var(--surface-2);
  border: 1px solid var(--border);
}

.tagline {
  margin-top: 1px;
  font-size: var(--fs-xs);
  line-height: 1.55;
  color: var(--text-2);
}

.badge {
  flex: none;
  font-size: 0.625rem;
  font-weight: 700;
  padding: 1px 7px;
  border-radius: var(--r-full);
  color: var(--brand-on);
  background: var(--grad-brand);
}

.go {
  flex: none;
  font-size: 1rem;
  color: var(--text-3);
}
</style>
