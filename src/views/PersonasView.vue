<script setup lang="ts">
/**
 * 角色广场
 * ------------------------------------------------------------------
 * P1 时这里叫「先贤堂」，只列 8 位先贤；现在是一个覆盖多行业、80 多位角色的广场。
 *
 * 三个设计决定值得说明：
 *   1. **分类合并发生在展示层**。细分分类（老师 / 律师 / 医生…）保留在数据里，
 *      这里只把它们归成 7 个展示分组，chip 从 14 个降到 7 个。
 *      合并掉的信息由卡片上的「细分身份标签」补回来。
 *   2. **折叠展开**。组变少了、每组变大了，一次全铺开要滚很久，所以默认只展开第一组。
 *      一旦搜索或选中某个分类，就把命中的组全部展开 —— 折叠着等于把结果藏起来。
 *   3. **搜索**跨全部组匹配角色名 / 一句话介绍 / 时代。
 *
 * 两种进入方式，用查询参数区分：
 *   /personas              从首页进来 → 选一位，开一段新对话
 *   /personas?mode=switch  从对话页进来 → 给当前会话换一位
 *
 * 「换人不清空历史」是刻意的：换角色只替换 system prompt，对话记录照旧带进上下文。
 *
 * 「我的角色」单独用列表排版而不是复用 PersonaCard：
 * 自建角色卡上要挂「编辑 / 保存」按钮，而 PersonaCard 本身是个 button，
 * 按钮不能嵌套按钮（HTML 不允许，点上去行为也不确定）。
 */
import { computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import AppIcon from '@/components/AppIcon.vue'
import PersonaCard from '@/components/PersonaCard.vue'
import { CATEGORY_GROUPS, categoryMeta, groupOf, groupPersonas } from '@/data/personas'
import { useToast } from '@/composables/useToast'
import { useChatStore } from '@/stores/chat'
import { usePersonasStore } from '@/stores/personas'

const route = useRoute()
const router = useRouter()
const chat = useChatStore()
const personas = usePersonasStore()
const { show } = useToast()

const switchMode = computed(() => route.query.mode === 'switch')
const busy = ref(false)

/** 'all' 表示不筛分组，否则是 CATEGORY_GROUPS 的 id */
const activeCat = ref<string>('all')
const query = ref('')

const searching = computed(() => query.value.trim().length > 0)

/**
 * 默认展开前两组（万能对话 + 先贤智者）。
 * 只展开第一组的话，首屏只有一张卡，最有内容的先贤反而被折叠着。
 */
const DEFAULT_OPEN = CATEGORY_GROUPS.slice(0, 2).map((group) => group.id)

/** 展开的分组 id */
const expanded = ref<string[]>([...DEFAULT_OPEN])

// 搜索或切换分类时重置展开状态：这时把命中的组全部打开，
// 否则用户搜到了东西却看不到，还得再点一次展开
watch([query, activeCat], () => {
  expanded.value =
    searching.value || activeCat.value !== 'all' ? CATEGORY_GROUPS.map((group) => group.id) : [...DEFAULT_OPEN]
})

function toggleGroup(id: string): void {
  expanded.value = expanded.value.includes(id) ? expanded.value.filter((item) => item !== id) : [...expanded.value, id]
}

function isOpen(id: string): boolean {
  return expanded.value.includes(id)
}

const allOpen = computed(() => CATEGORY_GROUPS.every((group) => expanded.value.includes(group.id)))

function toggleAll(): void {
  expanded.value = allOpen.value ? [] : CATEGORY_GROUPS.map((group) => group.id)
}

const groupCounts = computed(() => {
  const map = new Map<string, number>()
  for (const item of personas.all) {
    const id = groupOf(item.category).id
    map.set(id, (map.get(id) ?? 0) + 1)
  }
  return map
})

const chips = computed(() => [
  { id: 'all', label: '全部', emoji: '🧭', count: personas.all.length },
  ...CATEGORY_GROUPS.map((group) => ({
    id: group.id,
    label: group.label,
    emoji: group.emoji,
    count: groupCounts.value.get(group.id) ?? 0
  }))
])

/** 搜索只匹配展示用字段，不搜提示词正文（identity/taboos 太长，命中会很意外） */
const matched = computed(() => {
  const keyword = query.value.trim().toLowerCase()
  if (!keyword) return personas.all
  return personas.all.filter((item) =>
    [item.name, item.tagline, item.era].some((field) => field.toLowerCase().includes(keyword))
  )
})

const groups = computed(() => {
  // 「我的角色」为空时也要留着「新建角色」入口；但搜索时不留 ——
  // 搜不到东西还挂着它，会让人误以为它在结果里
  const entries = groupPersonas(matched.value, searching.value ? [] : ['custom'])
  if (activeCat.value === 'all') return entries
  return entries.filter((entry) => entry.group.id === activeCat.value)
})

const totalMatched = computed(() => groups.value.reduce((sum, entry) => sum + entry.items.length, 0))

async function pick(personaId: string): Promise<void> {
  if (busy.value) return
  busy.value = true
  try {
    if (switchMode.value) {
      await chat.switchPersona(personaId)
      show('已换人，接着聊', 'success')
      router.back()
    } else {
      await chat.createSession(personaId)
      await router.push('/chat')
    }
  } finally {
    busy.value = false
  }
}

function createNew(): void {
  void router.push('/personas/edit')
}

function edit(id: string): void {
  void router.push({ path: '/personas/edit', query: { id } })
}

function goBack(): void {
  if (switchMode.value) router.back()
  else void router.push('/chat')
}
</script>

<template>
  <div class="page scroll-y">
    <header class="head safe-x">
      <button v-if="switchMode" class="back" aria-label="返回" @click="goBack">
        <AppIcon name="back" />
      </button>
      <div class="head-text">
        <h1 class="title">{{ switchMode ? '换一位角色' : '找人聊' }}</h1>
        <p class="subtitle">
          {{ switchMode ? '对话记录会保留，换个口吻接着聊' : '选一个视角，开始一场好对话' }}
        </p>
      </div>
    </header>

    <div class="search-wrap safe-x">
      <div class="search">
        <AppIcon name="search" class="search-icon" />
        <input v-model="query" class="search-input" type="search" placeholder="搜角色或用途，例如：健身、合同、面试" />
        <button v-if="searching" class="search-clear" aria-label="清空搜索" @click="query = ''">
          <AppIcon name="close" />
        </button>
      </div>
    </div>

    <div class="chips safe-x">
      <button
        v-for="chip in chips"
        :key="chip.id"
        class="chip"
        :class="{ on: activeCat === chip.id }"
        @click="activeCat = chip.id"
      >
        <span class="chip-emoji" aria-hidden="true">{{ chip.emoji }}</span>
        <span>{{ chip.label }}</span>
        <span class="chip-count">{{ chip.count }}</span>
      </button>
    </div>

    <p v-if="personas.storageWarning" class="warn safe-x">{{ personas.storageWarning }}</p>

    <div v-if="activeCat === 'all' && !searching" class="tools safe-x">
      <span class="tools-total"
        >共 <strong>{{ personas.all.length }}</strong> 位</span
      >
      <button class="tools-btn" @click="toggleAll">
        {{ allOpen ? '收起全部' : '展开全部' }}
        <AppIcon name="chevron" :class="{ flipped: allOpen }" />
      </button>
    </div>

    <p v-if="searching && totalMatched === 0" class="empty safe-x">
      没有匹配「{{ query.trim() }}」的角色。可以回首页用一句话描述「想和谁聊」，或者直接新建一个角色。
    </p>

    <section v-for="entry in groups" :key="entry.group.id" class="acc safe-x">
      <button class="acc-head" :aria-expanded="isOpen(entry.group.id)" @click="toggleGroup(entry.group.id)">
        <span class="acc-emoji" aria-hidden="true">{{ entry.group.emoji }}</span>
        <span class="acc-text">
          <span class="acc-title">{{ entry.group.label }}</span>
          <span class="acc-hint">{{ entry.group.hint }}</span>
        </span>
        <span class="acc-count">{{ entry.items.length }}</span>
        <span class="acc-chevron" :class="{ open: isOpen(entry.group.id) }"><AppIcon name="chevron" /></span>
      </button>

      <div class="acc-body" :class="{ open: isOpen(entry.group.id) }">
        <div class="acc-inner">
          <div class="acc-content">
            <template v-if="entry.group.id === 'custom'">
              <button class="new-card" @click="createNew">
                <span class="new-plus" aria-hidden="true"><AppIcon name="plus" /></span>
                <span class="new-text">
                  <span class="new-title">新建角色</span>
                  <span class="new-desc">给它一个身份、语气和禁忌</span>
                </span>
              </button>

              <div v-for="item in entry.items" :key="item.id" class="mine" :style="{ '--accent': item.accent }">
                <button class="mine-main" @click="pick(item.id)">
                  <span class="mine-avatar" aria-hidden="true">{{ item.name.slice(0, 1) }}</span>
                  <span class="mine-body">
                    <span class="mine-name">
                      {{ item.name }}
                      <span v-if="personas.isTemp(item)" class="temp-tag">临时</span>
                      <span v-else-if="switchMode && chat.currentPersona.id === item.id" class="temp-tag">当前</span>
                    </span>
                    <span class="mine-tagline">{{ item.tagline }}</span>
                  </span>
                </button>
                <button class="mine-action" @click="edit(item.id)">
                  {{ personas.isTemp(item) ? '保存' : '编辑' }}
                </button>
              </div>

              <p v-if="entry.items.length === 0" class="empty">
                还没有自建角色。也可以回首页，用一句话描述「想和谁聊」，即时生成一个临时角色。
              </p>
            </template>

            <div v-else class="grid">
              <PersonaCard
                v-for="item in entry.items"
                :key="item.id"
                :persona="item"
                :active="switchMode && chat.currentPersona.id === item.id"
                :sub-label="entry.group.categories.length > 1 ? categoryMeta(item.category).label : ''"
                @click="pick(item.id)"
              />
            </div>
          </div>
        </div>
      </div>
    </section>

    <p class="footnote safe-x">
      <AppIcon name="sparkle" class="footnote-icon" />
      <span>
        先贤被限定了「知识边界」——问到超出他时代的事物，他会坦承"此物吾所未闻"，再用自己的思想体系去理解。
        其余角色都活在当代，可以正常聊当下的事。
      </span>
    </p>

    <div class="tail" />
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
}

