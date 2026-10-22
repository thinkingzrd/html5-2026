<script setup lang="ts">
/**
 * 消息气泡
 * ------------------------------------------------------------------
 * 用户消息靠右、纯文本；助手消息靠左、Markdown 渲染。
 * 支持长按复制（自带触觉反馈），并用 -webkit-touch-callout 屏蔽
 * 系统原生的长按菜单，保持统一的应用内手感。
 */
import { computed } from 'vue'
import type { Persona } from '@/data/personas'
import type { StoredMessage } from '@/stores/chat'
import { useToast } from '@/composables/useToast'
import MarkdownView from './MarkdownView.vue'

const props = defineProps<{
  message: StoredMessage
  persona: Persona
  /** 是否是正在生成中的那一条（用于显示光标） */
  live?: boolean
}>()

const { show } = useToast()

const isUser = computed(() => props.message.role === 'user')
const isEmpty = computed(() => props.message.content.length === 0)

let pressTimer: number | undefined

function clearPress(): void {
  if (pressTimer !== undefined) {
    window.clearTimeout(pressTimer)
    pressTimer = undefined
  }
}

function onPressStart(): void {
  if (isEmpty.value) return
  clearPress()
  pressTimer = window.setTimeout(() => {
    pressTimer = undefined
    void copyMessage()
  }, 520)
}

async function copyMessage(): Promise<void> {
  const text = props.message.content
  if (!text) return
  try {
    await navigator.clipboard.writeText(text)
    // 轻微震动反馈，让"复制成功"这件事可感知
    navigator.vibrate?.(12)
    show('已复制全文', 'success')
  } catch {
    show('复制失败，请手动选择文本', 'error')
  }
}
</script>

<template>
  <div
    class="row"
    :class="isUser ? 'from-user' : 'from-assistant'"
    :style="{ '--accent': persona.accent }"
    @touchstart.passive="onPressStart"
    @touchend="clearPress"
    @touchmove="clearPress"
    @touchcancel="clearPress"
    @mousedown="onPressStart"
    @mouseup="clearPress"
    @mouseleave="clearPress"
    @contextmenu.prevent
  >
    <div v-if="!isUser" class="avatar" aria-hidden="true">
      {{ persona.name.slice(0, 1) }}
    </div>

    <div class="bubble" :class="{ live }">
      <template v-if="isUser">
        <p class="plain selectable">{{ message.content }}</p>
      </template>

      <template v-else>
        <MarkdownView v-if="message.content" :content="message.content" :streaming="live" />
        <div v-if="message.references?.length && message.content" class="references" @mousedown.stop @touchstart.stop>
          <p class="references-label">本轮参考资料 · 点击编号查看原文</p>
          <details v-for="(source, index) in message.references" :key="index">
            <summary>
              <span class="ref-no">{{ index + 1 }}</span
              ><span class="ref-title">{{ source.title || '未命名资料' }}</span>
            </summary>
            <p class="source-note">回答时保存的原文片段；资料删除后仍可核对。</p>
            <p class="source-content">{{ source.content }}</p>
          </details>
        </div>
        <span v-if="live && !message.content" class="thinking" role="status" aria-label="正在思考">
          <i class="dot" />
          <i class="dot" />
          <i class="dot" />
        </span>
        <span v-else-if="live" class="caret" aria-hidden="true" />
        <p v-else-if="message.status === 'incomplete'" class="incomplete-note">
          {{ message.content ? '回复未完成' : '未生成回复' }}
        </p>
      </template>
    </div>
  </div>
</template>

