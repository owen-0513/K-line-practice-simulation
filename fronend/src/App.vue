<template>
  <div class="app-container" tabindex="0" @keydown="handleKeyDown">
    <div class="toolbar">
      <!-- 幣種與週期選擇 -->
      <select v-model="currentSymbol" @change="changeSymbol" class="symbol-select">
        <option value="BTCUSDT">BTC/USDT</option>
        <option value="ETHUSDT">ETH/USDT</option>
      </select>

      <select v-model="currentInterval" @change="changeInterval" class="symbol-select">
        <option value="5m">5 分鐘</option>
        <option value="15m">15 分鐘</option>
        <option value="30m">30 分鐘</option>
        <option value="1h">1 小時</option>
        <option value="4h">4 小時</option>
        <option value="1d">1 天</option>
      </select>

      <div class="divider"></div>

      <!-- 重播控制 -->
      <button @click="toggleReplaySelectMode" :class="{ active: isReplaySelectMode }">
        ✂️ {{ isReplaySelectMode ? '點擊圖表剪裁起點...' : '選擇重播起點' }}
      </button>
      <button @click="togglePlay" :disabled="isReplaySelectMode">
        {{ isPlaying ? '⏸️ 暫停' : '▶️ 播放' }}
      </button>
      <button @click="nextFrame" :disabled="isReplaySelectMode">⏭️ 下一支</button>
      <button @click="resetReplay" :disabled="isReplaySelectMode">🔄 最新行情</button>

      <div class="divider"></div>

      <!-- 繪圖工具區 -->
      <button @click="toggleDrawMode('trend')" :class="{ active: drawMode === 'trend' }">
        ✏️ 畫趨勢線
      </button>
      <button @click="toggleDrawMode('fib')" :class="{ active: drawMode === 'fib' }">
        📏 斐波那契
      </button>
      <button @click="toggleDrawMode('vp')" :class="{ active: drawMode === 'vp' }">
        📊 平均成交量 (VP)
      </button>
      <button @click="toggleDrawMode('pen')" :class="{ active: drawMode === 'pen' }">
        ✒️ 筆
      </button>
      <button @click="toggleDrawMode('rect')" :class="{ active: drawMode === 'rect' }">
        ⬜ 方框
      </button>

      <div class="divider"></div>

      <!-- 清除與刪除 -->
      <button @click="deleteSelected" :disabled="!selectedDrawingId" class="btn-delete-selected">
        🗑️ 刪除選取 (Del)
      </button>
      <button @click="clearAllDrawings" class="btn-clear-all">
        ❌ 全部清除
      </button>
    </div>

    <!-- 模擬下單與測驗控制面板 -->
    <div class="trading-panel">
      <div class="account-info">
        <span>💰 餘額: <b>${{ balance.toFixed(2) }} USDT</b></span>
        <span :class="unrealizedPnL >= 0 ? 'text-green' : 'text-red'">
          📈 未平倉損益: <b>{{ unrealizedPnL >= 0 ? '+' : '' }}{{ unrealizedPnL.toFixed(2) }} USDT ({{ unrealizedPnLPercent.toFixed(2) }}%)</b>
        </span>
        <span v-if="position">
          🏷️ 持倉: <b :class="position.type === 'LONG' ? 'text-green' : 'text-red'">{{ position.type }}</b> 
          @ {{ position.entryPrice }} (TP: {{ position.takeProfitPrice || '無' }} | SL: {{ position.stopLossPrice || '無' }})
        </span>
        <span v-else>🏷️ 持倉: <b>無</b></span>
      </div>

      <div class="trading-actions">
        <!-- 自訂投入金額輸入框 -->
        <div class="input-group">
          <label>投入 ${{ inputMargin }}</label>
          <input 
            type="number" 
            v-model.number="inputMargin" 
            min="10" 
            step="100" 
            class="margin-input" 
            :disabled="position"
          />
        </div>

        <!-- 止盈價格輸入框 -->
        <div class="input-group">
          <label>止盈價</label>
          <input 
            type="number" 
            v-model.number="inputTakeProfit" 
            step="any" 
            class="margin-input" 
            placeholder="選填"
            :disabled="position"
          />
        </div>

        <!-- 停損價格輸入框 -->
        <div class="input-group">
          <label>停損價</label>
          <input 
            type="number" 
            v-model.number="inputStopLoss" 
            step="any" 
            class="margin-input" 
            placeholder="選填"
            :disabled="position"
          />
        </div>

        <!-- 槓桿選擇 -->
        <select v-model="selectedLeverage" class="symbol-select leverage-select" :disabled="position">
          <option :value="1">1x</option>
          <option :value="5">5x</option>
          <option :value="10">10x</option>
          <option :value="25">25x</option>
          <option :value="50">50x</option>
          <option :value="100">100x</option>
          <option :value="150">150x</option>
        </select>

        <button @click="openPosition('LONG')" :disabled="position || isReplaySelectMode" class="btn-long">🟢 做多</button>
        <button @click="openPosition('SHORT')" :disabled="position || isReplaySelectMode" class="btn-short">🔴 做空</button>
        <button @click="closePosition" :disabled="!position" class="btn-close">🔒 平倉</button>
        <div class="divider"></div>
        <button @click="startQuizMode" class="btn-quiz">🎯 隨機出題測驗</button>
        
        <!-- 測驗勝率統計顯示 -->
        <span class="quiz-stats" v-if="quizStats.total > 0">
          📊 勝率: <b>{{ winRate }}%</b> (勝:{{ quizStats.wins }} / 敗:{{ quizStats.losses }} / 總:{{ quizStats.total }})
        </span>
        <span class="quiz-hint" v-if="isQuizMode">🔥 測驗中：請下單後點擊「⏭️ 下一支」觀察結果再平倉！</span>
      </div>
    </div>

    <!-- 圖表與 Canvas 疊加層 -->
    <div 
      ref="chartWrapper" 
      class="chart-wrapper"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
    >
      <div ref="chartContainer" class="chart-container"></div>
      <canvas ref="overlayCanvas" class="overlay-canvas"></canvas>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import { createChart, CandlestickSeries, CrosshairMode } from 'lightweight-charts';