.head {
  align-items: flex-start;
}

.head-text {
  min-width: 0;
}

/* ---- 搜索框：图标、输入、清空按钮装在同一个圆角框里 ---- */
.search-wrap {
  margin: 0 max(20px, var(--safe-right)) var(--sp-3) max(20px, var(--safe-left));
  padding: 0;
}

.search {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  min-height: 46px;
  padding: 0 6px 0 14px;
  border-radius: var(--r-md);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}

.search:focus-within {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}

.search-icon {
  flex: none;
  font-size: 1.05rem;
  color: var(--text-3);
}

.search-input {
  flex: 1;
  min-width: 0;
  padding: 11px 0;
  border: none;
  background: transparent;
  font-size: var(--fs-sm);
  font-family: inherit;
  color: var(--text-1);
  outline: none;
  /* 去掉 Safari 给 type=search 的默认圆角与清除按钮，否则和自绘的清除按钮并排很怪 */
  -webkit-appearance: none;
  appearance: none;
}

.search-input::-webkit-search-cancel-button {
  display: none;
}

.search-clear {
  flex: none;
  width: 32px;
  height: 32px;
  display: grid;
  place-items: center;
  border-radius: var(--r-full);
  font-size: 0.875rem;
  color: var(--text-2);
  background: var(--surface-2);
}

.search-clear:active {
  background: var(--surface-3);
}

