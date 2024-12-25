<script lang="ts" setup>
import { Search } from '@vicons/fa'
import type { TooltipInstance } from 'element-plus'
import {
  onClickOutside,
  useActiveElement,
  useElementSize,
  useTimeoutFn,
  useWindowFocus
} from '@vueuse/core'
import { onMounted, ref, watch } from 'vue'

import { i18n } from '@/.wxt/i18n'
import { searchEngines } from '@/newtab/scripts/api/search'
import { searchHistoriesStorage } from '@/newtab/scripts/storages/searchStorages'
import { useFocusStore, useSettingsStore } from '@/newtab/scripts/store'

import SearchEngineMenu from './components/SearchEngineMenu.vue'
import SearchSuggestionArea from './components/SearchSuggestionArea.vue'

// 引用元素
const searchBox = ref<HTMLDivElement>()
const searchForm = ref<HTMLFormElement>()
const searchInput = ref<HTMLInputElement>()
const suggedtionArea = ref<InstanceType<typeof SearchSuggestionArea>>()
const searchEngineMenuRef = ref<TooltipInstance>()

// 搜索文本
const searchText = ref('')
const originSearchText = ref<string | null>(null) // 保存原始搜索文本，用于上下键操作

// 组件状态
const mounted = ref(false)

// 状态管理
const focusStore = useFocusStore()
const settingsStore = useSettingsStore()
const isWindowFocused = useWindowFocus()
const activeElement = useActiveElement()

// 获取表单宽度
const { width: searchFormWidth } = useElementSize(searchForm)

// URL 正则匹配
const urlRegex = /^(https?:\/\/)?([a-zA-Z0-9-]+\.)+[a-zA-Z]{2,6}(\/[^\s]*)?$/

// 监控窗口焦点
watch(isWindowFocused, (isFocused) => {
  if (searchText.value.length > 0 || isFocused) return
  searchText.value = ''
  originSearchText.value = ''
  if (suggedtionArea.value) {
    suggedtionArea.value.clearSearchSuggestions()
  }
  searchForm.value?.classList.remove('focus')
  searchInput.value?.blur()
  focusStore.blur()
})

// 监听点击外部
onClickOutside(searchBox, (e) => {
  if (activeElement.value?.classList.contains('search-engine-menu')) {
    searchInput.value?.focus()
    searchEngineMenuRef.value?.hide()
    return
  }
  if ((e.target as HTMLElement).localName !== 'main') {
    return
  }
  searchText.value = ''
  originSearchText.value = ''
  if (suggedtionArea.value) {
    suggedtionArea.value.clearSearchSuggestions()
  }
  searchForm.value?.classList.remove('focus')
  focusStore.blur()
})

// 聚焦处理
function handleFocus() {
  searchForm.value?.classList.add('focus')
  suggedtionArea.value!.showSearchHistories()
  focusStore.focus()
}

// 上键处理
function handleUp() {
  if (suggedtionArea.value!.searchSuggestions.length <= 0) return
  const _current = suggedtionArea.value!.currentActiveSuggest
  suggedtionArea.value!.clearActiveSuggest()
  if (originSearchText.value === null) originSearchText.value = searchText.value
  if (_current === null) activeOneSuggest(suggedtionArea.value!.searchSuggestions.length - 1)
  else if (_current === 0) {
    searchText.value = originSearchText.value || ''
    originSearchText.value = ''
    suggedtionArea.value!.currentActiveSuggest = null
  } else activeOneSuggest(_current - 1)
}

// 下键处理
function handleDown() {
  if (suggedtionArea.value!.searchSuggestions.length <= 0) return
  const _current = suggedtionArea.value!.currentActiveSuggest
  suggedtionArea.value!.clearActiveSuggest()
  if (originSearchText.value === null) originSearchText.value = searchText.value
  if (_current === null) activeOneSuggest(0)
  else if (_current === suggedtionArea.value!.searchSuggestions.length - 1) {
    searchText.value = originSearchText.value || ''
    originSearchText.value = ''
    suggedtionArea.value!.currentActiveSuggest = null
  } else activeOneSuggest(_current + 1)
}