import axios from 'axios';

const chartContainer = ref(null);
const chartWrapper = ref(null);
const overlayCanvas = ref(null);

let chart = null;
let candleSeries = null;
let resizeObserver = null;
let isUnmounted = false; 

const allData = ref([]);
const currentIndex = ref(0);
const isPlaying = ref(false);
const isReplaySelectMode = ref(false);
const currentSymbol = ref('BTCUSDT');
const currentInterval = ref('1h');
const isLoading = ref(false);

// 模擬下單狀態、保證金、槓桿、止盈與停損
const balance = ref(10000);
const inputMargin = ref(1000); 
const inputTakeProfit = ref(null);
const inputStopLoss = ref(null);
const selectedLeverage = ref(10);
const position = ref(null);

// 測驗與勝率統計狀態
const isQuizMode = ref(false);
const quizStats = ref({
  total: 0,
  wins: 0,
  losses: 0
});

// 繪圖模式與物件
const drawMode = ref('none');
let isDragging = false;
let dragStart = null;
const drawings = ref([]);
const selectedDrawingId = ref(null);
let tempDrawing = null;
let timer = null;
let rafId = null;

// 計算即時價格與損益
const currentPrice = computed(() => {
  if (!allData.value.length || currentIndex.value === 0) return 0;
  return allData.value[currentIndex.value - 1].close;
});

const unrealizedPnL = computed(() => {
  if (!position.value) return 0;
  const priceDiff = position.value.type === 'LONG' 
    ? currentPrice.value - position.value.entryPrice 
    : position.value.entryPrice - currentPrice.value;
  
  const returnRate = priceDiff / position.value.entryPrice;
  return returnRate * position.value.margin * position.value.leverage;
});

const unrealizedPnLPercent = computed(() => {
  if (!position.value) return 0;
  const priceDiff = position.value.type === 'LONG' 
    ? (currentPrice.value - position.value.entryPrice) / position.value.entryPrice 
    : (position.value.entryPrice - currentPrice.value) / position.value.entryPrice;
  return priceDiff * 100 * position.value.leverage;
});

const winRate = computed(() => {
  if (quizStats.value.total === 0) return 0;
  return ((quizStats.value.wins / quizStats.value.total) * 100).toFixed(1);
});

onMounted(async () => {
  if (!chartContainer.value) return;

  chart = createChart(chartContainer.value, {
    width: chartWrapper.value.clientWidth || 800,
    height: chartWrapper.value.clientHeight || 600,
    layout: { background: { color: '#161a25' }, textColor: '#d1d4dc' },
    grid: { vertLines: { color: '#2B2B43' }, horzLines: { color: '#2B2B43' } },
    timeScale: { borderColor: '#2B2B43', timeVisible: true, rightOffset: 12 },
    // 設定十字線為 Normal 模式，讓它跟隨滑鼠自由移動而不吸附 K 棒
    crosshair: {
      mode: CrosshairMode.Normal,
      vertLine: {
        color: '#758696',
        width: 1,
        style: 3,
        labelBackgroundColor: '#2962ff',
      },
      horzLine: {
        color: '#758696',
        width: 1,
        style: 3,
        labelBackgroundColor: '#2962ff',
      },
    },
  });

  candleSeries = chart.addSeries(CandlestickSeries, {
    upColor: '#089981', downColor: '#f23645',
    borderVisible: false, wickUpColor: '#089981', wickDownColor: '#f23645',
  });

  setupCanvas();
  
  resizeObserver = new ResizeObserver(() => handleResize());
  if (chartWrapper.value) resizeObserver.observe(chartWrapper.value);

  chart.timeScale().subscribeVisibleLogicalRangeChange(() => redrawCanvas());

  await fetchKlines();

  chart.subscribeClick((param) => {
    if (isReplaySelectMode.value && param.time) {
      const clickedIndex = allData.value.findIndex(d => d.time === param.time);
      if (clickedIndex !== -1) {
        currentIndex.value = clickedIndex + 1;
        renderChart();
        isReplaySelectMode.value = false;
      }
      return;
    }

    if (param.point && !position.value) {
      const clickedPrice = candleSeries.coordinateToPrice(param.point.y);
      if (clickedPrice) {
        const formattedPrice = parseFloat(clickedPrice.toFixed(2));
        if (!inputTakeProfit.value) {
          inputTakeProfit.value = formattedPrice;
        } else if (!inputStopLoss.value) {
          inputStopLoss.value = formattedPrice;
        }
      }
    }
  });
});

const fetchKlines = async (targetStartTime = null, targetEndTime = null) => {
  try {
    let url = `https://api.binance.com/api/v3/klines?symbol=${currentSymbol.value}&interval=${currentInterval.value}&limit=1000`;
    if (targetStartTime && targetEndTime) {
      url = `https://api.binance.com/api/v3/klines?symbol=${currentSymbol.value}&interval=${currentInterval.value}&startTime=${targetStartTime}&endTime=${targetEndTime}&limit=1000`;
    }
    const res = await axios.get(url);
    if (isUnmounted) return;
    if (res.data && res.data.length > 0) {
      allData.value = res.data.map(item => ({
        time: item[0] / 1000,
        open: parseFloat(item[1]),
        high: parseFloat(item[2]),
        low: parseFloat(item[3]),
        close: parseFloat(item[4]),
        volume: parseFloat(item[5]),
      }));
      currentIndex.value = allData.value.length;
      renderChart();
      if (chart && !targetStartTime) chart.timeScale().fitContent();
    }
  } catch (error) {
    console.error('API 讀取失敗', error);
  }
};

