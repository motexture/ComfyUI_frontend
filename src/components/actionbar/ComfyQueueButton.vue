<template>
  <div class="queue-button-group flex">
    <SplitButton
      class="comfyui-queue-button"
      :label="activeQueueModeMenuItem.label"
      severity="primary"
      size="small"
      @click="queuePrompt"
      :model="queueModeMenuItems"
      data-testid="queue-button"
      v-tooltip.bottom="{
        value: workspaceStore.shiftDown
          ? $t('menu.runWorkflow')
          : $t('menu.runWorkflow'),
        showDelay: 600
      }"
    >
      <template #icon>
        <i-lucide:play v-if="queueMode === 'disabled'" />
      </template>
      <template #item="{ item }">
        <Button
          :label="String(item.label)"
          :icon="item.icon"
          :severity="item.key === queueMode ? 'primary' : 'secondary'"
          size="small"
          text
          v-tooltip="{
            value: item.tooltip,
            showDelay: 600
          }"
        />
      </template>
    </SplitButton>
  </div>
</template>

<script setup lang="ts">
import { storeToRefs } from 'pinia'
import Button from 'primevue/button'
import SplitButton from 'primevue/splitbutton'
import { computed } from 'vue'
import { useI18n } from 'vue-i18n'

import { useCommandStore } from '@/stores/commandStore'
import { useQueueSettingsStore } from '@/stores/queueStore'
import { useWorkspaceStore } from '@/stores/workspaceStore'

const workspaceStore = useWorkspaceStore()
const { mode: queueMode } = storeToRefs(useQueueSettingsStore())

const { t } = useI18n()

// Only the 'disabled' key is defined.
const queueModeMenuItemLookup = computed(() => ({
  disabled: {
    key: 'disabled',
    label: t('menu.run'),
    tooltip: t('menu.disabledTooltip'),
    command: () => {
      queueMode.value = 'disabled'
    }
  }
}))

const activeQueueModeMenuItem = computed(() => {
  // Assert that lookup can be indexed with any string.
  const lookup = queueModeMenuItemLookup.value as Record<
    string,
    { key: string; label: string; tooltip: string; command: () => void }
  >
  // If the current queueMode key isn't found, fall back to 'disabled'.
  return lookup[queueMode.value] || lookup.disabled
})

const queueModeMenuItems = computed(() =>
  Object.values(queueModeMenuItemLookup.value)
)

// The following computed properties are not used in the template.
// Remove or uncomment them if needed.
// const executingPrompt = computed(() => !!queueCountStore.count.value)
// const hasPendingTasks = computed(
//   () => queueCountStore.count.value > 1 || queueMode.value !== 'disabled'
// )

const commandStore = useCommandStore()
const queuePrompt = (e: Event) => {
  const commandId =
    'shiftKey' in e && e.shiftKey ? 'Comfy.QueuePrompt' : 'Comfy.QueuePrompt'
  commandStore.execute(commandId)
}
</script>

<style scoped>
.comfyui-queue-button :deep(.p-splitbutton-dropdown) {
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
}
</style>