// 激活一个建议
function activeOneSuggest(index: number) {
  const suggestions = suggedtionArea.value!.searchSuggestionArea?.children
  if (!suggestions) return
  suggestions[index].classList.add('active')
  suggedtionArea.value!.currentActiveSuggest = index
  searchText.value = suggedtionArea.value!.searchSuggestions[index]
}

// 选择搜索引擎
function selectSearch(index: number) {
  settingsStore.search.selectedSearchEngine = index
}

// 切换到上一个搜索引擎
function handlePrevTab() {
  if (settingsStore.search.selectedSearchEngine === 0) selectSearch(searchEngines.length - 1)
  else selectSearch(settingsStore.search.selectedSearchEngine - 1)
}

// 切换到下一个搜索引擎
function handleNextTab() {
  if (settingsStore.search.selectedSearchEngine === searchEngines.length - 1) selectSearch(0)
  else selectSearch(settingsStore.search.selectedSearchEngine + 1)
}

// 执行搜索
async function doSearch() {
  doSearchWithText(searchText.value)
  searchText.value = ''
}

// 执行文本搜索
async function doSearchWithText(text: string) {
  if (text.length <= 0) {
    // 如果没有输入内容，跳转到搜索引擎主页
    searchInput.value?.focus()
    return
  }

  // 如果历史搜索功能启用，记录搜索历史
  if (settingsStore.search.recordSearchHistory) {
    const searchHistories: string[] = await searchHistoriesStorage.getValue()
    if (searchHistories.includes(text)) {
      const index = searchHistories.indexOf(text)
      searchHistories.splice(index, 1)
      searchHistories.unshift(text)
    } else {
      searchHistories.unshift(text)
    }

    if (searchHistories.length > 15) {
      searchHistories.splice(15)
    }
    await searchHistoriesStorage.setValue(searchHistories)
  }

  // 根据选定的搜索引擎进行搜索
  window.open(
    searchEngines[settingsStore.search.selectedSearchEngine].url.replace('%s', text),
    settingsStore.search.searchInNewTab ? '_blank' : '_self'
  )
  suggedtionArea.value!.clearSearchSuggestions()
}

// 判断输入是否是URL或域名
function isURLOrDomain(input: string): boolean {
  return urlRegex.test(input)
}

// 获取翻译链接
function getTranslateLink(text: string): string {
  const selectedEngine = searchEngines[settingsStore.search.selectedSearchEngine]
  if (!selectedEngine) return ''

  // 生成翻译页面的URL
  return selectedEngine.translateUrl.replace('%s', encodeURIComponent(text))
}

// 获取翻译建议
function getTranslateSuggestion() {
  if (isURLOrDomain(searchText.value.trim())) return null
  const translateText = `翻译 ${searchText.value}`
  const translateLink = getTranslateLink(searchText.value)
  return {
    text: translateText,
    link: translateLink
  }
}

// 延时组件渲染
onMounted(() => useTimeoutFn(() => (mounted.value = true), 100))
</script>


<template>
  <section ref="searchBox" class="search-box">
    <form
      ref="searchForm"
      class="search-form"
      :class="[settingsStore.search.enableShadow ? 'shadow' : '', settingsStore.background.bgType === 0 ? 'dark' : '']"
      :style="{ '--width': mounted ? '' : '0' }"
      @submit.prevent="doSearch"
    >
      <search-engine-menu />
      <input
        ref="searchInput"
        v-model="searchText"
        :placeholder="focusStore.isFocused ? '' : i18n.t('newtab.search.placeholder')"
        class="search-input"
        @input="suggedtionArea!.handleInput"
        @focus="handleFocus"
        @keydown.up.prevent="handleUp"
        @keydown.down.prevent="handleDown"
        @keydown.tab.shift.prevent.exact="handlePrevTab"
        @keydown.tab.prevent.exact="handleNextTab"
        :autofocus="settingsStore.search.autoFocus"
      />
      <div class="search-btn">
        <el-icon @click="doSearch"><search /></el-icon>
      </div>
    </form>

    <!-- 翻译建议 -->
    <template v-if="getTranslateSuggestion()">
      <div class="translate-suggestion">
        <a :href="getTranslateSuggestion().link" target="_blank">{{ getTranslateSuggestion().text }}</a>
      </div>
    </template>

    <!-- 搜索建议区域 -->
    <search-suggestion-area
      ref="suggedtionArea"
      :search-text="searchText"
      :origin-search-text="originSearchText"
      :search-form-width="searchFormWidth"
      @do-search-with-text="doSearchWithText"
    />
  </section>