const changeSymbol = async () => {
  if (isPlaying.value) togglePlay();
  await fetchKlines();
};

const changeInterval = async () => {
  if (isPlaying.value) togglePlay();

  // 1. 如果目前在「測驗模式」，保留當前測驗的時間邊界；否則取得目前畫面的目標時間點
  let targetTime = null;
  let quizStartTime = null;
  let quizEndTime = null;

  if (isQuizMode.value && allData.value.length > 0) {
    const currentCandle = allData.value[currentIndex.value - 1] || allData.value[0];
    targetTime = currentCandle.time;
    quizStartTime = Math.floor(allData.value[0].time * 1000);
    quizEndTime = Math.floor(allData.value[allData.value.length - 1].time * 1000);
  } else {
    const currentCandle = allData.value[currentIndex.value - 1];
    targetTime = currentCandle ? currentCandle.time : null;
  }

  let minTime = targetTime ? Math.floor(targetTime * 1000) : null;
  let maxTime = minTime;

  // 收集畫線的時間範圍（僅一般模式）
  if (!isQuizMode.value) {
    drawings.value.forEach(item => {
      let t1 = null, t2 = null;
      if (item.type === 'trend' || item.type === 'rect') {
        t1 = item.time1;
        t2 = item.time2;
      } else if (item.type === 'fib') {
        t1 = item.time1;
        t2 = item.time2;
      } else if (item.type === 'vp') {
        t1 = item.startTime;
        t2 = item.endTime;
      } else if (item.type === 'pen' && item.points) {
        item.points.forEach(p => {
          if (p.time) {
            if (!t1 || p.time < t1) t1 = p.time;
            if (!t2 || p.time > t2) t2 = p.time;
          }
        });
      }

      if (t1) {
        const ms1 = Math.floor(t1 * 1000);
        if (!minTime || ms1 < minTime) minTime = ms1;
        if (!maxTime || ms1 > maxTime) maxTime = ms1;
      }
      if (t2) {
        const ms2 = Math.floor(t2 * 1000);
        if (!minTime || ms2 < minTime) minTime = ms2;
        if (!maxTime || ms2 > maxTime) maxTime = ms2;
      }
    });
  }

  try {
    const intervalMs = getIntervalMs(currentInterval.value);
    let url = `https://api.binance.com/api/v3/klines?symbol=${currentSymbol.value}&interval=${currentInterval.value}&limit=1000`;

    if (isQuizMode.value && quizStartTime && quizEndTime) {
      // 2. 測驗模式：維持原本測驗的區間去要新週期的 K 棒
      const startTime = Math.max(0, quizStartTime);
      const endTime = quizEndTime + (200 * intervalMs);
      url = `https://api.binance.com/api/v3/klines?symbol=${currentSymbol.value}&interval=${currentInterval.value}&startTime=${startTime}&endTime=${endTime}&limit=1000`;
    } else if (maxTime) {
      // 3. 一般模式：以結束點往前推 1000 根，確保有足夠的 K 棒數量
      const endTime = maxTime;
      const startTime = Math.max(0, endTime - (1000 * intervalMs));
      url = `https://api.binance.com/api/v3/klines?symbol=${currentSymbol.value}&interval=${currentInterval.value}&startTime=${startTime}&endTime=${endTime}&limit=1000`;
    }

    const res = await axios.get(url);
    if (isUnmounted) return;
    if (!res.data || res.data.length === 0) return;

    const newData = res.data.map(item => ({
      time: item[0] / 1000,
      open: Number(item[1]),
      high: Number(item[2]),
      low: Number(item[3]),
      close: Number(item[4]),
      volume: Number(item[5]),
    }));

    allData.value = newData;

    if (targetTime) {
      let closestIndex = 0;
      let minDiff = Infinity;
      newData.forEach((item, index) => {
        const diff = Math.abs(item.time - targetTime);
        if (diff < minDiff) {
          minDiff = diff;
          closestIndex = index;
        }
      });
      currentIndex.value = Math.min(newData.length, Math.max(1, closestIndex + 1));
    } else {
      currentIndex.value = newData.length;
    }

    renderChart();

    if (chart && !maxTime && !isQuizMode.value) {
      chart.timeScale().fitContent();
    }
  } catch (error) {
    console.error('切換週期失敗:', error);
  }
};

const getIntervalMs = (interval) => {
  const map = {
    '1m': 60 * 1000,
    '5m': 5 * 60 * 1000,
    '15m': 15 * 60 * 1000,
    '30m': 30 * 60 * 1000,
    '1h': 60 * 60 * 1000,
    '4h': 4 * 60 * 60 * 1000,
    '1d': 24 * 60 * 60 * 1000,
  };
  return map[interval] || 60 * 60 * 1000;
};

