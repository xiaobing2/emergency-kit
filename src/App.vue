<template>
  <div class="app">
    <header class="header">
      <div class="container header-inner">
        <div class="brand">
          <div class="brand-mark">🧭</div>
          <div>
            <div class="brand-name">应急包·速查助手</div>
            <div class="brand-sub">家庭 / 办公室 / 车载 一键盘点 + 演练提示</div>
          </div>
        </div>
        <div class="header-actions">
          <button class="btn ghost" @click="resetAll">重置清单</button>
          <button class="btn primary" @click="exportJson">导出 JSON</button>
        </div>
      </div>
    </header>

    <main class="container main">
      <section class="card hero">
        <div>
          <p class="pill">防患于未然</p>
          <h1>把“应急物资”从想法变成行动。</h1>
          <p class="muted">
            预设多场景清单（家庭/车载/办公），支持自定义与本地存储。还有 5
            步演练提示，帮助全家/团队一起“知道物资在哪、怎么用”。
          </p>
          <div class="tags">
            <span class="chip">离线可用</span>
            <span class="chip">本地存储</span>
            <span class="chip">可导出</span>
            <span class="chip">演练提示</span>
          </div>
        </div>
        <div class="hero-stats">
          <div class="stat">
            <div class="stat-label">已完成</div>
            <div class="stat-value">{{ doneCount }}</div>
            <div class="stat-sub muted">项物资已确认</div>
          </div>
          <div class="stat">
            <div class="stat-label">剩余</div>
            <div class="stat-value">{{ remainingCount }}</div>
            <div class="stat-sub muted">待补齐 / 待检查</div>
          </div>
          <div class="stat">
            <div class="stat-label">场景</div>
            <div class="stat-value">{{ currentScenarioLabel }}</div>
            <div class="stat-sub muted">可切换或自定义</div>
          </div>
        </div>
      </section>

      <section class="card">
        <div class="card-head">
          <div>
            <h2>场景选择</h2>
            <p class="muted tiny">切换后可再勾选/修改；数据存本地。</p>
          </div>
          <div class="scenario-list">
            <button
              v-for="s in scenarios"
              :key="s.key"
              class="chip"
              :class="{ active: scenario === s.key }"
              @click="applyScenario(s.key)"
            >
              {{ s.label }}
            </button>
          </div>
        </div>

        <div class="card-head">
          <div>
            <h2>应急物资清单</h2>
            <p class="muted tiny">按类别拆分，勾选或备注有效期/位置。</p>
          </div>
          <button class="btn ghost small" @click="addCustomItem">新增物资</button>
        </div>

        <div class="grid">
          <article v-for="cat in categories" :key="cat.key" class="box">
            <div class="box-head">
              <div class="box-title">{{ cat.label }}</div>
              <div class="muted tiny">{{ cat.desc }}</div>
            </div>
            <div class="list">
              <div v-for="item in itemsByCategory(cat.key)" :key="item.id" class="item">
                <label class="item-row">
                  <input type="checkbox" v-model="item.checked" @change="persist" />
                  <div class="item-main">
                    <div class="item-title">
                      {{ item.name }}
                      <span v-if="item.tag" class="tag">{{ item.tag }}</span>
                    </div>
                    <div class="muted tiny">{{ item.note || '可添加备注/有效期/存放位置' }}</div>
                  </div>
                </label>
                <div class="item-actions">
                  <input
                    class="note"
                    v-model="item.note"
                    placeholder="备注/有效期/位置"
                    @blur="persist"
                  />
                  <button class="mini danger" @click="removeItem(item.id)">删除</button>
                </div>
              </div>
            </div>
          </article>
        </div>
      </section>

      <section class="card drill">
        <div class="card-head">
          <h2>5 步演练提示</h2>
          <button class="btn ghost small" @click="shuffleDrill">换一组</button>
        </div>
        <ol class="drill-list">
          <li v-for="(step, i) in drillSteps" :key="i">
            <strong>步骤 {{ i + 1 }}</strong>
            <span>{{ step }}</span>
          </li>
        </ol>
        <p class="muted tiny">提示：尝试用 10 分钟完成一次小演练，确认每个人都知道物资位置与使用方法。</p>
      </section>
    </main>

    <footer class="footer">
      <div class="container footer-inner">
        <div>应急包·速查助手 | 仅本地存储，不上传数据</div>
        <div class="muted tiny">本项目由阿里云ESA提供加速、计算和保护</div>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { computed, reactive, ref, watch } from 'vue'
import { nanoid } from 'nanoid/non-secure'

const scenarios = [
  { key: 'home', label: '家庭', items: ['water', 'food', 'firstAid', 'tools', 'docs'] },
  { key: 'car', label: '车载', items: ['water', 'firstAid', 'tools', 'power', 'warm'] },
  { key: 'office', label: '办公室', items: ['water', 'food', 'firstAid', 'tools', 'comm'] },
  { key: 'custom', label: '自定义', items: ['water', 'food'] },
]

