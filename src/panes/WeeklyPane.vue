<script setup lang="ts">
import { onMounted, ref, computed, watch } from 'vue'
import { GetWeeklyInfoRespData, commands } from '../bindings.ts'
import { NAlert, NButton, NEmpty, NSelect, NTabPane, NTabs, SelectProps } from 'naive-ui'
import ComicCard from '../components/ComicCard.vue'
import { useStore } from '../store.ts'

const store = useStore()

const weeklyInfo = ref<GetWeeklyInfoRespData>()

const selectedCategoryId = ref<string>('')
const currentWeeklyTypeId = ref<string>('')
const loading = ref(false)
const errorMessage = ref('')

const categoryOptions = computed<SelectProps['options']>(() =>
  weeklyInfo.value?.categories.map((category) => ({
    label: category.time,
    value: category.id,
  })),
)

onMounted(() => {
  refreshWeekly()
})

watch(
  () => [selectedCategoryId.value, currentWeeklyTypeId.value],
  () => {
    if (!loading.value) {
      loadWeekly()
    }
  },
  // Selection updates during refresh must not trigger another request.
  { flush: 'sync' },
)

async function refreshWeekly() {
  await loadWeekly(true)
}

async function loadWeekly(refreshInfo = false) {
  if (loading.value) {
    return
  }

  loading.value = true
  errorMessage.value = ''
  store.getWeeklyResult = undefined

  try {
    if (refreshInfo || weeklyInfo.value === undefined) {
      const result = await commands.getWeeklyInfo()
      if (result.status === 'error') {
        throw result.error
      }

      weeklyInfo.value = result.data
      if (!result.data.categories.some((category) => category.id === selectedCategoryId.value)) {
        selectedCategoryId.value = result.data.categories[0]?.id ?? ''
      }
      if (!result.data.type.some((type) => type.id === currentWeeklyTypeId.value)) {
        currentWeeklyTypeId.value = result.data.type[result.data.type.length - 1]?.id ?? ''
      }
    }

    if (!selectedCategoryId.value || !currentWeeklyTypeId.value) {
      return
    }

    const result = await commands.getWeekly(selectedCategoryId.value, currentWeeklyTypeId.value)
    if (result.status === 'error') {
      throw result.error
    }
    store.getWeeklyResult = result.data
  } catch (error) {
    console.error(error)
    errorMessage.value =
      typeof error === 'object' && error !== null && 'message' in error ? String(error.message) : String(error)
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="h-full flex flex-col">
    <div class="flex box-border px-2 pt-2 gap-2">
      <n-select
        v-if="categoryOptions?.length"
        class="flex-1 min-w-0"
        v-model:value="selectedCategoryId"
        :options="categoryOptions"
        :disabled="loading"
        :show-checkmark="false"
        size="small" />
      <n-button size="small" type="primary" secondary :loading="loading" :disabled="loading" @click="refreshWeekly">
        刷新
      </n-button>
    </div>
    <n-alert v-if="errorMessage" class="box-border m-2" type="error" title="每周必看加载失败">
      {{ errorMessage }}
    </n-alert>
    <div v-if="loading" class="p-2" role="status">正在加载每周必看…</div>
    <n-empty
      v-if="!loading && !errorMessage && !store.getWeeklyResult?.list.length"
      class="p-2"
      description="暂无每周必看内容，可点击刷新重试" />
    <n-tabs
      v-if="weeklyInfo !== undefined"
      class="h-full overflow-auto mb-2"
      v-model:value="currentWeeklyTypeId"
      type="line"
      size="small">
      <n-tab-pane
        class="h-full overflow-auto"
        v-for="weeklyType in weeklyInfo.type"
        :key="weeklyType.id"
        :name="weeklyType.id"
        :tab="weeklyType.title"
        :disabled="loading">
        <div v-if="store.getWeeklyResult !== undefined" class="flex flex-col gap-row-2 overflow-auto box-border px-2">
          <ComicCard
            v-for="comicInWeekly in store.getWeeklyResult.list"
            :key="comicInWeekly.id"
            :comic-id="comicInWeekly.id"
            :comic-title="comicInWeekly.name"
            :comic-author="comicInWeekly.author"
            :comic-category="comicInWeekly.category"
            :comic-category-sub="comicInWeekly.category_sub"
            :comic-downloaded="comicInWeekly.is_downloaded"
            :comic-download-dir="comicInWeekly.comic_download_dir" />
        </div>
      </n-tab-pane>
    </n-tabs>
  </div>
</template>