const checkRiskTriggers = () => {
  if (!position.value) return;
  const currentCandle = allData.value[currentIndex.value - 1];
  if (!currentCandle) return;

  // 計算當前未實現損益並檢查是否爆倉
  const priceDiff = position.value.type === 'LONG' 
    ? currentPrice.value - position.value.entryPrice 
    : position.value.entryPrice - currentPrice.value;
  const currentPnL = (priceDiff / position.value.entryPrice) * position.value.margin * position.value.leverage;

  if (balance.value + currentPnL <= 0) {
    balance.value = 0;
    if (isQuizMode.value) {
      quizStats.value.total++;
      quizStats.value.losses++;
      isQuizMode.value = false;
    }
    alert('💥 靠北喔傻屌 玩到爆倉你他媽要死是嗎 請倉位管理');
    position.value = null;
    if (isPlaying.value) togglePlay();
    return;
  }

  let triggeredType = null;
  let triggerPrice = 0;

  if (position.value.type === 'LONG') {
    if (position.value.takeProfitPrice && currentCandle.high >= position.value.takeProfitPrice) {
      triggeredType = 'TP';
      triggerPrice = position.value.takeProfitPrice;
    } else if (position.value.stopLossPrice && currentCandle.low <= position.value.stopLossPrice) {
      triggeredType = 'SL';
      triggerPrice = position.value.stopLossPrice;
    }
  } else if (position.value.type === 'SHORT') {
    if (position.value.takeProfitPrice && currentCandle.low <= position.value.takeProfitPrice) {
      triggeredType = 'TP';
      triggerPrice = position.value.takeProfitPrice;
    } else if (position.value.stopLossPrice && currentCandle.high >= position.value.stopLossPrice) {
      triggeredType = 'SL';
      triggerPrice = position.value.stopLossPrice;
    }
  }

  if (triggeredType) {
    const pnl = position.value.type === 'LONG'
      ? (triggerPrice - position.value.entryPrice) / position.value.entryPrice * position.value.margin * position.value.leverage
      : (position.value.entryPrice - triggerPrice) / position.value.entryPrice * position.value.margin * position.value.leverage;

    balance.value += pnl;

    if (isQuizMode.value) {
      quizStats.value.total++;
      if (triggeredType === 'TP') {
        quizStats.value.wins++;
      } else {
        quizStats.value.losses++;
      }
      isQuizMode.value = false;
    }

    const msg = triggeredType === 'TP' ? '🎯 觸發止盈！' : '⚠️ 觸發停損！';
    alert(`${msg} 已自動平倉，損益: ${pnl.toFixed(2)} USDT`);
    position.value = null;
    if (isPlaying.value) togglePlay();
  }
};

const openPosition = (type) => {
  if (position.value || currentIndex.value === 0) return;

  if (balance.value <= 0) {
    alert('白癡 沒錢了拉！案F5刷新畫面重新給你錢');
    return;
  }
  
  const marginToUse = Number(inputMargin.value);
  if (!marginToUse || marginToUse <= 0) {
    alert('請輸入有效的投入金額！');
    return;
  }
  if (balance.value < marginToUse) {
    alert('你沒那麼多錢！');
    return;
  }

  const currentCandle = allData.value[currentIndex.value - 1];
  const entry = currentCandle.close;

  position.value = { 
    type, 
    entryPrice: entry, 
    entryTime: currentCandle.time,
    margin: marginToUse,
    leverage: selectedLeverage.value,
    takeProfitPrice: inputTakeProfit.value ? Number(inputTakeProfit.value) : null,
    stopLossPrice: inputStopLoss.value ? Number(inputStopLoss.value) : null
  };
  redrawCanvas();
};

const closePosition = () => {
  if (!position.value) return;
  const pnl = unrealizedPnL.value;
  balance.value += pnl;

  if (isQuizMode.value) {
    quizStats.value.total++;
    if (pnl > 0) {
      quizStats.value.wins++;
    } else {
      quizStats.value.losses++;
    }
    isQuizMode.value = false;
  }

  position.value = null;
  redrawCanvas();
};

const startQuizMode = async () => {
  if (isPlaying.value) togglePlay();
  clearAllDrawings();
  if (position.value) closePosition();
  isLoading.value = true;

  try {
    const symbols = ['BTCUSDT', 'ETHUSDT'];
    currentSymbol.value = symbols[Math.floor(Math.random() * symbols.length)];

    const intervals = ['15m', '1h', '4h'];
    currentInterval.value = intervals[Math.floor(Math.random() * intervals.length)];

    const now = Date.now();
    const startYear2023 = new Date('2023-01-01').getTime();
    
    const maxRandomEnd = now - (30 * 24 * 60 * 60 * 1000);
    const randomEndTime = startYear2023 + Math.random() * (maxRandomEnd - startYear2023);
    
    const intervalMs = getIntervalMs(currentInterval.value);
    const randomStartTime = randomEndTime - (1000 * intervalMs);

    let url = `https://api.binance.com/api/v3/klines?symbol=${currentSymbol.value}&interval=${currentInterval.value}&startTime=${Math.floor(randomStartTime)}&endTime=${Math.floor(randomEndTime)}&limit=1000`;
    
    const res = await axios.get(url);
    
    if (!res.data || res.data.length < 200) {
      isLoading.value = false;
      return startQuizMode();
    }

    allData.value = res.data.map(item => ({
      time: item[0] / 1000,
      open: parseFloat(item[1]),
      high: parseFloat(item[2]),
      low: parseFloat(item[3]),
      close: parseFloat(item[4]),
      volume: parseFloat(item[5]),
    }));

    const minIdx = 100;
    const maxIdx = allData.value.length - 30;
    currentIndex.value = Math.floor(Math.random() * (maxIdx - minIdx + 1)) + minIdx;

    isQuizMode.value = true;
    renderChart();
    if (chart) chart.timeScale().fitContent();

  } catch (error) {
    console.error('測驗模式載入歷史資料失敗:', error);
  } finally {
    isLoading.value = false;
  }
};

const setupCanvas = () => {
  if (!overlayCanvas.value || !chartWrapper.value) return;
  const width = chartWrapper.value.clientWidth;
  const height = chartWrapper.value.clientHeight;
  if (overlayCanvas.value.width !== width || overlayCanvas.value.height !== height) {
    overlayCanvas.value.width = width;
    overlayCanvas.value.height = height;
  }
};

