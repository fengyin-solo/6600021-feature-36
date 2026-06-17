<template>
  <div class="min-h-screen p-4 flex flex-col gap-4 max-w-6xl mx-auto">
    <h1 class="text-3xl font-bold text-purple-400">盲文翻译与触觉学习器</h1>

    <div class="flex gap-2">
      <button v-for="t in tabs" :key="t.id" @click="activeTab = t.id"
        class="px-4 py-2 rounded text-sm"
        :class="activeTab === t.id ? 'bg-purple-500 text-white' : 'bg-gray-800 text-gray-300 hover:bg-gray-700'">
        {{ t.label }}
      </button>
    </div>

    <!-- Translate -->
    <div v-if="activeTab === 'translate'" class="grid grid-cols-2 gap-4">
      <div class="bg-gray-900 rounded-xl p-4">
        <h3 class="text-purple-300 font-bold mb-2">文本输入</h3>
        <textarea v-model="store.inputText" @input="store.translate()"
          class="w-full h-32 bg-gray-800 rounded p-3 text-white resize-none" placeholder="输入英文文本..." />
      </div>
      <div class="bg-gray-900 rounded-xl p-4">
        <h3 class="text-purple-300 font-bold mb-2">盲文输出</h3>
        <div class="text-4xl tracking-wider text-purple-300 h-16">{{ store.brailleUnicode }}</div>
        <div class="flex flex-wrap gap-2 mt-3">
          <BrailleCell v-for="(dots, i) in store.brailleOutput" :key="i" :dots="dots" :size="40" />
        </div>
      </div>
    </div>

    <!-- Learn -->
    <div v-if="activeTab === 'learn'" class="grid grid-cols-2 gap-4">
      <div class="bg-gray-900 rounded-xl p-4 flex flex-col items-center gap-4">
        <h3 class="text-purple-300 font-bold">猜盲文</h3>
        <div v-if="!store.quizChar">
          <button @click="store.generateQuiz()" class="bg-purple-500 px-6 py-3 rounded-lg text-lg hover:bg-purple-400">
            开始训练
          </button>
        </div>
        <div v-else class="flex flex-col items-center gap-3">
          <div class="text-7xl font-bold text-purple-400">{{ store.quizChar }}</div>
          <div class="text-sm text-gray-400">点击下方 6 点阵选择对应盲文</div>
          <div class="grid grid-cols-2 gap-2 p-4 bg-gray-800 rounded-xl">
            <button v-for="d in 6" :key="d" @click="store.toggleDot(d)"
              class="w-14 h-14 rounded-full border-2 transition-all"
              :class="store.selectedDots.includes(d) ? 'bg-purple-500 border-purple-400 scale-110' : 'bg-gray-700 border-gray-600 hover:border-purple-400'">
              <span class="text-xs">{{ d }}</span>
            </button>
          </div>
          <button @click="store.checkQuizAnswer()" class="bg-purple-500 px-6 py-2 rounded hover:bg-purple-400">确认</button>
        </div>
      </div>
      <div class="bg-gray-900 rounded-xl p-4">
        <div class="flex justify-between mb-2">
          <h3 class="text-purple-300 font-bold">统计</h3>
          <button @click="store.resetScore()" class="text-red-400 text-xs hover:underline">重置</button>
        </div>
        <div class="grid grid-cols-3 gap-2 text-center mb-3">
          <div class="bg-gray-800 rounded p-2">
            <div class="text-2xl font-bold text-green-400">{{ store.score.correct }}</div>
            <div class="text-xs text-gray-400">正确</div>
          </div>
          <div class="bg-gray-800 rounded p-2">
            <div class="text-2xl font-bold text-red-400">{{ store.score.total - store.score.correct }}</div>
            <div class="text-xs text-gray-400">错误</div>
          </div>
          <div class="bg-gray-800 rounded p-2">
            <div class="text-2xl font-bold text-purple-400">{{ store.score.total ? Math.round(store.score.correct / store.score.total * 100) : 0 }}%</div>
            <div class="text-xs text-gray-400">正确率</div>
          </div>
        </div>
        <div class="space-y-1 max-h-48 overflow-y-auto">
          <div v-for="(h, i) in store.history.slice(0, 20)" :key="i"
            class="flex justify-between bg-gray-800 rounded p-2 text-sm"
            :class="h.correct ? 'border-l-4 border-green-500' : 'border-l-4 border-red-500'">
            <span>{{ h.input }}</span><span>{{ h.correct ? '✓' : '✗' }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Reference -->
    <div v-if="activeTab === 'ref'" class="bg-gray-900 rounded-xl p-4 flex gap-4 relative">
      <div class="flex-1">
        <h3 class="text-purple-300 font-bold mb-3">盲文速查表</h3>
        <div ref="scrollContainer" class="h-[600px] overflow-y-auto pr-2 scroll-smooth" @scroll="onScroll">
          <div v-for="group in brailleGroups" :key="group.id" :id="'group-' + group.id" class="mb-6">
            <div class="sticky top-0 z-10 bg-gray-900 py-2 border-b border-purple-500/30 mb-3 transition-all"
              :class="activeGroup === group.id ? 'bg-purple-900/40 border-b-2 border-purple-500' : ''">
              <h4 class="text-lg font-bold" :class="activeGroup === group.id ? 'text-purple-300' : 'text-gray-300'">
                {{ group.label }}
              </h4>
            </div>
            <div class="grid grid-cols-5 md:grid-cols-8 lg:grid-cols-10 gap-3">
              <div v-for="char in group.chars" :key="char" class="flex flex-col items-center p-2 rounded-lg transition-all"
                :class="activeGroup === group.id ? 'bg-purple-900/20' : 'bg-gray-800/50'">
                <div class="text-xl font-bold text-purple-400">{{ char }}</div>
                <BrailleCell :dots="brailleMap[char]" :size="30" />
                <div class="text-xs text-gray-500">{{ brailleMap[char].join(',') }}</div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="flex flex-col gap-1 sticky top-4 self-start">
        <button v-for="group in brailleGroups" :key="group.id" @click="scrollToGroup(group.id)"
          class="px-3 py-2 text-sm rounded-lg text-left transition-all whitespace-nowrap"
          :class="activeGroup === group.id 
            ? 'bg-purple-500 text-white font-bold shadow-lg shadow-purple-500/30' 
            : 'bg-gray-800 text-gray-400 hover:bg-gray-700 hover:text-gray-200'">
          {{ group.label }}
        </button>
      </div>
    </div>

    <button @click="doExport" class="bg-green-700 px-4 py-2 rounded self-start hover:bg-green-600 text-sm">
      导出翻译文本
    </button>
  </div>
</template>

<script setup lang="ts">
import { ref, nextTick } from 'vue'
import { useBrailleStore } from './store/braille'
import { BRAILLE_MAP, BRAILLE_GROUPS } from './utils/braille'
import BrailleCell from './components/BrailleCell.vue'

const store = useBrailleStore()
const brailleMap = BRAILLE_MAP
const brailleGroups = BRAILLE_GROUPS
const tabs = [
  { id: 'translate', label: '翻译模式' },
  { id: 'learn', label: '训练模式' },
  { id: 'ref', label: '速查表' },
]
const activeTab = ref('translate')
const scrollContainer = ref<HTMLElement | null>(null)
const activeGroup = ref('aj')

function onScroll() {
  if (!scrollContainer.value) return
  const container = scrollContainer.value
  const scrollTop = container.scrollTop
  const containerTop = container.getBoundingClientRect().top

  let currentGroup = brailleGroups[0].id
  for (const group of brailleGroups) {
    const el = document.getElementById('group-' + group.id)
    if (!el) continue
    const rect = el.getBoundingClientRect()
    const relativeTop = rect.top - containerTop
    if (relativeTop <= 60) {
      currentGroup = group.id
    }
  }
  activeGroup.value = currentGroup
}

async function scrollToGroup(groupId: string) {
  await nextTick()
  const el = document.getElementById('group-' + groupId)
  if (el && scrollContainer.value) {
    const container = scrollContainer.value
    const containerTop = container.getBoundingClientRect().top
    const elTop = el.getBoundingClientRect().top
    container.scrollBy({
      top: elTop - containerTop,
      behavior: 'smooth'
    })
    activeGroup.value = groupId
  }
}

function doExport() {
  const text = store.exportPDF()
  const blob = new Blob([text], { type: 'text/plain' })
  const a = document.createElement('a')
  a.href = URL.createObjectURL(blob)
  a.download = 'braille-output.txt'
  a.click()
}
</script>
