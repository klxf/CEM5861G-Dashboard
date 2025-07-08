<template>
  <div class="min-h-screen bg-gray-100 p-4">
    <h1 class="text-3xl font-extrabold text-center text-gray-800 mb-6">CEM5861G 毫米波雷达上位机</h1>

    <div class="fixed top-4 left-4 z-40">
      <button
          @click="isConfigPanelOpen = true"
          class="px-6 py-3 bg-blue-500 text-white rounded-lg shadow-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-opacity-75 transition duration-150 ease-in-out"
      >
        <Cog6ToothIcon class="h-6 w-6 inline-block mr-2" aria-hidden="true" />
        配置面板
      </button>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 w-full">
      <SerialPortControl ref="serialControlRef"
                         @data-received="handleDataReceived"
                         @log-message="handleLogMessage"
                         @command-response="handleCommandResponse"
                         @parameter-response="handleParameterResponse"
                         @show-toast="showToast"
      />

      <DataDisplay :latest-data="latestReceivedData" />

      <LogDisplay :log-messages="allLogMessages" class="md:col-span-2 xl:col-span-1" />

      <div class="md:col-span-2 lg:col-span-3 w-full">
        <ChartComponent :chart-data="historyChartData" />
      </div>

      <SidebarDrawer
          :is-open="isConfigPanelOpen"
          title="雷达参数配置"
          @close="isConfigPanelOpen = false"
      >
        <ConfigPanel
            @send-serial-data="handleSendSerialData"
            :parameter-response="lastParameterResponse"
            @show-toast="showToast"
        />
      </SidebarDrawer>
    </div>

    <div class="grid grid-cols-1 gap-2 text-gray-600 mt-6 text-sm md:grid-cols-2">
      <span class="text-center mx-4 md:text-right">
        <CodeBracketIcon class="h-5 inline-block mr-2" />
        <a class="text-blue-500 focus:text-blue-700 transition duration-150 ease-in-out" href="https://github.com/klxf/CEM5861G-Dashboard">CEM5861G-Dashboard</a>
      </span>
      <span class="text-center mx-4 md:text-left">
        <ScaleIcon class="h-5 inline-block mr-2" />
        <a class="text-blue-500 focus:text-blue-700 transition duration-150 ease-in-out" href="https://choosealicense.com/licenses/apache-2.0/">Apache License 2.0</a>
      </span>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import Toastify from 'toastify-js';
import { Cog6ToothIcon, CodeBracketIcon, ScaleIcon } from '@heroicons/vue/24/outline';
import SerialPortControl from './components/SerialPortControl.vue';
import DataDisplay from './components/DataDisplay.vue';
import ChartComponent from './components/ChartComponent.vue';
import LogDisplay from "./components/LogDisplay.vue";
import ConfigPanel from "./components/ConfigPanel.vue";
import SidebarDrawer from "./components/SidebarDrawer.vue";

const latestReceivedData = ref(null);
const historyChartData = ref([]);
const allLogMessages = ref([]);
const serialControlRef = ref(null);
const lastParameterResponse = ref(null);

const isConfigPanelOpen = ref(false);

const MAX_CHART_DATA_POINTS = 100;
const MAX_LOG_MESSAGES = 200;

const handleDataReceived = (data) => {
  latestReceivedData.value = data;
  historyChartData.value.push(data);

  if (historyChartData.value.length > MAX_CHART_DATA_POINTS) {
    historyChartData.value.shift();
  }
};

const handleLogMessage = (message) => {
  allLogMessages.value.push(message);
  if (allLogMessages.value.length > MAX_LOG_MESSAGES) {
    allLogMessages.value.shift();
  }
};

const handleSendSerialData = (data) => {
  if (serialControlRef.value && serialControlRef.value.sendData) {
    serialControlRef.value.sendData(data);
  } else {
    handleLogMessage({ type: 'error', text: '串口控制器未准备好发送数据' });
  }
};

const handleCommandResponse = (response) => {
  // 处理命令响应，暂无具体实现
};

const handleParameterResponse = (response) => {
  lastParameterResponse.value = response;
};

const showToast = (type, message) => {
  let backgroundColor = '';
  switch (type) {
    case 'success':
      backgroundColor = 'linear-gradient(to right, #00b09b, #96c93d)';
      break;
    case 'error':
      backgroundColor = 'linear-gradient(to right, #ff5f6d, #ffc371)';
      break;
    case 'warning':
      backgroundColor = 'linear-gradient(to right, #ffcc00, #ff9900)';
      break;
    case 'info':
      backgroundColor = 'linear-gradient(to right, #2196F3, #2196F3)';
      break;
    default:
      backgroundColor = 'linear-gradient(to right, #007bff, #007bff)';
  }

  Toastify({
    text: message,
    duration: 3000,
    newWindow: true,
    close: true,
    gravity: 'top',
    position: 'right',
    stopOnFocus: true,
    style: {
      background: backgroundColor,
    },
  }).showToast();
};
</script>
