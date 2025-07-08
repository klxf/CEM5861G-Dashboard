<template>
  <div class="p-4 bg-white shadow rounded-lg">
    <h2 class="text-xl font-bold mb-4">
      <PresentationChartLineIcon class="w-6 inline-block" aria-hidden="true" />
      数据图表
    </h2>
    <div class="relative" style="height: 400px;">
      <canvas ref="chartCanvas"></canvas>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, onUnmounted } from 'vue';
import { Chart, registerables } from 'chart.js';
import { PresentationChartLineIcon } from "@heroicons/vue/24/outline";

Chart.register(...registerables);

const props = defineProps({
  chartData: Array,
});

const chartCanvas = ref(null);
let myChart = null;

const createChart = () => {
  if (myChart) {
    myChart.destroy();
  }

  const ctx = chartCanvas.value.getContext('2d');
  myChart = new Chart(ctx, {
    type: 'line',
    data: {
      labels: props.chartData.map(d => new Date(d.timestamp).toLocaleTimeString()),
      datasets: [
        {
          label: '距离 (cm)',
          data: props.chartData.map(d => d.distance),
          borderColor: 'rgb(75, 192, 192)',
          tension: 0.1,
          fill: false,
        },
        {
          label: '速度 (cm/s)',
          data: props.chartData.map(d => d.speed),
          borderColor: 'rgb(255, 99, 132)',
          tension: 0.1,
          fill: false,
          hidden: true, // 默认隐藏
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        x: {
          title: {
            display: true,
            text: '时间',
          },
        },
        y: {
          title: {
            display: true,
            text: '值',
          },
        },
      },
      plugins: {
        legend: {
          display: true,
        },
        tooltip: {
          mode: 'index',
          intersect: false,
        },
      },
    },
  });
};

onMounted(() => {
  createChart();
});

watch(() => props.chartData, () => {
  if (myChart) {
    myChart.data.labels = props.chartData.map(d => new Date(d.timestamp).toLocaleTimeString());
    myChart.data.datasets[0].data = props.chartData.map(d => d.distance);
    myChart.data.datasets[1].data = props.chartData.map(d => d.speed);
    myChart.update();
  } else {
    createChart();
  }
}, { deep: true });

onUnmounted(() => {
  if (myChart) {
    myChart.destroy();
  }
});
</script>
