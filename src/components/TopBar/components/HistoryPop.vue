<script setup lang="ts">
import { useDateFormat } from '@vueuse/core'
import type { Ref } from 'vue'
import { useI18n } from 'vue-i18n'

import Empty from '~/components/Empty.vue'
import Loading from '~/components/Loading.vue'
import Progress from '~/components/Progress.vue'
import type { HistoryResult, List as HistoryItem } from '~/models/history/history'
import { Business } from '~/models/history/history'
import api from '~/utils/api'
import { calcCurrentTime } from '~/utils/dataFormatter'
import { removeHttpFromUrl, scrollToTop } from '~/utils/main'

const { t } = useI18n()
const historys = reactive<Array<HistoryItem>>([])
const historyTabs = computed(() => [
  {
    id: 0,
    name: t('topbar.moments_dropdown.tabs.videos'),
    isSelected: true,
  },
  {
    id: 1,
    name: t('topbar.moments_dropdown.tabs.live'),
    isSelected: false,
  },
  {
    id: 2,
    name: t('topbar.moments_dropdown.tabs.articles'),
    isSelected: false,
  },
])
/**
 * Active tab (0: archive, 1: live, 2: article)
 */
const activatedTab = ref<number>(0)
const isLoading = ref<boolean>(false)
// when noMoreContent is true, the user can't scroll down to load more content
const noMoreContent = ref<boolean>(false)
const livePage = ref<number>(1)
const historysWrap = ref<HTMLElement>() as Ref<HTMLElement>

watch(activatedTab, (newVal: number | undefined, oldVal: number | undefined) => {
  if (newVal === oldVal)
    return

  historys.length = 0
  if (historysWrap.value)
    scrollToTop(historysWrap.value)

  if (newVal === 0) {
    getHistoryList(Business.ARCHIVE)
  }
  else if (newVal === 1) {
    livePage.value = 1
    getHistoryList(Business.LIVE)
  }
  else if (newVal === 2) {
    getHistoryList(Business.ARTICLE)
  }
}, { immediate: true })

onMounted(() => {
  if (historysWrap.value) {
    historysWrap.value.addEventListener('scroll', () => {
      // When you scroll to the bottom, they will automatically
      // add the next page of data to the history list
      if (
        historysWrap.value.clientHeight + historysWrap.value.scrollTop
        >= historysWrap.value.scrollHeight - 20
        && historys.length > 0
        && !isLoading.value
      ) {
        if (activatedTab.value === 0 && !noMoreContent.value) {
          getHistoryList(
            Business.ARCHIVE,
            historys[historys.length - 1].view_at,
          )
        }
        else if (activatedTab.value === 1 && !noMoreContent.value) {
          getHistoryList(
            Business.LIVE,
            historys[historys.length - 1].view_at,
          )
        }
        else if (activatedTab.value === 2 && !noMoreContent.value) {
          getHistoryList(
            Business.ARTICLE,
            historys[historys.length - 1].view_at,
          )
        }
      }
    })
  }
})

function onClickTab(tabId: number) {
  // Prevent changing tab when loading, cuz it will cause a bug
  if (isLoading.value)
    return

  activatedTab.value = tabId
  historyTabs.value.forEach((tab) => {
    tab.isSelected = tab.id === tabId
  })
}

/**
 * Return the URL of the history item
 * @param item history item
 * @return {string} url
 */
function getHistoryUrl(item: HistoryItem) {
  if (item.uri)
    return item.uri

  // Video
  if (item.history.business === Business.ARCHIVE) {
    if (item?.videos && item.videos > 0)
      return `//www.bilibili.com/video/${item.history.bvid}?p=${item.history.page}`
    return `//www.bilibili.com/video/${item.history.bvid}`
  }
  // Live
  else if (item.history.business === Business.LIVE) {
    return `//live.bilibili.com/${item.history.oid}`
  }
  // Article
  else if (item.history.business === Business.ARTICLE || item.history.business === Business.ARTICLE_LIST) {
    if (item.history.cid === 0)
      return `//www.bilibili.com/read/cv${item.history.oid}`
    else
      return `//www.bilibili.com/read/cv${item.history.cid}`
  }
  return ''
}

