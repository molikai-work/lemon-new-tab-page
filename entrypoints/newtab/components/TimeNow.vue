<!-- 首页时间组件 -->
<script lang="ts" setup>
// 引入必要的库和功能
import { browser } from 'wxt/browser'  // 获取浏览器语言
import { type ComputedRef, ref, watch } from 'vue'  // Vue的响应式和计算属性功能
import { useDateFormat, useElementHover, useNow } from '@vueuse/core'  // 格式化日期、元素hover状态和获取当前时间等等

// 从store中引入用户设置
import { useSettingsStore } from '@/newtab/scripts/store'

// 初始化设置store
const settingsStore = useSettingsStore()

// 定义当前时间的响应式变量
const time = ref()  // 用于存储DOM元素的引用，后面用于控制hover状态时的样式
const isTimeHovered = useElementHover(time)  // 检测时间元素是否被hover

// 监听时间元素的hover状态，进行样式变化
/*watch(isTimeHovered, (isTimeHovered) => {
  if (isTimeHovered) {
    time.value.style.transform = 'scale(1.1)'  // hover时放大元素
  } else {
    time.value.style.transform = null  // 离开hover时恢复原样
  }
})*/

// 自定义的时间段返回文字，如“深夜”、“早上”等
function customMeridiem(hours: number) {
  if (hours < 2) {
    return '深夜'
  } else if (hours < 7) {
    return '凌晨'
  } else if (hours < 11) {
    return '早上'
  } else if (hours < 14) {
    return '中午'
  } else if (hours < 17) {
    return '下午'
  } else if (hours < 19) {
    return '傍晚'
  } else if (hours < 23) {
    return '晚上'
  } else {
    return '深夜'
  }
}

// 获取当前时间（每秒更新一次）
const timeNow = useNow({ interval: 1000 })
// 获取浏览器当前语言（用于国际化）
const lang = browser.i18n.getUILanguage()
// 判断是否为中文语言环境
const isChinese = lang.startsWith('zh')

// 使用VueUse的useDateFormat来格式化当前时间
const timeNowHour: ComputedRef<string> = useDateFormat(timeNow, 'HH')  // 24小时制的小时
const timeNowHourMeridiem: ComputedRef<string> = useDateFormat(timeNow, 'h')  // 12小时制的小时
const timeNowMinute = useDateFormat(timeNow, 'mm')  // 分钟
const timeNowMeridiemZH = useDateFormat(timeNow, 'aa', { customMeridiem })  // 中文的午前/下午（使用自定义的meridiem）
const timeNowMeridiem = useDateFormat(timeNow, 'A', { locales: lang })  // 英文的AM/PM
const timeNowWeekday = useDateFormat(timeNow, 'dddd')  // 当前星期几（如Monday）

// 获取农历日期
function getlunarCalendar() {
  const lunarCalendarMatchRes = /([^年]{1,2}月.{2})/.exec(
    timeNow.value.toLocaleDateString(isChinese ? lang : 'zh', {
      dateStyle: 'long',
      calendar: 'chinese'  // 使用农历日历
    })
  )
  return lunarCalendarMatchRes ? lunarCalendarMatchRes[0] : ''  // 返回农历字符串
}
</script>

<template>
  <div
    ref="time"
    class="clock"
    :class="[  // 根据用户设置动态添加class
      settingsStore.time.enableShadow ? 'shadow' : '',  // 根据设置是否启用阴影
      settingsStore.background.bgType === 0 ? 'dark' : ''  // 根据背景类型是否为暗色模式
    ]"
  >
    <div class="time">
      <!-- 显示午前/下午（中文时显示自定义的Meridiem） -->
      <span
        v-if="settingsStore.time.showMeridiem && isChinese"
        class="meridiem"
        style="margin-right: 5px"
      >
        {{ timeNowMeridiemZH }}
      </span>

      <!-- 显示当前小时和分钟 -->
      <span>
        <span class="hour">
          {{
            settingsStore.time.isMeridiem ? timeNowHourMeridiem : timeNowHour
          }}
        </span>
        <span class="colon">:</span>
        <span class="minute">{{ timeNowMinute }}</span>
      </span>

      <!-- 显示午前/下午（英文时显示英文AM/PM） -->
      <span
        v-if="settingsStore.time.showMeridiem && !isChinese"
        class="meridiem"
        style="margin-left: 5px"
      >
        {{ timeNowMeridiem }}
      </span>
    </div>

    <!-- 显示当前日期和星期 -->
    <div v-if="settingsStore.time.showDate" class="date">
      <span>
        {{
          timeNow.toLocaleDateString(undefined, {
            dateStyle: 'long'  // 显示长日期格式（如：2024年12月25日）
          })
        }}
        {{ timeNowWeekday }}  <!-- 显示星期 -->
      </span>
      <span v-if="settingsStore.time.showLunar && isChinese">{{ ` ${getlunarCalendar()}` }}</span>  <!-- 显示农历（如果设置开启） -->
    </div>
  </div>
</template>

<style scoped lang="scss">
/* 定义闪烁效果的动画 */
@keyframes twinkle {
  0% {
    opacity: 0.5;
  }
  50% {
    opacity: 1;
  }
  100% {
    opacity: 0.5;
  }
}

/* 定义延迟渐变动画 */
@keyframes delayedFadeIn {
  0% {
    opacity: 0;
  }
  50% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}

/* 时钟的样式 */
.clock {
  text-align: center;
  color: var(--el-fill-color-blank);  // 默认文本颜色
  animation: delayedFadeIn 0.5s;  // 进入时的渐变动画
  transition:  // 样式变化的过渡动画
    font-size 0.25s cubic-bezier(0.5, 0, 0.5, 2),
    transform 0.25s cubic-bezier(0.5, 0, 0.5, 2),
    text-shadow var(--el-transition-duration-fast) ease,
    color var(--el-transition-duration-fast) ease;

  /* 启用阴影样式 */
  &.shadow {
    text-shadow: 0 6px 16px rgba(0, 0, 0, 0.4);
  }

  .time {
    font-size: 60px;  // 时间显示的字体大小
  }

  .date {
    margin-bottom: 5px;  // 日期和星期的底部间距
  }

  &.dark,
  html.dark & {
    color: var(--el-text-color-primary);  // 如果是暗色模式，字体颜色改变
  }

  .meridiem {
    font-size: 40px;  // 上午/下午标识的字体大小
  }

  .colon {
    animation: twinkle 1s ease infinite;  // 冒号的闪烁动画
  }
}
</style>
