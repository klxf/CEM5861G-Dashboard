<template>
  <div class="p-4 bg-white shadow rounded-lg h-200">
    <h2 class="text-xl font-bold mb-4">
      <SwatchIcon class="w-6 inline-block" aria-hidden="true" />
      实时数据
    </h2>
    <div v-if="latestData" class="grid grid-cols-2 gap-2 text-gray-800">
      <p><strong>目标ID:</strong> {{ latestData.targetID }}</p>
      <p><strong>目标状态:</strong> {{ getStatusText(latestData.targetStatus) }}</p>
      <p><strong>距离:</strong> {{ latestData.distance }} cm</p>
      <p><strong>速度:</strong> {{ latestData.speed }} cm/s</p>
      <p><strong>方位角度:</strong> {{ latestData.azimuthAngle }} 度</p>
      <p><strong>俯仰角度:</strong> {{ latestData.elevationAngle }} 度</p>
      <p><strong>信号强度:</strong> {{ latestData.signalStrength }}</p>
      <p class="col-span-2 text-sm text-gray-500">更新时间: {{ new Date(latestData.timestamp).toLocaleTimeString() }}</p>
    </div>
    <div v-else class="text-gray-500">
      等待串口数据...
    </div>
  </div>
</template>

<script setup>
import { defineProps } from 'vue';
import { SwatchIcon } from "@heroicons/vue/24/outline";

const props = defineProps({
  latestData: Object,
});

const getStatusText = (status) => {
  switch (status) {
    case 0: return '无目标';
    case 1: return '移动';
    case 2: return '存在';
    default: return '未知';
  }
};
</script>