const handleResize = () => {
  if (chart && chartWrapper.value) {
    chart.applyOptions({
      width: chartWrapper.value.clientWidth,
      height: chartWrapper.value.clientHeight,
    });
    setupCanvas();
    redrawCanvas();
  }
};

const renderChart = () => {
  if (!allData.value.length || !candleSeries) return;

  const logicalRange = chart ? chart.timeScale().getVisibleLogicalRange() : null;

  candleSeries.setData(allData.value.slice(0, currentIndex.value));

  if (chart && logicalRange) {
    chart.timeScale().setVisibleLogicalRange(logicalRange);
  }

  checkRiskTriggers();
  redrawCanvas();
};

const toggleDrawMode = (mode) => {
  drawMode.value = drawMode.value === mode ? 'none' : mode;
  isReplaySelectMode.value = false;
  if (chart) {
    chart.applyOptions({
      handleScroll: drawMode.value === 'none',
      handleScale: drawMode.value === 'none',
    });
  }
};

const getTimeFromCoordinate = (x) => {
  let time = chart.timeScale().coordinateToTime(x);
  if (!time && allData.value.length > 1) {
    const visibleData = allData.value.slice(0, currentIndex.value);
    if (visibleData.length > 1) {
      const lastItem = visibleData[visibleData.length - 1];
      const firstItem = visibleData[0];
      const timeStep = (lastItem.time - firstItem.time) / (visibleData.length - 1);
      const logical = chart.timeScale().coordinateToLogical(x);
      time = lastItem.time + (logical - (visibleData.length - 1)) * timeStep;
    } else {
      time = allData.value[0].time;
    }
  }
  return time;
};

const handleMouseDown = (e) => {
  if (!chartWrapper.value || !chart || !candleSeries) return;
  const rect = chartWrapper.value.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  if (drawMode.value === 'none') {
    checkSelection(x, y);
    return;
  }

  isDragging = true;

  if (drawMode.value === 'pen') {
    dragStart = { x, y };
    tempDrawing = {
      type: 'pen',
      points: [{ x, y }]
    };
  } else {
    const price = candleSeries.coordinateToPrice(y);
    let time = getTimeFromCoordinate(x);
    dragStart = { x, y, price, time };

    if (drawMode.value === 'trend') {
      if (price && time) {
        tempDrawing = {
          type: 'trend',
          time1: time, price1: price,
          time2: time, price2: price,
        };
      }
    } else if (drawMode.value === 'fib') {
      if (price && time) {
        tempDrawing = {
          type: 'fib',
          price1: price, price2: price,
          time1: time, time2: time
        };
      }
    } else if (drawMode.value === 'rect') {
      if (price && time) {
        tempDrawing = {
          type: 'rect',
          time1: time, price1: price,
          time2: time, price2: price
        };
      }
    }
  }
};

const handleMouseMove = (e) => {
  if (!isDragging || !chartWrapper.value || !chart || !candleSeries) return;
  
  if (rafId) return;
  rafId = requestAnimationFrame(() => {
    rafId = null;
    const rect = chartWrapper.value.getBoundingClientRect();
    const currentX = e.clientX - rect.left;
    const currentY = e.clientY - rect.top;

    if (drawMode.value === 'pen' && tempDrawing) {
      tempDrawing.points.push({ x: currentX, y: currentY });
      redrawCanvas();
      return;
    }

    if (!dragStart) return;
    const currentPriceVal = candleSeries.coordinateToPrice(currentY);
    let currentTime = getTimeFromCoordinate(currentX);

    if (drawMode.value === 'trend') {
      if (currentPriceVal && currentTime) {
        tempDrawing = {
          type: 'trend',
          time1: dragStart.time, price1: dragStart.price,
          time2: currentTime, price2: currentPriceVal,
        };
      }
    } else if (drawMode.value === 'fib') {
      if (currentPriceVal && currentTime) {
        tempDrawing = {
          type: 'fib',
          price1: dragStart.price,
          price2: currentPriceVal,
          time1: dragStart.time,
          time2: currentTime
        };
      }
    } else if (drawMode.value === 'vp') {
      if (currentTime) {
        tempDrawing = buildVPItem(dragStart.time, currentTime);
      }
    } else if (drawMode.value === 'rect') {
      if (currentPriceVal && currentTime) {
        tempDrawing = {
          type: 'rect',
          time1: dragStart.time, price1: dragStart.price,
          time2: currentTime, price2: currentPriceVal
        };
      }
    }
    redrawCanvas();
  });
};

const handleMouseUp = () => {
  if (!isDragging) return;
  isDragging = false;
  if (rafId) {
    cancelAnimationFrame(rafId);
    rafId = null;
  }

  if (tempDrawing) {
    const newDrawing = {
      ...tempDrawing,
      id: Date.now() + Math.random().toString(36).substring(2, 9)
    };
    drawings.value.push(newDrawing);
    selectedDrawingId.value = newDrawing.id;
    tempDrawing = null;
  }

  drawMode.value = 'none';
  if (chart) {
    chart.applyOptions({ handleScroll: true, handleScale: true });
  }
  redrawCanvas();
};