const baseItems = {
  water: [
    { name: '瓶装水 / 净水片', tag: '72 小时', cat: 'water' },
  ],
  food: [
    { name: '能量棒 / 压缩饼干', tag: '72 小时', cat: 'food' },
    { name: '糖 / 巧克力', cat: 'food' },
  ],
  firstAid: [
    { name: '创可贴 / 无菌纱布 / 胶带', cat: 'firstAid' },
    { name: '碘伏棉签 / 酒精棉片', cat: 'firstAid' },
    { name: '一次性手套 / 口罩', cat: 'firstAid' },
    { name: '常用药（感冒/止泻/退烧/过敏）', cat: 'firstAid' },
  ],
  tools: [
    { name: '手电 / 头灯 + 备用电池', cat: 'tools' },
    { name: '多功能刀 / 剪刀', cat: 'tools' },
    { name: '打火机 / 火柴', cat: 'tools' },
    { name: '胶带 / 扎带 / 绳子', cat: 'tools' },
  ],
  docs: [
    { name: '身份证/社保卡复印件', cat: 'docs' },
    { name: '现金小额', cat: 'docs' },
    { name: '紧急联系人卡片', cat: 'docs' },
  ],
  power: [
    { name: '车充 / 点烟器转接口', cat: 'power' },
    { name: '移动电源 + 线材', cat: 'power' },
  ],
  warm: [
    { name: '保暖毯 / 雨披', cat: 'warm' },
    { name: '一次性暖宝宝', cat: 'warm' },
  ],
  comm: [
    { name: '哨子 / 口哨', cat: 'comm' },
    { name: '对讲机（可选）', cat: 'comm' },
  ],
}

const categories = [
  { key: 'water', label: '饮水与食物', desc: '72 小时基础供给' },
  { key: 'food', label: '能量与补给', desc: '保质期标记，轮换使用' },
  { key: 'firstAid', label: '医疗与防护', desc: '消毒、包扎、常用药' },
  { key: 'tools', label: '照明与工具', desc: '手电、刀具、胶带' },
  { key: 'docs', label: '证件与信息', desc: '复印件、紧急联络' },
  { key: 'power', label: '供电与通讯', desc: '车充、移动电源' },
  { key: 'warm', label: '保暖与避雨', desc: '保暖毯、雨披' },
  { key: 'comm', label: '信号与呼救', desc: '哨子、对讲' },
]

const scenario = ref('home')
const items = ref([])

const drillPresets = [
  [
    '30 秒内指出家中灭火器/逃生绳/手电放哪。',
    '模拟停电，1 分钟内找到照明设备并打开。',
    '检查急救包有效期，标记将近过期的物品。',
    '把紧急联系人卡放进应急包，并同步到手机备注。',
    '练习对讲机或手机免提呼叫紧急联系人。',
  ],
  [
    '车上：确认三脚架、反光背心、灭火器位置与压力。',
    '车上：模拟电量告急，使用车充/移动电源充电。',
    '找到雨披/保暖毯，确认使用方式。',
    '检查备胎或补胎工具是否齐全（如有）。',
    '录一条 30 秒家庭应急语音，保存到手机快捷方式。',
  ],
]

const drillSteps = ref(drillPresets[0])

const currentScenarioLabel = computed(() => scenarios.find((s) => s.key === scenario.value)?.label || '自定义')
const doneCount = computed(() => items.value.filter((i) => i.checked).length)
const remainingCount = computed(() => items.value.filter((i) => !i.checked).length)

function itemsByCategory(cat) {
  return items.value.filter((i) => i.cat === cat)
}

function applyScenario(key) {
  scenario.value = key
  const def = scenarios.find((s) => s.key === key)
  const set = []
  def.items.forEach((cat) => {
    ;(baseItems[cat] || []).forEach((item) => {
      set.push({
        id: nanoid(6),
        ...item,
        note: '',
        checked: false,
      })
    })
  })
  items.value = set
  persist()
}

function addCustomItem() {
  items.value.push({
    id: nanoid(6),
    name: '自定义物资',
    cat: 'tools',
    note: '',
    checked: false,
  })
  persist()
}

function removeItem(id) {
  items.value = items.value.filter((i) => i.id !== id)
  persist()
}

function shuffleDrill() {
  const idx = Math.random() > 0.5 ? 0 : 1
  drillSteps.value = drillPresets[idx]
}

function resetAll() {
  applyScenario('home')
  shuffleDrill()
}

function exportJson() {
  const payload = {
    exportedAt: new Date().toISOString(),
    scenario: scenario.value,
    items: items.value,
  }
  const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `emergency-kit-${scenario.value}.json`
  a.click()
  URL.revokeObjectURL(url)
}

function persist() {
  const data = { scenario: scenario.value, items: items.value }
  localStorage.setItem('emergency_kit_v1', JSON.stringify(data))
}

function load() {
  const raw = localStorage.getItem('emergency_kit_v1')
  if (!raw) {
    resetAll()
    return
  }
  try {
    const d = JSON.parse(raw)
    scenario.value = d.scenario || 'home'
    items.value = d.items || []
    if (!items.value.length) applyScenario(scenario.value)
  } catch {
    resetAll()
  }
}

load()

watch(items, persist, { deep: true })
</script>