/* 分组筛选：横向滚动。合并成 7 组后基本能一屏放下 */
.chips {
  display: flex;
  gap: var(--sp-2);
  padding: 2px max(20px, var(--safe-right)) var(--sp-3) max(20px, var(--safe-left));
  overflow-x: auto;
  scrollbar-width: none;
}

.chips::-webkit-scrollbar {
  display: none;
}

.chip {
  flex: none;
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 7px 12px;
  border-radius: var(--r-full);
  font-size: var(--fs-xs);
  color: var(--text-2);
  background: var(--surface-1);
  border: 1px solid var(--border);
  white-space: nowrap;
  transition:
    background var(--dur) var(--ease),
    color var(--dur) var(--ease);
}

.chip.on {
  color: var(--brand-on);
  font-weight: 700;
  background: var(--grad-brand);
  border-color: transparent;
  box-shadow: var(--shadow-brand);
}

.chip-emoji {
  font-size: 0.8125rem;
}

.chip-count {
  min-width: 18px;
  padding: 0 5px;
  border-radius: var(--r-full);
  text-align: center;
  font-size: 0.625rem;
  font-weight: 700;
  background: color-mix(in srgb, currentColor 14%, transparent);
}

.warn {
  margin: 0 0 var(--sp-3);
  padding: 0 max(20px, var(--safe-left));
  font-size: var(--fs-xs);
  line-height: 1.65;
  color: var(--warning);
}

.tools {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 max(20px, var(--safe-right)) var(--sp-3) max(20px, var(--safe-left));
}

.tools-total {
  font-size: var(--fs-xs);
  color: var(--text-3);
}

.tools-total strong {
  color: var(--text-1);
}

.tools-btn {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  padding: 4px 0;
  font-size: var(--fs-xs);
  font-weight: 600;
  color: var(--brand);
}

.tools-btn .icon {
  transition: transform var(--dur) var(--ease);
}

.tools-btn .flipped {
  transform: rotate(180deg);
}

/* ---------- 折叠分组 ---------- */
.acc {
  padding: 0 max(20px, var(--safe-right)) 0 max(20px, var(--safe-left));
  margin-bottom: 10px;
}

.acc-head {
  width: 100%;
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  padding: 11px 14px 11px 11px;
  border-radius: var(--r-lg);
  text-align: left;
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  transition: background var(--dur-fast) var(--ease);
}

.acc-head:active {
  background: var(--surface-2);
}

.acc-emoji {
  display: grid;
  place-items: center;
  flex: none;
  width: 42px;
  height: 42px;
  border-radius: 14px;
  font-size: 1.3rem;
  line-height: 1;
  background: var(--surface-2);
  border: 1px solid var(--border);
}

.acc-text {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.acc-title {
  font-size: var(--fs-sm);
  font-weight: 700;
}

.acc-hint {
  font-size: var(--fs-xs);
  color: var(--text-3);
}

.acc-count {
  flex: none;
  min-width: 26px;
  padding: 1px 8px;
  border-radius: var(--r-full);
  text-align: center;
  font-size: var(--fs-xs);
  font-weight: 700;
  color: var(--brand);
  background: var(--brand-soft);
}

.acc-chevron {
  flex: none;
  font-size: 1rem;
  color: var(--text-3);
  transition: transform var(--dur) var(--ease);
}

.acc-chevron.open {
  transform: rotate(180deg);
}

/*
 * 展开动画用 grid-template-rows 0fr → 1fr：
 * 不需要事先知道内容高度，也不会像 max-height 那样要么卡顿要么提前停住。
 * visibility 让收起后的内容脱离 Tab 焦点顺序，延迟到动画结束再隐藏；
 * 展开时立即可见。--dur 在 prefers-reduced-motion 下是 1ms，动画自动退化为瞬时。
 */
.acc-body {
  display: grid;
  grid-template-rows: 0fr;
  visibility: hidden;
  transition:
    grid-template-rows var(--dur) var(--ease),
    visibility 0s linear var(--dur);
}

.acc-body.open {
  grid-template-rows: 1fr;
  visibility: visible;
  transition:
    grid-template-rows var(--dur) var(--ease),
    visibility 0s;
}

.acc-inner {
  overflow: hidden;
}

.acc-content {
  padding: 10px 0 4px;
}

.grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr);
  gap: 8px;
}