const buildVPItem = (startTime, endTime) => {
  const visibleData = allData.value.slice(0, currentIndex.value);
  if (!visibleData.length) return null;

  const tStart = Math.min(startTime, endTime);
  const tEnd = Math.max(startTime, endTime);

  const klinesInRange = visibleData.filter(k => k.time >= tStart && k.time <= tEnd);
  if (klinesInRange.length === 0) return null;

  let minPrice = Infinity;
  let maxPrice = -Infinity;

  klinesInRange.forEach(k => {
    if (k.low < minPrice) minPrice = k.low;
    if (k.high > maxPrice) maxPrice = k.high;
  });

  const binCount = 20;
  const priceStep = (maxPrice - minPrice) / binCount;
  const bins = new Array(binCount).fill(0);

  klinesInRange.forEach(k => {
    const avgPrice = (k.high + k.low + k.close) / 3;
    const binIdx = Math.min(binCount - 1, Math.max(0, Math.floor((avgPrice - minPrice) / (priceStep || 1))));
    bins[binIdx] += (k.volume || 100);
  });

  return { type: 'vp', startTime: tStart, endTime: tEnd, minPrice, maxPrice, priceStep, bins };
};

const checkSelection = (clickX, clickY) => {
  if (!chart || !candleSeries) return;
  let foundId = null;
  for (let i = drawings.value.length - 1; i >= 0; i--) {
    const item = drawings.value[i];
    if (item.type === 'trend') {
      const x1 = chart.timeScale().timeToCoordinate(item.time1);
      const y1 = candleSeries.priceToCoordinate(item.price1);
      const x2 = chart.timeScale().timeToCoordinate(item.time2);
      const y2 = candleSeries.priceToCoordinate(item.price2);
      if (x1 !== null && y1 !== null && x2 !== null && y2 !== null) {
        if (distToSegment({ x: clickX, y: clickY }, { x: x1, y: y1 }, { x: x2, y: y2 }) < 8) {
          foundId = item.id;
          break;
        }
      }
    } else if (item.type === 'fib') {
      const y1 = candleSeries.priceToCoordinate(item.price1);
      const y2 = candleSeries.priceToCoordinate(item.price2);
      if (y1 !== null && y2 !== null) {
        if (clickY >= Math.min(y1, y2) - 5 && clickY <= Math.max(y1, y2) + 5) {
          foundId = item.id;
          break;
        }
      }
    } else if (item.type === 'vp') {
      const startX = chart.timeScale().timeToCoordinate(item.startTime);
      const endX = chart.timeScale().timeToCoordinate(item.endTime);
      if (startX !== null && endX !== null) {
        if (clickX >= Math.min(startX, endX) && clickX <= Math.max(startX, endX)) {
          foundId = item.id;
          break;
        }
      }
    } else if (item.type === 'pen') {
      const hit = item.points.some(p => Math.hypot(clickX - p.x, clickY - p.y) < 10);
      if (hit) {
        foundId = item.id;
        break;
      }
    } else if (item.type === 'rect') {
      const x1 = chart.timeScale().timeToCoordinate(item.time1);
      const x2 = chart.timeScale().timeToCoordinate(item.time2);
      const y1 = candleSeries.priceToCoordinate(item.price1);
      const y2 = candleSeries.priceToCoordinate(item.price2);
      if (x1 !== null && x2 !== null && y1 !== null && y2 !== null) {
        const minX = Math.min(x1, x2);
        const maxX = Math.max(x1, x2);
        const minY = Math.min(y1, y2);
        const maxY = Math.max(y1, y2);
        if (clickX >= minX && clickX <= maxX && clickY >= minY && clickY <= maxY) {
          foundId = item.id;
          break;
        }
      }
    }
  }
  selectedDrawingId.value = foundId;
  redrawCanvas();
};

const distToSegment = (p, v, w) => {
  const l2 = (v.x - w.x) ** 2 + (v.y - w.y) ** 2;
  if (l2 === 0) return Math.hypot(p.x - v.x, p.y - v.y);
  let t = ((p.x - v.x) * (w.x - v.x) + (p.y - v.y) * (w.y - v.y)) / l2;
  t = Math.max(0, Math.min(1, t));
  return Math.hypot(p.x - (v.x + t * (w.x - v.x)), p.y - (v.y + t * (w.y - v.y)));
};

const redrawCanvas = () => {
  const canvas = overlayCanvas.value;
  if (!canvas || !chart || !candleSeries) return;
  const ctx = canvas.getContext('2d');
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  drawings.value.forEach(item => {
    renderItem(ctx, item, item.id === selectedDrawingId.value);
  });
  if (tempDrawing) {
    renderItem(ctx, tempDrawing, false, true);
  }

  if (position.value) {
    const entryY = candleSeries.priceToCoordinate(position.value.entryPrice);
    const entryX = chart.timeScale().timeToCoordinate(position.value.entryTime);

    if (entryY !== null) {
      ctx.beginPath();
      ctx.moveTo(0, entryY);
      ctx.lineTo(canvas.width, entryY);
      ctx.strokeStyle = position.value.type === 'LONG' ? '#089981' : '#f23645';
      ctx.lineWidth = 2;
      ctx.setLineDash([4, 4]);
      ctx.stroke();
      ctx.setLineDash([]);

      ctx.fillStyle = position.value.type === 'LONG' ? '#089981' : '#f23645';
      ctx.font = 'bold 12px sans-serif';
      ctx.fillText(`📍 開倉價 (${position.value.type}): ${position.value.entryPrice}`, 10, entryY - 6);
    }

    if (entryX !== null) {
      ctx.beginPath();
      ctx.arc(entryX, entryY ?? (canvas.height / 2), 6, 0, Math.PI * 2);
      ctx.fillStyle = '#ff9800';
      ctx.fill();
      ctx.lineWidth = 2;
      ctx.strokeStyle = '#ffffff';
      ctx.stroke();
    }
  }

  if (position.value && position.value.takeProfitPrice) {
    const tpY = candleSeries.priceToCoordinate(position.value.takeProfitPrice);
    if (tpY !== null) {
      ctx.beginPath();
      ctx.moveTo(0, tpY);
      ctx.lineTo(canvas.width, tpY);
      ctx.strokeStyle = '#2962ff';
      ctx.lineWidth = 2;
      ctx.setLineDash([6, 6]);
      ctx.stroke();
      ctx.setLineDash([]);

      ctx.fillStyle = '#2962ff';
      ctx.font = 'bold 12px sans-serif';
      ctx.fillText(`🎯 止盈線 (TP: ${position.value.takeProfitPrice})`, 10, tpY - 6);
    }
  }

  if (position.value && position.value.stopLossPrice) {
    const slY = candleSeries.priceToCoordinate(position.value.stopLossPrice);
    if (slY !== null) {
      ctx.beginPath();
      ctx.moveTo(0, slY);
      ctx.lineTo(canvas.width, slY);
      ctx.strokeStyle = '#f23645';
      ctx.lineWidth = 2;
      ctx.setLineDash([6, 6]);
      ctx.stroke();
      ctx.setLineDash([]);

      ctx.fillStyle = '#f23645';
      ctx.font = 'bold 12px sans-serif';
      ctx.fillText(`🛑 停損線 (SL: ${position.value.stopLossPrice})`, 10, slY - 6);
    }
  }
};