/**
 * Get history list
 * @param type
 * @param view_at Last viewed timestamp
 */
function getHistoryList(type: Business, view_at = 0 as number) {
  if (isLoading.value)
    return
  if (noMoreContent.value)
    return

  isLoading.value = true
  api.history.getHistoryList({
    type,
    view_at,
  })
    .then((res: HistoryResult) => {
      if (res.code === 0) {
        if (Array.isArray(res.data.list) && res.data.list.length > 0)
          historys.push(...res.data.list)

        if (res.data.list.length < 20) {
          noMoreContent.value = true
        }

        noMoreContent.value = false
      }
      isLoading.value = false
    })
}
</script>

<template>
  <div
    style="backdrop-filter: var(--bew-filter-glass-1);"
    bg="$bew-elevated"
    w="380px"
    rounded="$bew-radius"
    pos="relative"
    shadow="[var(--bew-shadow-edge-glow-1),var(--bew-shadow-3)]"
    border="1 $bew-border-color"
  >
    <!-- top bar -->
    <header
      style="backdrop-filter: var(--bew-filter-glass-1);"
      flex="~"
      justify="between"
      p="y-4 x-6"
      pos="fixed top-0 left-0"
      w="full"
      bg="$bew-elevated"
      z="2"
      border="!rounded-t-$bew-radius"
    >
      <div flex="~">
        <div
          v-for="tab in historyTabs"
          :key="tab.id"
          m="r-4"
          transition="all duration-300"
          class="tab"
          :class="tab.isSelected ? 'tab-selected' : ''"
          cursor="pointer"
          @click="onClickTab(tab.id)"
        >
          {{ tab.name }}
        </div>
      </div>
      <ALink
        href="https://www.bilibili.com/account/history"
        type="topBar"
        flex="~ items-center"
      >
        <span text="sm">{{ $t('common.view_all') }}</span>
      </ALink>
    </header>

    <!-- historys wrapper -->
    <main overflow-hidden rounded="$bew-radius">
      <div
        ref="historysWrap"
        flex="~ col gap-2"
        h="430px"
        overflow="y-scroll"
        p="x-4"
      >
        <!-- loading -->
        <Loading
          v-if="isLoading && historys.length === 0"
          h="full"
          flex="~ items-center"
        />

        <!-- empty -->
        <Empty
          v-if="!isLoading && historys.length === 0"
          pos="absolute top-0 left-0"
          bg="$bew-content"
          z="0" w="full" h="full"
          flex="~ items-center"
          rounded="$bew-radius"
        />

        <!-- historys -->

        <!-- Use a transparent `div` instead of `margin-top` to prevent the list item bouncing problem -->
        <!-- https://github.com/BewlyBewly/BewlyBewly/pull/889#issue-2394127922 -->
        <div v-if="!isLoading && historys.length > 0" min-h="50px" />

        <TransitionGroup name="list">
          <ALink
            v-for="historyItem in historys"
            :key="historyItem.kid"
            :href="getHistoryUrl(historyItem)"
            type="topBar"
            m="last:b-4" p="2"
            rounded="$bew-radius"
            hover:bg="$bew-fill-2"
            duration-300
          >
            <section flex="~ gap-4 item-start">
              <!-- Video cover, live cover, ariticle cover -->
              <div
                bg="$bew-skeleton"
                w="150px"
                flex="shrink-0"
                border="rounded-$bew-radius-half"
                overflow="hidden"
              >
                <!-- Video -->
                <template v-if="activatedTab === 0">
                  <div pos="relative">
                    <img
                      w="150px" h-full
                      class="aspect-video"
                      :src="`${removeHttpFromUrl(
                        historyItem.cover,
                      )}@256w_144h_1c`"
                      :alt="historyItem.title"
                      object-cover
                    >
                    <div
                      pos="absolute bottom-0 right-0"
                      bg="black opacity-60"
                      m="1"
                      p="x-2 y-1"
                      text="white xs"
                      border="rounded-full"
                    >
                      <!--  When progress = -1 means that the user watched the full video -->
                      {{
                        `${
                          historyItem.progress === -1
                            ? calcCurrentTime(historyItem.duration)
                            : calcCurrentTime(historyItem.progress)
                        } /
                    ${calcCurrentTime(historyItem.duration)}`
                      }}
                    </div>
                  </div>
                  <Progress
                    :percentage="
                      (historyItem.progress / historyItem.duration) * 100
                    "
                  />
                </template>

                <!-- Live -->
                <template v-else-if="activatedTab === 1">
                  <div pos="relative">
                    <img
                      w="150px"
                      class="aspect-video"
                      :src="`${removeHttpFromUrl(
                        historyItem.cover,
                      )}@256w_144h_1c`"
                      :alt="historyItem.title"
                      bg="contain"
                    >
                    <div
                      v-if="historyItem.live_status === 1"
                      pos="absolute top-0 left-0"
                      bg="$bew-error-color"
                      text="xs white"
                      p="x-2 y-1"
                      m="1"
                      rounded="$bew-radius-half"
                      font="semibold"
                    >
                      LIVE
                    </div>
                    <div
                      v-else
                      pos="absolute top-0 left-0"
                      bg="black opacity-60"
                      text="xs white"
                      p="x-2 y-1"
                      m="1"
                      rounded="$bew-radius-half"
                    >
                      OFFLINE
                    </div>
                  </div>
                </template>

                <!-- Article -->
                <div v-else-if="activatedTab === 2">
                  <img
                    w="150px"
                    class="aspect-video"
                    :src="`${
                      Array.isArray(historyItem.covers)
                        ? historyItem.covers[0]
                        : ''
                    }@256w_144h_1c`"
                    object-cover
                    :alt="historyItem.title"
                    bg="contain"
                  >
                </div>
              </div>

              <!-- Description -->
              <div>
                <h3
                  class="keep-two-lines"
                  overflow="hidden"
                  text="ellipsis"
                  break-anywhere
                >
                  {{ historyItem.title }}
                </h3>
                <div text="$bew-text-2 sm" m="t-4" flex="~" align="items-center">
                  <ALink
                    :href="`https://space.bilibili.com/${historyItem.author_mid}`"
                    type="topBar"
                  >
                    {{ historyItem.author_name }}
                  </ALink>
                  <span
                    v-if="historyItem.live_status === 1"
                    text="$bew-theme-color"
                    flex
                    items-center
                    gap-1
                    m="l-2"
                  ><div i-tabler:live-photo />
                    LIVE
                  </span>
                </div>
                <p text="$bew-text-2 sm">
                  {{
                    useDateFormat(
                      historyItem.view_at * 1000,
                      'YYYY-MM-DD HH:mm:ss',
                    ).value
                  }}
                </p>
              </div>
            </section>
          </ALink>
        </TransitionGroup>

        <!-- loading -->
        <Transition name="fade">
          <Loading v-if="isLoading && historys.length !== 0" m="-t-4" />
        </Transition>
      </div>
    </main>
  </div>
</template>

<style lang="scss" scoped>
.tab {
  --uno: "relative text-$bew-text-2";

  &::after {
    --uno: "absolute bottom-0 left-0 w-full h-12px bg-$bew-theme-color opacity-0 transform scale-x-0 -z-1";
    --uno: "transition-all duration-300";
    content: "";
  }
}

.tab-selected {
  --uno: "font-bold text-$bew-text-1";

  &::after {
    --uno: "scale-x-80 opacity-40";
  }
}
</style>