/* 「我的角色」：新建入口 */
.new-card {
  width: 100%;
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  margin-bottom: var(--sp-2);
  padding: 12px 14px;
  border-radius: var(--r-md);
  text-align: left;
  background: color-mix(in srgb, var(--brand) 6%, transparent);
  border: 1px dashed color-mix(in srgb, var(--brand) 45%, var(--border));
}

.new-card:active {
  background: var(--brand-soft);
}

.new-plus {
  display: grid;
  place-items: center;
  flex: none;
  width: 40px;
  height: 40px;
  border-radius: 13px;
  font-size: 1.2rem;
  color: var(--brand-on);
  background: var(--grad-brand);
  box-shadow: var(--shadow-brand);
}

.new-text {
  display: flex;
  flex-direction: column;
  gap: 2px;
  min-width: 0;
}

.new-title {
  font-size: var(--fs-sm);
  font-weight: 700;
}

.new-desc {
  font-size: var(--fs-xs);
  color: var(--text-3);
}

/* 「我的角色」：一行一个，右侧挂操作按钮 */
.mine {
  display: flex;
  align-items: stretch;
  gap: var(--sp-2);
  margin-bottom: var(--sp-2);
  border-radius: var(--r-md);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  overflow: hidden;
}

.mine-main {
  flex: 1;
  min-width: 0;
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  padding: 12px 12px 12px 14px;
  text-align: left;
}

.mine-main:active {
  background: var(--surface-2);
}

.mine-avatar {
  flex: none;
  width: 40px;
  height: 40px;
  display: grid;
  place-items: center;
  font-size: 1.05rem;
  font-weight: 700;
  font-family: Georgia, 'Songti SC', serif;
  border-radius: 13px;
  color: color-mix(in srgb, var(--accent) 62%, var(--text-1));
  background: color-mix(in srgb, var(--accent) 20%, var(--surface-1));
  border: 1px solid color-mix(in srgb, var(--accent) 34%, transparent);
}

.mine-body {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.mine-name {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: var(--fs-sm);
  font-weight: 700;
}

.mine-tagline {
  font-size: var(--fs-xs);
  color: var(--text-3);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.temp-tag {
  flex: none;
  font-size: 0.625rem;
  font-weight: 600;
  padding: 2px 7px;
  border-radius: var(--r-full);
  color: color-mix(in srgb, var(--accent) 62%, var(--text-1));
  background: color-mix(in srgb, var(--accent) 16%, transparent);
}

.mine-action {
  flex: none;
  padding: 0 var(--sp-4);
  font-size: var(--fs-xs);
  font-weight: 700;
  color: var(--brand);
  border-left: 1px solid var(--border);
}

.mine-action:active {
  background: var(--surface-2);
}

.empty {
  margin: 0;
  padding: var(--sp-4);
  border-radius: var(--r-md);
  font-size: var(--fs-xs);
  line-height: 1.7;
  color: var(--text-2);
  background: color-mix(in srgb, var(--surface-1) 70%, transparent);
  border: 1px dashed var(--border-strong);
}

p.empty.safe-x {
  margin: 0 max(20px, var(--safe-left)) var(--sp-3);
}

.footnote {
  display: flex;
  align-items: flex-start;
  gap: 8px;
  margin: var(--sp-5) max(20px, var(--safe-left)) 0;
  padding: 12px 14px;
  border-radius: var(--r-md);
  font-size: var(--fs-xs);
  line-height: 1.75;
  color: var(--text-2);
  background: color-mix(in srgb, var(--surface-1) 70%, transparent);
  border: 1px solid var(--border);
}

.footnote-icon {
  flex: none;
  margin-top: 3px;
  color: var(--brand);
}

.tail {
  height: var(--sp-6);
}
</style>