</template>


<style lang="scss" scoped>
.search-box {
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
  --cubic-bezier: cubic-bezier(0.65, 0.05, 0.1, 1);
  position: relative;
}

.search-form {
  --height: 44px;
  --width: 300px;
  --search-form-placeholder-color: var(--el-text-color-regular);
  height: var(--height);
  width: var(--width);
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: relative;
  border-radius: calc(var(--height) / 2);
  font-size: 15px;
  backdrop-filter: blur(10px);
  background-color: color-mix(in oklab, var(--el-fill-color), transparent 60%);
  color: transparent;
  transition:
    background-color var(--el-transition-duration-fast) ease,
    box-shadow var(--el-transition-duration-fast) ease,
    border var(--el-transition-duration-fast) ease,
    width var(--el-transition-duration-fast) var(--cubic-bezier);

  &.shadow {
    box-shadow: var(--el-box-shadow-dark);
  }

  html.dark & {
    --search-form-placeholder-color: var(--el-text-color-secondary);

    &.shadow {
      box-shadow: var(--el-box-shadow-light);
    }
  }

  &:hover:not(.focus) {
    --width: 500px;
    background-color: color-mix(in oklab, var(--el-fill-color), transparent 40%);
    --search-form-placeholder-color: var(--el-text-color-primary);
  }

  &.focus {
    background-color: color-mix(in oklab, var(--el-fill-color), transparent 20%);
    --width: 500px;
  }

  html:not(.dark) &.dark {
    background-color: var(--el-fill-color-blank);
    border: solid 1px var(--el-border-color-light);
  }

  :deep() .search-engine-icon,
  :deep() .search-btn {
    display: flex;
    align-items: center;
    justify-content: center;
    margin: 5px;
    height: calc(var(--height) - 10px);
    width: calc(var(--height) - 10px);
    overflow: hidden;
    border-radius: 50%;
    transition:
      color var(--el-transition-duration-fast) ease,
      background-color var(--el-transition-duration-fast) ease;
  }

  &.focus:deep() {
    .search-engine-icon,
    .search-btn {
      cursor: pointer;

      &:hover {
        background: white;
      }
    }
    .search-engine-icon {
      color: var(--el-text-color-regular);
    }
    .search-btn {
      color: var(--el-color-primary);
    }
  }

  .search-input {
    height: 100%;
    width: calc(100% - 2 * var(--height) - 20px);
    color: var(--el-text-color-primary);
    text-align: center;
    font-size: 1em;
    outline: none;
    border: none;
    background: none;
    transition:
      width var(--el-transition-duration-fast) var(--cubic-bezier),
      color var(--el-transition-duration-fast) ease;

    &::placeholder {
      color: var(--search-form-placeholder-color);
      -moz-user-select: none;
      -webkit-user-select: none;
      -ms-user-select: none;
      -khtml-user-select: none;
      user-select: none;
    }
  }

  .translate-suggestion {
    text-align: center;
    margin-top: 10px;
    font-size: 14px;
    color: var(--el-text-color-regular);

    a {
      color: var(--el-color-primary);
      text-decoration: none;

      &:hover {
        text-decoration: underline;
      }
    }
  }
}
</style>
