<template>
  <div class="queue-button-group flex">
    <Button
      class="comfyui-queue-button"
      :label="activeQueueModeMenuItem.label"
      severity="primary"
      size="small"
      :disabled="isLocallyProcessing"
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
  </div>
</template>

<script setup lang="ts">
import { storeToRefs } from 'pinia'
import Button from 'primevue/button'
import { computed, onUnmounted, ref } from 'vue'
import { useI18n } from 'vue-i18n'

import { api } from '@/scripts/api'
import { useCommandStore } from '@/stores/commandStore'
import { useQueueSettingsStore } from '@/stores/queueStore'
import { useWorkspaceStore } from '@/stores/workspaceStore'

// adjust path as needed

// Setup stores and i18n
const workspaceStore = useWorkspaceStore()
const { mode: queueMode } = storeToRefs(useQueueSettingsStore())
const { t } = useI18n()
const commandStore = useCommandStore()

// Define menu item lookup (only the 'disabled' key is defined here)
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

// Local run state: true only after the button is clicked on this client
const isLocallyProcessing = ref(false)

// Polling interval handle; we'll start polling only when the client has initiated a run
let pollInterval: ReturnType<typeof setInterval> | null = null

// Function to start polling the server's queue status
const startPollingQueueStatus = () => {
  // Adjust the polling interval as needed (currently set to poll every 1 second)
  pollInterval = setInterval(async () => {
    try {
      const info = await api.getQueue()
      // Check if both Running and Pending queues are empty
      if (
        (!info.Running || info.Running.length === 0) &&
        (!info.Pending || info.Pending.length === 0)
      ) {
        isLocallyProcessing.value = false
        if (pollInterval) {
          clearInterval(pollInterval)
          pollInterval = null
        }
      }
    } catch (error) {
      console.error('Error fetching queue status:', error)
    }
  }, 1000)
}

// Function to execute the queue prompt command (i.e. add the job)
const queuePrompt = async () => {
  const commandId = 'Comfy.QueuePrompt'
  return commandStore.execute(commandId)
}

// Handle button click: set local flag, add job, then start polling server state
async function handleQueuePrompt() {
  // Prevent multiple clicks if already processing locally
  if (isLocallyProcessing.value) return
  isLocallyProcessing.value = true

  try {
    // Execute the command to add to the queue
    await queuePrompt()
    // Start polling the server for pending/running jobs
    startPollingQueueStatus()
  } catch (error) {
    console.error('Error executing queue prompt:', error)
    // Reset local state and clear polling if there is an error
    isLocallyProcessing.value = false
    if (pollInterval) {
      clearInterval(pollInterval)
      pollInterval = null
    }
  }
}

// Ensure polling is stopped if the component unmounts
onUnmounted(() => {
  if (pollInterval) {
    clearInterval(pollInterval)
    pollInterval = null
  }
})
</script>

<style scoped>
.comfyui-queue-button :deep(.p-splitbutton-dropdown) {
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
}
</style>