const renderItem = (ctx, item, isSelected = false, isTemp = false) => {
  if (!chart || !candleSeries) return;
  if (item.type === 'trend') {
    const x1 = chart.timeScale().timeToCoordinate(item.time1);
    const y1 = candleSeries.priceToCoordinate(item.price1);
    const x2 = chart.timeScale().timeToCoordinate(item.time2);
    const y2 = candleSeries.priceToCoordinate(item.price2);
    if (x1 === null || y1 === null || x2 === null || y2 === null) return;

    ctx.beginPath();
    ctx.moveTo(x1, y1);
    ctx.lineTo(x2, y2);
    ctx.strokeStyle = isTemp ? '#f0b90b' : (isSelected ? '#ff9800' : '#2962ff');
    ctx.lineWidth = isSelected ? 4 : 2;
    ctx.stroke();
  } else if (item.type === 'fib') {
    const high = Math.max(item.price1, item.price2);
    const low = Math.min(item.price1, item.price2);
    const diff = high - low;
    const levels = [
      { ratio: 0, color: '#787b86' },
      { ratio: 0.236, color: '#f23645' },
      { ratio: 0.382, color: '#ff9800' },
      { ratio: 0.5, color: '#4caf50' },
      { ratio: 0.618, color: '#089981' },
      { ratio: 0.786, color: '#ab47bc' },
      { ratio: 1, color: '#787b86' }
    ];

    levels.forEach(({ ratio, color }) => {
      const price = high - diff * ratio;
      const y = candleSeries.priceToCoordinate(price);
      if (y === null) return;

      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(overlayCanvas.value.width, y);
      ctx.strokeStyle = isSelected ? '#ff9800' : color;
      ctx.lineWidth = isSelected ? 2 : 1;
      ctx.setLineDash([4, 4]);
      ctx.stroke();
      ctx.setLineDash([]);

      ctx.fillStyle = isSelected ? '#ff9800' : color;
      ctx.font = '11px sans-serif';
      ctx.fillText(`Fib ${ratio} (${price.toFixed(2)})`, 10, y - 4);
    });
  } else if (item.type === 'vp') {
    const { minPrice, priceStep, bins, startTime, endTime } = item;
    const maxVol = Math.max(...bins);
    if (maxVol === 0) return;
    const startX = chart.timeScale().timeToCoordinate(startTime);
    const endX = chart.timeScale().timeToCoordinate(endTime);
    if (startX === null || endX === null) return;

    const regionWidth = Math.abs(endX - startX);
    const leftX = Math.min(startX, endX);

    ctx.fillStyle = isSelected ? 'rgba(255, 152, 0, 0.15)' : 'rgba(41, 98, 255, 0.08)';
    ctx.fillRect(leftX, 0, regionWidth, overlayCanvas.value.height);

    bins.forEach((vol, i) => {
      const priceLow = minPrice + i * priceStep;
      const priceHigh = priceLow + priceStep;
      const yTop = candleSeries.priceToCoordinate(priceHigh);
      const yBottom = candleSeries.priceToCoordinate(priceLow);
      if (yTop === null || yBottom === null) return;

      const barHeight = Math.max(1, Math.abs(yBottom - yTop));
      const barWidth = (vol / maxVol) * regionWidth * 0.8;
      ctx.fillStyle = isSelected ? 'rgba(255, 152, 0, 0.6)' : 'rgba(38, 166, 154, 0.45)';
      ctx.fillRect(leftX, yTop, barWidth, barHeight);
    });
  } else if (item.type === 'pen') {
    if (!item.points || item.points.length === 0) return;
    ctx.beginPath();
    item.points.forEach((p, idx) => {
      const px = p.x;
      const py = p.y;
      if (idx === 0) ctx.moveTo(px, py);
      else ctx.lineTo(px, py);
    });
    ctx.strokeStyle = isTemp ? '#f0b90b' : (isSelected ? '#ff9800' : '#2962ff');
    ctx.lineWidth = isSelected ? 4 : 2;
    ctx.stroke();
  } else if (item.type === 'rect') {
    const x1 = chart.timeScale().timeToCoordinate(item.time1);
    const x2 = chart.timeScale().timeToCoordinate(item.time2);
    const y1 = candleSeries.priceToCoordinate(item.price1);
    const y2 = candleSeries.priceToCoordinate(item.price2);
    if (x1 === null || x2 === null || y1 === null || y2 === null) return;

    const minX = Math.min(x1, x2);
    const maxX = Math.max(x1, x2);
    const minY = Math.min(y1, y2);
    const maxY = Math.max(y1, y2);
    const width = maxX - minX;
    const height = maxY - minY;

    ctx.fillStyle = isSelected ? 'rgba(255, 152, 0, 0.2)' : 'rgba(41, 98, 255, 0.1)';
    ctx.fillRect(minX, minY, width, height);

    ctx.strokeStyle = isTemp ? '#f0b90b' : (isSelected ? '#ff9800' : '#2962ff');
    ctx.lineWidth = isSelected ? 3 : 2;
    ctx.strokeRect(minX, minY, width, height);
  }
};

