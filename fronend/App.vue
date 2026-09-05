<template>
  <div class="container">
    <div class="toolbar">
      <button @click="togglePlay">{{ isPlaying ? '⏸️ 暫停' : '▶️ 播放' }}</button>
      <button @click="enableDrawFib">📏 手動拉斐波那契</button>
    </div>
    <div ref="chartContainer" class="chart"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { createChart } from 'lightweight-charts';
import axios from 'axios';

const chartContainer = ref(null);
let chart, candleSeries;
const allData = ref([]);
const currentIndex = ref(50);
const isPlaying = ref(false);
let isDrawing = false, points = [], timer = null;

onMounted(async () => {
  chart = createChart(chartContainer.value, { width: 800, height: 500 });
  candleSeries = chart.addCandlestickSeries();

  const res = await axios.get('http://localhost:8000/api/klines');
  allData.value = res.data;
  candleSeries.setData(allData.value.slice(0, currentIndex.value));

  // 監聽滑鼠點擊進行手動繪圖
  chart.subscribeClick((param) => {
    if (!isDrawing || !param.point) return;
    const price = candleSeries.coordinateToPrice(param.point.y);
    if (price) points.push(price);

    if (points.length === 2) {
      drawFib(points[0], points[1]);
      isDrawing = false;
      points = [];
    }
  });
});

const togglePlay = () => {
  isPlaying.value = !isPlaying.value;
  if (isPlaying.value) {
    timer = setInterval(() => {
      if (currentIndex.value < allData.value.length) {
        currentIndex.value++;
        candleSeries.setData(allData.value.slice(0, currentIndex.value));
      } else clearInterval(timer);
    }, 500);
  } else clearInterval(timer);
};

const enableDrawFib = () => { isDrawing = true; points = []; };

const drawFib = (p1, p2) => {
  const high = Math.max(p1, p2), low = Math.min(p1, p2), diff = high - low;
  [0, 0.236, 0.382, 0.5, 0.618, 0.786, 1].forEach(ratio => {
    candleSeries.createPriceLine({
      price: high - diff * ratio,
      color: ratio === 0.618 ? '#ffd700' : '#888',
      title: `Fib ${ratio}`,
    });
  });
};
</script>