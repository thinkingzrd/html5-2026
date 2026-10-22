<script setup lang="ts">
/**
 * 新建 / 编辑自定义角色
 * ------------------------------------------------------------------
 * 一个页面承担三件事，靠查询参数区分：
 *   /personas/edit            新建一个角色
 *   /personas/edit?id=xxx     编辑自己造的角色
 *   /personas/edit?id=temp_x  把「想和谁聊」生成的临时人设转正
 *
 * 表单只收四项：名称 / 身份 / 语气 / 禁忌 —— 与需求文档 FR-B3 一致。
 * 手机上填十几个输入框没人填得完，其余字段（traits、greeting、主色）
 * 全部由这四项推导，见 data/personas.ts 的 buildCustomPersona。
 *
 * 临时人设与正式角色在这里必须区别对待：临时人设「保存」是**新建**一个
 * 正式角色（临时的那份用完即弃），而不是就地改一个不存在于库里的对象。
 */
import { computed, reactive, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import AppIcon from '@/components/AppIcon.vue'
import { useToast } from '@/composables/useToast'
import { usePersonasStore } from '@/stores/personas'

const route = useRoute()
const router = useRouter()
const personas = usePersonasStore()
const { show } = useToast()

const sourceId = computed(() => (typeof route.query.id === 'string' ? route.query.id : ''))
const source = computed(() => (sourceId.value ? personas.find(sourceId.value) : null))
/** 来源是临时人设：保存等于新建 */
const fromTemp = computed(() => Boolean(source.value && personas.isTemp(source.value)))
/** 只有用户自建的角色才能就地编辑和删除 */
const editable = computed(() => Boolean(source.value && personas.isCustom(source.value)))
/** 带 id 进来但库里没有：被删过，或者链接失效 */
const missing = computed(() => Boolean(sourceId.value) && personas.ready && !source.value)

const AVATARS = ['✨', '🧑‍💻', '🧑‍🏫', '🎯', '🦊', '🌊', '🎨', '🍀', '🚀', '🧩']

const form = reactive({
  name: '',
  avatar: '✨',
  tagline: '',
  identity: '',
  speechStyle: '',
  taboos: ''
})

/** 表单只灌一次：source 会随 store 更新而变，反复重灌会冲掉用户正在输入的内容 */
const prefilled = ref(false)

watch(
  () => [personas.ready, sourceId.value, source.value?.id] as const,
  () => {
    const item = source.value
    if (prefilled.value || !item) return
    form.name = item.name
    form.avatar = item.avatar
    form.tagline = item.tagline
    form.identity = item.identity
    form.speechStyle = item.speechStyle.join('\n')
    form.taboos = item.taboos.join('\n')
    prefilled.value = true
  },
  { immediate: true }
)

const title = computed(() => {
  if (fromTemp.value) return '保存这个角色'
  if (editable.value) return '编辑角色'
  return '新建角色'
})

const submitText = computed(() => {
  if (fromTemp.value) return '存为我的角色'
  if (editable.value) return '保存修改'
  return '创建角色'
})

async function submit(): Promise<void> {
  if (personas.busy) return
  // 临时人设传 undefined，让 store 生成新 id；正式角色就地更新
  const saved = await personas.saveCustom({ ...form }, editable.value ? source.value?.id : undefined)
  if (!saved) return

  // 转正之后把临时的那份清掉，否则角色广场里会出现两个同名角色
  if (fromTemp.value && source.value) personas.removeTemp(source.value.id)

  show(fromTemp.value ? '已存到「我的角色」' : editable.value ? '已保存修改' : '角色已创建', 'success')
  await router.replace('/personas')
}

/** 删除是两步确认：第一次点只是「上膛」，避免误触把用户手写的角色弄丢 */
const armed = ref(false)
let armTimer = 0

function armDelete(): void {
  if (armed.value) return
  armed.value = true
  window.clearTimeout(armTimer)
  armTimer = window.setTimeout(() => {
    armed.value = false
  }, 3000)
}

async function remove(): Promise<void> {
  const item = source.value
  if (!item || !personas.isCustom(item) || !armed.value) return
  const ok = await personas.removeCustom(item.id)
  if (!ok) return
  show('角色已删除', 'success')
  await router.replace('/personas')
}

function goBack(): void {
  router.back()
}
</script>

<template>
  <div class="page scroll-y">
    <header class="head safe-x">
      <button class="back" aria-label="返回" @click="goBack">
        <AppIcon name="back" />
      </button>
      <div class="head-text">
        <h1 class="title">{{ title }}</h1>
        <p class="subtitle">只填四项就够了，剩下的交给系统推导</p>
      </div>
    </header>

    <p v-if="missing" class="warn safe-x"><AppIcon name="alert" />找不到这个角色，可能已被删除。下面按新建处理。</p>

    <section class="form safe-x">
      <div class="form-card">
        <div class="field">
          <span class="field-label">头像</span>
          <div class="avatar-row">
            <input v-model="form.avatar" class="input avatar-input" maxlength="8" aria-label="自定义头像" />
            <div class="avatar-picks">
              <button
                v-for="emoji in AVATARS"
                :key="emoji"
                class="avatar-pick"
                :class="{ on: form.avatar === emoji }"
                type="button"
                @click="form.avatar = emoji"
              >
                {{ emoji }}
              </button>
            </div>
          </div>
        </div>

        <label class="field">
          <span class="field-label">名称</span>
          <input v-model="form.name" class="input" maxlength="12" placeholder="例如：健身教练" />
        </label>

        <label class="field">
          <span class="field-label">一句话介绍</span>
          <input v-model="form.tagline" class="input" maxlength="30" placeholder="例如：帮我把训练计划排明白" />
        </label>
      </div>

      <div class="form-card">
        <label class="field">
          <span class="field-label">身份设定</span>
          <textarea
            v-model="form.identity"
            class="input area"
            rows="4"
            maxlength="300"
            placeholder="它是谁？做什么的？以什么立场说话？例如：你是一位有十年经验的健身教练，负责给零基础的人排可执行的训练计划。"
          />
        </label>

        <label class="field">
          <span class="field-label">说话语气（一行一条）</span>
          <textarea
            v-model="form.speechStyle"
            class="input area"
            rows="3"
            maxlength="400"
            placeholder="先问清我的目标和可用时间，再给方案&#10;说人话，不要堆专业名词"
          />
        </label>

        <label class="field">
          <span class="field-label">禁忌（一行一条）</span>
          <textarea
            v-model="form.taboos"
            class="input area"
            rows="3"
            maxlength="300"
            placeholder="不要给出我身体条件做不到的计划&#10;不要编造医学结论"
          />
        </label>
      </div>

      <p v-if="fromTemp" class="hint">
        这是由「想和谁聊」临时生成的角色。保存后会成为一个正式角色，长期留在「我的角色」里。
      </p>

      <p v-if="personas.errorText" class="error">{{ personas.errorText }}</p>

      <button class="primary-btn" :disabled="personas.busy" @click="submit">
        <AppIcon name="check" />{{ submitText }}
      </button>

      <button v-if="editable" class="danger-btn" :class="{ armed }" @click="armed ? remove() : armDelete()">
        <AppIcon name="trash" />{{ armed ? '再点一次确认删除' : '删除这个角色' }}
      </button>
    </section>

    <div class="tail" />
  </div>
</template>

<style scoped>
.page {
  height: 100%;
  padding-top: var(--safe-top);
}

.head-text {
  min-width: 0;
}

.warn {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0 max(20px, var(--safe-right)) var(--sp-4) max(20px, var(--safe-left));
  padding: 10px 12px;
  border-radius: var(--r-md);
  font-size: var(--fs-xs);
  line-height: 1.65;
  color: var(--text-1);
  background: var(--warning-soft);
  border: 1px solid var(--warning-line);
}

.warn .icon {
  flex: none;
  color: var(--warning);
}

.form {
  padding: 0 max(20px, var(--safe-right)) 0 max(20px, var(--safe-left));
}

.form-card {
  margin-bottom: 14px;
  padding: 16px 16px 4px;
  border-radius: var(--r-xl);
  background: var(--surface-1);
  border: 1px solid var(--border);
  box-shadow: var(--card-shadow);
}

.field {
  display: block;
  margin-bottom: var(--sp-4);
}

.field-label {
  display: block;
  margin-bottom: 7px;
  font-size: var(--fs-xs);
  font-weight: 600;
  color: var(--text-2);
}

.input {
  width: 100%;
  padding: 11px 13px;
  border-radius: var(--r-sm);
  background: var(--surface-2);
  border: 1px solid var(--border);
  font-size: var(--fs-sm);
  font-family: inherit;
  color: var(--text-1);
  outline: none;
  transition:
    border-color var(--dur) var(--ease),
    box-shadow var(--dur) var(--ease);
}

.input:focus {
  border-color: var(--brand);
  box-shadow: var(--focus-ring);
}

.area {
  resize: none;
  line-height: 1.65;
}

/* 头像：左边大预览可手输，右边一排候选 */
.avatar-row {
  display: flex;
  gap: var(--sp-3);
  align-items: center;
}

/* 提高优先级：触屏下全局会把输入框字号压成 16px，这里需要保留大号表情预览 */
.input.avatar-input {
  flex: none;
  width: 72px;
  height: 72px;
  padding: 0;
  border-radius: 22px;
  text-align: center;
  font-size: 2rem;
  background: var(--brand-soft);
  border-color: var(--brand-line);
}

.avatar-picks {
  flex: 1;
  min-width: 0;
  display: grid;
  grid-template-columns: repeat(5, minmax(0, 1fr));
  gap: 6px;
}

.avatar-pick {
  aspect-ratio: 1;
  display: grid;
  place-items: center;
  font-size: 1.125rem;
  border-radius: 11px;
  background: var(--surface-2);
  border: 1px solid var(--border);
  transition:
    transform var(--dur-fast) var(--ease),
    border-color var(--dur) var(--ease);
}

.avatar-pick:active {
  transform: scale(0.9);
}

.avatar-pick.on {
  border-color: var(--brand);
  background: var(--brand-soft);
  box-shadow: 0 0 0 2px color-mix(in srgb, var(--brand) 22%, transparent);
}

.hint {
  margin: 0 0 var(--sp-3);
  padding: 10px 12px;
  border-radius: var(--r-md);
  font-size: var(--fs-xs);
  line-height: 1.65;
  color: var(--text-2);
  background: var(--brand-soft);
}

.error {
  margin: 0 0 var(--sp-3);
  padding: 10px 12px;
  border-radius: var(--r-sm);
  font-size: var(--fs-xs);
  line-height: 1.6;
  color: var(--danger);
  background: var(--danger-soft);
  border: 1px solid var(--danger-line);
}

.primary-btn,
.danger-btn {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 14px;
  border-radius: var(--r-md);
  font-size: var(--fs-sm);
  font-weight: 700;
  transition:
    transform var(--dur-fast) var(--ease),
    opacity var(--dur) var(--ease);
}

.primary-btn:active,
.danger-btn:active {
  transform: scale(0.985);
}

.primary-btn {
  color: var(--brand-on);
  background: var(--grad-brand);
  box-shadow: var(--shadow-brand);
}

.primary-btn:disabled {
  opacity: 0.55;
  box-shadow: none;
}

.danger-btn {
  margin-top: var(--sp-3);
  color: var(--danger);
  background: var(--surface-1);
  border: 1px solid var(--danger-line);
}

.danger-btn.armed {
  color: var(--bg);
  background: var(--danger);
  border-color: transparent;
}

.tail {
  height: var(--sp-6);
}
</style>
