<script setup lang="ts">
import type { ContestConfigQueryResult } from '@putongoj/shared'
import Button from 'primevue/button'
import DatePicker from 'primevue/datepicker'
import Dialog from 'primevue/dialog'
import IftaLabel from 'primevue/iftalabel'
import InputText from 'primevue/inputtext'
import { ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRouter } from 'vue-router'
import { createContest, getConfig, updateConfig } from '@/api/contest'
import { useMessage } from '@/utils/message'
import LabeledSwitch from './LabeledSwitch.vue'

const props = defineProps<{
  contestId: number
}>()
const visible = defineModel<boolean>('visible')

const { t } = useI18n()
const router = useRouter()
const message = useMessage()

const sourceConfig = ref<ContestConfigQueryResult | null>(null)
const submitting = ref(false)

const form = ref({
  title: '',
  startsAt: new Date(),
  endsAt: new Date(),
  isHidden: true,
  isPublic: false,
})

watch(visible, async (val) => {
  if (!val) return
  sourceConfig.value = null
  const resp = await getConfig(props.contestId)
  if (!resp.success) return
  const cfg = resp.data
  sourceConfig.value = cfg
  form.value = {
    title: t('ptoj.clone_of', { title: cfg.title }),
    startsAt: new Date(cfg.startsAt),
    endsAt: new Date(cfg.endsAt),
    isHidden: cfg.isHidden,
    isPublic: cfg.isPublic,
  }
})

async function onClone () {
  if (!sourceConfig.value) return
  submitting.value = true
  try {
    const createResp = await createContest({
      title: form.value.title,
      startsAt: form.value.startsAt,
      endsAt: form.value.endsAt,
      isHidden: form.value.isHidden,
      isPublic: form.value.isPublic,
      course: sourceConfig.value.course?.courseId ?? null,
    })
    if (!createResp.success) {
      message.error(t('ptoj.failed_clone_contest'), createResp.message)
      return
    }

    const newContestId = createResp.data.contestId
    const updateResp = await updateConfig(newContestId, {
      problems: sourceConfig.value.problems.map(p => p.problemId),
      allowedLanguages: sourceConfig.value.allowedLanguages,
      labelingStyle: sourceConfig.value.labelingStyle,
    })
    if (!updateResp.success) {
      message.error(t('ptoj.failed_clone_contest'), updateResp.message)
      return
    }

    message.success(t('ptoj.successful_clone_contest'), t('ptoj.successful_clone_contest_detail', { title: sourceConfig.value.title }))
    visible.value = false
    router.push({ name: 'contestEdit', params: { contestId: newContestId } })
  } finally {
    submitting.value = false
  }
}
</script>

<template>
  <Dialog
    v-model:visible="visible" modal :header="t('ptoj.clone_contest')" :closable="false"
    class="max-w-md mx-6 w-full"
  >
    <div v-if="!sourceConfig" class="flex h-32 items-center justify-center text-muted-color">
      <i class="mr-2 pi pi-spin pi-spinner" />
    </div>
    <form v-else @submit.prevent="onClone">
      <div class="space-y-4">
        <IftaLabel>
          <InputText id="title" v-model="form.title" required fluid />
          <label for="title">{{ t('ptoj.title') }}</label>
        </IftaLabel>
        <IftaLabel>
          <DatePicker
            v-model="form.startsAt" show-time show-seconds date-format="yy-mm-dd" time-format="HH:mm:ss"
            :step-second="15" fluid required
          />
          <label for="startsAt">{{ t('ptoj.starts_at') }}</label>
        </IftaLabel>
        <IftaLabel>
          <DatePicker
            v-model="form.endsAt" show-time show-seconds date-format="yy-mm-dd" time-format="HH:mm:ss"
            :step-second="15" fluid required
          />
          <label for="endsAt">{{ t('ptoj.ends_at') }}</label>
        </IftaLabel>
        <LabeledSwitch v-model="form.isHidden" :label="t('ptoj.hidden')" :description="t('ptoj.hide_from_listings')" />
        <LabeledSwitch v-model="form.isPublic" :label="t('ptoj.public')" :description="t('ptoj.anyone_can_join')" />
        <div class="text-sm text-muted-color">
          {{ t('ptoj.course') }}: {{ sourceConfig.course?.name ?? '—' }}
        </div>
      </div>
      <div class="flex gap-2 justify-end mt-5">
        <Button
          type="button" :label="t('ptoj.cancel')" icon="pi pi-times" severity="secondary" outlined
          :disabled="submitting" @click="visible = false"
        />
        <Button type="submit" :label="t('ptoj.clone_contest')" icon="pi pi-copy" :loading="submitting" />
      </div>
    </form>
  </Dialog>
</template>
