<template>
  <div class="queue-button-group flex items-center">
    <Button
      class="comfyui-queue-button"
      :label="buttonLabel"
      severity="primary"
      size="small"
      :disabled="isExecuting || isOnQueue"
      @click="handleQueuePrompt"
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
    </Button>
    <!-- (Optional) Display your local queue count -->
  </div>
</template>

<script setup lang="ts">
import { storeToRefs } from 'pinia'
import Button from 'primevue/button'
import { computed, onUnmounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'

import { api } from '@/scripts/api'
import { useCommandStore } from '@/stores/commandStore'
import { useExecutionStore } from '@/stores/executionStore'
import { useQueueSettingsStore } from '@/stores/queueStore'
import { useWorkspaceStore } from '@/stores/workspaceStore'

const workspaceStore = useWorkspaceStore()
const { mode: queueMode } = storeToRefs(useQueueSettingsStore())
const { t } = useI18n()
const commandStore = useCommandStore()
const executionStore = useExecutionStore()

// Menu lookup (only "disabled" defined)
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
  const lookup = queueModeMenuItemLookup.value as Record<
    string,
    { key: string; label: string; tooltip: string; command: () => void }
  >
  return lookup[queueMode.value] || lookup.disabled
})
const queueModeMenuItems = computed(() =>
  Object.values(queueModeMenuItemLookup.value)
)

// local queue count (only your tasks)
const queueCount = ref(0)
// Flag that keeps your button disabled until your own task is finished
const isOnQueue = ref(false)
// Global execution flag (from the execution store)
const isExecuting = computed(() => !!executionStore.executingNodeId)

// Computed label:
// - "In esecuzione" if execution is active,
// - "In coda" if you have a queued task,
// - Else the default menu label.
const buttonLabel = computed(() => {
  if (isExecuting.value) {
    return 'In esecuzione'
  } else if (queueCount.value > 0) {
    return 'In coda'
  } else {
    return activeQueueModeMenuItem.value.label
  }
})

// Polling interval handle
let pollInterval: ReturnType<typeof setInterval> | null = null

// Helper: Count all tasks (non-local) in the API response without filtering by client_id.
function getGlobalCount(info: { Running?: any[]; Pending?: any[] }): number {
  const running = info.Running || []
  const pending = info.Pending || []
  return running.length + pending.length
}

// Poll your own queue status and only re-enable the button when your local count is zero and no execution is active.
const startPollingQueueStatus = () => {
  pollInterval = setInterval(async () => {
    try {
      const info = await api.getQueue()
      if (!isExecuting.value && isOnQueue.value) {
        isOnQueue.value = true
        queueCount.value = getGlobalCount(info) - 1
      } else {
        isOnQueue.value = false
        queueCount.value = 0
      }
    } catch (error) {
      console.error('Error fetching queue status:', error)
    }
  }, 1000)
}
startPollingQueueStatus()

// Execute the queue prompt command.
const queuePrompt = async () => {
  const commandId = 'Comfy.QueuePrompt'
  const result = await commandStore.execute(commandId)
  console.log(result)
  return result
}

// When the button is clicked:
// 1. Immediately disable the button and set queueCount to 1 so the label becomes "In coda".
// 2. Execute the queue prompt command.
// 3. Then start polling for updates.
async function handleQueuePrompt() {
  isOnQueue.value = true
  const error = await queuePrompt()
  if (error) {
    isOnQueue.value = false
  }
}

onUnmounted(() => {
  if (pollInterval) {
    clearInterval(pollInterval)
    pollInterval = null
  }
})
</script>

<style scoped>
.queue-count-badge {
  background-color: #ff4d4f;
  color: white;
  border-radius: 50%;
  padding: 0.2em 0.6em;
  font-size: 0.8rem;
  line-height: 1;
}
</style>