<style scoped>
.incomplete-note {
  display: inline-block;
  margin: 8px 0 0;
  padding: 2px 9px;
  border-radius: var(--r-full);
  color: var(--warning);
  background: var(--warning-soft);
  font-size: var(--fs-xs);
}
.references {
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px dashed var(--border-strong);
}
.references-label,
.source-note {
  color: var(--text-3);
  font-size: var(--fs-xs);
  margin: 0 0 6px;
}
.references details {
  margin-top: 6px;
  border-radius: var(--r-sm);
  background: var(--surface-2);
  border: 1px solid var(--border);
}
.references details[open] {
  padding-bottom: 8px;
}
.references summary {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 10px;
  list-style: none;
  color: var(--text-1);
  font-size: var(--fs-sm);
  cursor: pointer;
}
.references summary::-webkit-details-marker {
  display: none;
}
.ref-no {
  display: grid;
  place-items: center;
  flex: none;
  min-width: 20px;
  height: 20px;
  border-radius: 7px;
  color: var(--brand-on);
  background: var(--grad-brand);
  font-size: 0.6875rem;
  font-weight: 700;
}
.ref-title {
  min-width: 0;
  overflow-wrap: anywhere;
}
.references .source-note,
.source-content {
  margin-inline: 10px;
}
.source-content {
  margin-top: 0;
  margin-bottom: 0;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  max-height: 260px;
  overflow-y: auto;
  line-height: 1.7;
  font-size: var(--fs-sm);
  color: var(--text-2);
}
.row {
  display: flex;
  align-items: flex-start;
  gap: var(--sp-2);
  margin-bottom: var(--sp-4);
  -webkit-touch-callout: none;
}

.row.from-user {
  justify-content: flex-end;
}

/* 角色主色混入主题文字色：五套主题下都看得清 */
.avatar {
  flex: none;
  width: 34px;
  height: 34px;
  border-radius: 12px;
  display: grid;
  place-items: center;
  margin-top: 2px;
  font-size: 0.95rem;
  font-weight: 700;
  font-family: Georgia, 'Songti SC', serif;
  color: color-mix(in srgb, var(--accent) 62%, var(--text-1));
  background: color-mix(in srgb, var(--accent) 20%, var(--surface-1));
  border: 1px solid color-mix(in srgb, var(--accent) 34%, transparent);
}

.bubble {
  max-width: 82%;
  padding: 11px 15px;
  border-radius: 20px;
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
  animation: bubble-in var(--dur) var(--ease);
}

.from-user .bubble {
  color: var(--brand-on);
  background: var(--grad-brand);
  border-color: transparent;
  border-bottom-right-radius: 6px;
  box-shadow: 0 6px 18px color-mix(in srgb, var(--brand) 22%, transparent);
}

.from-user .plain::selection {
  background: color-mix(in srgb, var(--brand-on) 28%, transparent);
}

.from-assistant .bubble {
  border-bottom-left-radius: 6px;
}

@keyframes bubble-in {
  from {
    opacity: 0;
    transform: translateY(8px) scale(0.98);
  }
  to {
    opacity: 1;
    transform: none;
  }
}

.plain {
  margin: 0;
  font-size: var(--fs-md);
  line-height: 1.7;
  white-space: pre-wrap;
  word-break: break-word;
}

/* 流式光标：跟随在文字末尾闪烁 */
.caret {
  display: inline-block;
  width: 7px;
  height: 1.05em;
  margin-left: 2px;
  vertical-align: text-bottom;
  border-radius: 2px;
  background: var(--brand);
  animation: blink 1s steps(2, start) infinite;
}

@keyframes blink {
  50% {
    opacity: 0;
  }
}

/* 还没有任何内容时的三点等待动画 */
.thinking {
  display: inline-flex;
  gap: 5px;
  padding: 5px 2px;
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: var(--r-full);
  background: var(--brand);
  animation: bounce 1.3s var(--ease) infinite;
}

.dot:nth-child(2) {
  animation-delay: 0.16s;
}

.dot:nth-child(3) {
  animation-delay: 0.32s;
}

@keyframes bounce {
  0%,
  60%,
  100% {
    transform: translateY(0);
    opacity: 0.35;
  }
  30% {
    transform: translateY(-4px);
    opacity: 1;
  }
}
</style>
