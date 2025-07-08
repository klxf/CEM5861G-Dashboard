<template>
  <div class="p-4 bg-white shadow rounded-lg">
    <h2 class="text-xl font-bold mb-4">
      <ChatBubbleBottomCenterTextIcon class="w-6 inline-block" aria-hidden="true" />
      日志输出
    </h2>
    <div v-if="logMessages.length"
         ref="logContainer"
         class="mt-2 p-2 bg-gray-100 rounded-md max-h-36 overflow-y-auto text-sm">
      <p v-for="(msg, index) in logMessages" :key="index"
         :class="{ 'text-red-600': msg.type === 'error', 'text-orange-600': msg.type === 'warn', 'text-gray-800': msg.type === 'info' }">
        <span class="font-mono text-gray-500 mr-2">{{ msg.timestamp }}</span>
        <span class="font-semibold">{{ getLogLevelText(msg.type) }}:</span>
        {{ msg.text }}
      </p>
    </div>
    <div v-else class="text-gray-500">
      暂无日志信息...
    </div>
  </div>
</template>

<script setup>
import { defineProps, ref, watch, nextTick } from 'vue';
import { ChatBubbleBottomCenterTextIcon } from "@heroicons/vue/24/outline";

const props = defineProps({
  logMessages: {
    type: Array,
    default: () => [],
  },
});

const logContainer = ref(null);

const getLogLevelText = (type) => {
  switch (type) {
    case 'error':
      return '错误';
    case 'warn':
      return '警告';
    case 'info':
      return '信息';
    default:
      return '日志';
  }
};

watch(() => props.logMessages, async () => {
  await nextTick();
  if (logContainer.value) {
    logContainer.value.scrollTop = logContainer.value.scrollHeight;
  }
}, {deep: true});
</script>