const deleteSelected = () => {
  if (!selectedDrawingId.value) return;
  drawings.value = drawings.value.filter(d => d.id !== selectedDrawingId.value);
  selectedDrawingId.value = null;
  redrawCanvas();
};

const clearAllDrawings = () => {
  drawings.value = [];
  selectedDrawingId.value = null;
  tempDrawing = null;
  redrawCanvas();
};

const toggleReplaySelectMode = () => {
  if (isPlaying.value) togglePlay();
  isReplaySelectMode.value = !isReplaySelectMode.value;
  drawMode.value = 'none';
};

const togglePlay = () => {
  if (isPlaying.value) {
    clearInterval(timer);
    timer = null;
    isPlaying.value = false;
  } else {
    if (currentIndex.value >= allData.value.length) {
      currentIndex.value = 100;
    }
    isPlaying.value = true;
    timer = setInterval(() => {
      if (currentIndex.value < allData.value.length) {
        currentIndex.value++;
        renderChart();
      } else {
        togglePlay();
      }
    }, 500);
  }
};

const nextFrame = () => {
  if (isPlaying.value) togglePlay();
  if (currentIndex.value < allData.value.length) {
    currentIndex.value++;
    renderChart();
  }
};

const resetReplay = () => {
  if (isPlaying.value) togglePlay();
  currentIndex.value = allData.value.length;
  renderChart();
  if (chart) chart.timeScale().fitContent();
};

onUnmounted(() => {
  isUnmounted = true;
  if (resizeObserver) resizeObserver.disconnect();
  if (timer) clearInterval(timer);
  if (rafId) cancelAnimationFrame(rafId);
  if (chart) {
    chart.remove();
    chart = null;
  }
});
</script>
<style>
html, body, #app {
  width: 100%; height: 100%; margin: 0; padding: 0;
  background-color: #161a25; overflow: hidden;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.app-container {
  display: flex; flex-direction: column; width: 100vw; height: 100vh;
  outline: none;
}

.toolbar {
  display: flex; align-items: center; gap: 8px; padding: 8px 16px;
  background-color: #1e222d; border-bottom: 1px solid #2b2b43; z-index: 20;
}

.trading-panel {
  display: flex; align-items: center; justify-content: space-between; padding: 6px 16px;
  background-color: #161a25; border-bottom: 1px solid #2b2b43; z-index: 19;
  font-size: 13px; color: #d1d4dc;
}

.account-info {
  display: flex; gap: 16px; align-items: center;
}

.trading-actions {
  display: flex; gap: 8px; align-items: center;
}

.symbol-select {
  padding: 6px 10px; background-color: #2a2e39; color: #d1d4dc;
  border: 1px solid #363a45; border-radius: 4px; cursor: pointer; font-size: 13px;
  outline: none;
}
.symbol-select:hover { border-color: #2962ff; }

.input-group {
  display: flex; align-items: center; gap: 4px; background-color: #2a2e39;
  border: 1px solid #363a45; border-radius: 4px; padding: 2px 8px; color: #d1d4dc;
  font-size: 13px;
}

.margin-input {
  width: 70px; background: transparent; border: none; color: #fff;
  font-size: 13px; outline: none; font-weight: bold; text-align: right;
}

.leverage-select {
  color: #ffeb3b; font-weight: bold; background-color: #2a2e39;
}

.divider { width: 1px; height: 20px; background-color: #363a45; margin: 0 4px; }

button {
  padding: 6px 12px; background-color: #2a2e39; color: #d1d4dc;
  border: 1px solid #363a45; border-radius: 4px; cursor: pointer; font-size: 13px;
  transition: all 0.2s; user-select: none;
}
button:disabled { opacity: 0.4; cursor: not-allowed; }
button.active { background-color: #2962ff; color: #fff; border-color: #2962ff; }

.btn-long { background-color: #089981; border-color: #089981; color: #fff; }
.btn-long:hover:not(:disabled) { background-color: #067a67; }

.btn-short { background-color: #f23645; border-color: #f23645; color: #fff; }
.btn-short:hover:not(:disabled) { background-color: #c92a37; }

.btn-close { background-color: #ff9800; border-color: #ff9800; color: #000; font-weight: bold; }
.btn-close:hover:not(:disabled) { background-color: #e68900; }

.btn-quiz { background-color: #7b1fa2; border-color: #9c27b0; color: #fff; }
.btn-quiz:hover { background-color: #6a1b9a; }

.btn-delete-selected { background-color: #3a2e1d; border-color: #ff9800; color: #ff9800; }
.btn-clear-all { background-color: #3a242b; border-color: #f23645; color: #f23645; }

.text-green { color: #089981; }
.text-red { color: #f23645; }

.quiz-stats {
  font-size: 13px; color: #00bcd4; margin-left: 4px; font-weight: bold;
}

.quiz-hint {
  font-size: 12px; color: #ffeb3b; margin-left: 4px;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { opacity: 0.6; }
  50% { opacity: 1; }
  100% { opacity: 0.6; }
}

.chart-wrapper {
  flex: 1; width: 100%; height: 100%; position: relative;
}

.chart-container { width: 100%; height: 100%; }

.overlay-canvas {
  position: absolute; top: 0; left: 0;
  pointer-events: none;
  z-index: 10;
}
</style>