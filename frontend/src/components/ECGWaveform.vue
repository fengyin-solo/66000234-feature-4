<template>
  <div class="ecg-waveform-container">
    <div class="flex items-center justify-between mb-2">
      <h3 class="text-lg font-semibold text-emerald-400">
        心电图 - 导联 {{ leadName }}
      </h3>
      <div class="flex items-center gap-3">
        <span class="text-xs text-gray-400">{{ samplingRate }} Hz</span>
        <span class="text-xs text-gray-400">{{ duration }}s</span>
        <span class="text-xs text-gray-300 tabular-nums">
          范围: {{ ((viewStartPct / 100) * duration).toFixed(2) }}s –
          {{ ((viewEndPct / 100) * duration).toFixed(2) }}s
        </span>
        <button
          v-if="viewStartPct > 0.01 || viewEndPct < 99.99"
          class="text-xs px-2 py-0.5 rounded border border-emerald-500/50 text-emerald-400 hover:bg-emerald-500/10 transition-colors"
          @click="resetZoom"
        >
          复位
        </button>
      </div>
    </div>
    <p class="text-[11px] text-gray-500 mb-1">
      在波形区按住鼠标框选时间段放大 · 滚轮缩放 / 拖动条平移 · 双击复位
    </p>
    <div class="ecg-chart-wrapper relative select-none">
      <v-chart
        ref="chartRef"
        class="ecg-chart"
        :option="chartOption"
        autoresize
        :update-options="{ notMerge: true }"
        @zr:mousedown="onBrushStart"
        @zr:dblclick="onChartDblClick"
      />
      <!-- 框选矩形，覆盖绘图区纵向范围 -->
      <div
        v-if="brushing && brushWidth > 3"
        class="brush-rect"
        :style="{
          left: brushLeft + 'px',
          top: gridRect.y + 'px',
          width: brushWidth + 'px',
          height: gridRect.height + 'px',
        }"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, ref, watch } from 'vue';
import VChart from 'vue-echarts';
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { LineChart, ScatterChart } from 'echarts/charts';
import {
  TitleComponent,
  TooltipComponent,
  GridComponent,
  MarkPointComponent,
  DataZoomComponent,
  LegendComponent,
} from 'echarts/components';
import type { RPeak } from '../types';

use([
  CanvasRenderer,
  LineChart,
  ScatterChart,
  TitleComponent,
  TooltipComponent,
  GridComponent,
  MarkPointComponent,
  DataZoomComponent,
  LegendComponent,
]);

const props = withDefaults(
  defineProps<{
    samples: number[];
    rPeaks: RPeak[];
    leadName: string;
    samplingRate: number;
    duration: number;
    scrollOffset?: number;
  }>(),
  {
    scrollOffset: 0,
  }
);

const chartRef = ref<InstanceType<typeof VChart> | null>(null);

// ---- 当前视图（百分比 0~100，inside 与 slider 共用同一区间）----
const viewStartPct = ref(0);
const viewEndPct = ref(100);

// ---- 框选状态 ----
const brushing = ref(false);
const brushStartX = ref(0);
const brushCurrentX = ref(0);
const gridRect = ref({ x: 0, y: 0, width: 0, height: 0 });

const brushLeft = computed(() => Math.min(brushStartX.value, brushCurrentX.value));
const brushWidth = computed(() => Math.abs(brushCurrentX.value - brushStartX.value));

// ---- 数据预处理：数值时间轴上的 [time(s), amplitude(mV)] 点对 ----
// 仅依赖 props，悬停/拖动/缩放都不会触发重建，保证密点下交互不被出图拖慢。
const points = computed<[number, number][]>(() => {
  const sr = props.samplingRate;
  return props.samples.map((v, i) => [i / sr, v]);
});

// 全段初始纵向量程（以 0 为中心对称，零基线位置固定，缩放时波形不会纵向漂走）
const initialYLimit = computed(() => {
  let limit = 0.1;
  for (const v of props.samples) {
    const a = Math.abs(v);
    if (a > limit) limit = a;
  }
  return limit * 1.15;
});

// R 峰标注使用真实时间-电压坐标，任何缩放级别都与曲线峰值重合
const rPeakMarkers = computed(() =>
  props.rPeaks.map((rp) => ({
    name: 'R',
    coord: [rp.time, rp.amplitude] as [number, number],
    symbol: 'triangle',
    symbolSize: 10,
    itemStyle: { color: '#ef4444' },
    label: {
      show: true,
      formatter: 'R',
      color: '#ef4444',
      fontSize: 10,
      position: 'top' as const,
    },
  }))
);

const chartOption = computed(() => ({
  backgroundColor: '#0a0a0a',
  animation: false,
  grid: {
    left: 60,
    right: 30,
    top: 30,
    bottom: 50,
  },
  tooltip: {
    trigger: 'axis',
    axisPointer: {
      type: 'line',
      snap: true, // 吸附到最近采样点，读数与放大后的坐标严格对应
      lineStyle: { color: 'rgba(16, 185, 129, 0.5)' },
    },
    backgroundColor: 'rgba(0, 0, 0, 0.8)',
    borderColor: '#10b981',
    textStyle: { color: '#fff', fontSize: 12 },
    formatter: (params: any) => {
      const p = Array.isArray(params) ? params[0] : params;
      const value: [number, number] | undefined = p?.value;
      if (!value) return '';
      return `时间: ${value[0].toFixed(3)}s<br/>振幅: ${value[1].toFixed(3)} mV`;
    },
  },
  xAxis: {
    type: 'value',
    min: 0,
    max: props.duration,
    name: '时间 (s)',
    nameTextStyle: { color: '#9ca3af', fontSize: 11 },
    axisLine: { lineStyle: { color: '#374151' } },
    axisLabel: {
      color: '#9ca3af',
      fontSize: 10,
      formatter: (v: number) => v.toFixed(2),
    },
    splitLine: {
      show: true,
      lineStyle: { color: 'rgba(16, 185, 129, 0.08)', type: 'dashed' },
    },
  },
  yAxis: {
    type: 'value',
    name: 'mV',
    nameTextStyle: { color: '#9ca3af', fontSize: 11 },
    min: -initialYLimit.value,
    max: initialYLimit.value,
    axisLine: { lineStyle: { color: '#374151' } },
    axisLabel: {
      color: '#9ca3af',
      fontSize: 10,
      formatter: (v: number) => v.toFixed(2),
    },
    splitLine: {
      show: true,
      lineStyle: { color: 'rgba(16, 185, 129, 0.08)', type: 'dashed' },
    },
  },
  dataZoom: [
    {
      // 滚轮缩放（原整段浏览方式保留）；普通拖拽用于框选，按住 Shift 拖拽平移
      type: 'inside',
      xAxisIndex: 0,
      start: 0,
      end: 100,
      moveOnMouseMove: 'shift',
      zoomOnMouseWheel: true,
    },
    {
      type: 'slider',
      xAxisIndex: 0,
      start: 0,
      end: 100,
      height: 20,
      bottom: 5,
      borderColor: '#374151',
      fillerColor: 'rgba(16, 185, 129, 0.15)',
      handleStyle: { color: '#10b981' },
      textStyle: { color: '#9ca3af' },
      labelFormatter: (v: number) => `${Number(v).toFixed(2)}s`,
    },
  ],
  series: [
    {
      name: 'ECG',
      type: 'line',
      data: points.value,
      showSymbol: false,
      // 密点时由 ECharts 按当前可视宽度做 LTTB 抽样：保留峰值形态，
      // 且随缩放重新计算（放大后看到的是真实细节而非固定抽稀点）。
      // 抽样只影响绘制路径，数据数组保持全量，峰值标注/读数仍取真实点。
      sampling: 'lttb',
      lineStyle: {
        color: '#10b981',
        width: 1.5,
      },
      markPoint: {
        data: rPeakMarkers.value,
        animation: false,
        tooltip: { show: false },
      },
      z: 10,
    },
  ],
}));

// vue-echarts 组件代理仅暴露少量方法，原始 ECharts 实例挂在 `.chart`
function getChart(): any {
  const ref: any = chartRef.value;
  return ref ? (ref.chart ?? ref) : null;
}

// ---- 绘图区几何（框选矩形定位用）----
function updateGridRect() {
  const chart = getChart();
  if (!chart) return;
  try {
    const model = chart.getModel?.();
    const rect = model
      ?.getComponent?.('grid')
      ?.coordinateSystem?.getRect?.();
    if (rect) {
      gridRect.value = { x: rect.x, y: rect.y, width: rect.width, height: rect.height };
      return;
    }
  } catch {
    // fall through to fallback
  }
  const width: number = chart.getWidth?.() ?? 0;
  const height: number = chart.getHeight?.() ?? 0;
  gridRect.value = { x: 60, y: 30, width: Math.max(0, width - 90), height: Math.max(0, height - 80) };
}

// ---- 框选放大 ----
function onBrushStart(zrEvent: any) {
  const chart = getChart();
  if (!chart) return;
  const x: number = zrEvent?.offsetX;
  const y: number = zrEvent?.offsetY;
  if (x == null || y == null) return;
  // 按住 Shift 时交给 inside dataZoom 平移，不启动框选
  if (zrEvent?.event?.shiftKey || zrEvent?.shiftKey) return;
  // 只在绘图区内起选（避开坐标轴/拖动条）
  if (!chart.containPixel('grid', [x, y])) return;

  updateGridRect();
  brushing.value = true;
  brushStartX.value = x;
  brushCurrentX.value = x;

  window.addEventListener('mousemove', onBrushMove);
  window.addEventListener('mouseup', onBrushEnd);
}

function onBrushMove(e: MouseEvent) {
  const chart = getChart();
  if (!chart || !brushing.value) return;
  const rect = chart.getDom?.().getBoundingClientRect();
  if (!rect) return;
  const x = Math.min(Math.max(e.clientX - rect.left, gridRect.value.x), gridRect.value.x + gridRect.value.width);
  brushCurrentX.value = x;
  chart.dispatchAction({ type: 'hideTip' });
}

function onBrushEnd() {
  window.removeEventListener('mousemove', onBrushMove);
  window.removeEventListener('mouseup', onBrushEnd);

  const wasBrushing = brushing.value;
  brushing.value = false;
  if (!wasBrushing) return;

  // 宽度过小视为点击，不改变缩放
  if (brushWidth.value <= 3) return;

  const chart = getChart();
  if (!chart) return;
  const t0 = clampTime(chart.convertFromPixel({ xAxisIndex: 0 }, brushLeft.value));
  const t1 = clampTime(
    chart.convertFromPixel({ xAxisIndex: 0 }, brushLeft.value + brushWidth.value)
  );
  if (t1 - t0 < 1e-6) return;

  // 不指定 dataZoomIndex：inside 与底部 slider 同步收放到同一区间
  chart.dispatchAction({
    type: 'dataZoom',
    startValue: t0,
    endValue: t1,
  });
}

function clampTime(t: number): number {
  if (!Number.isFinite(t)) return 0;
  return Math.min(Math.max(t, 0), props.duration);
}

function onChartDblClick(zrEvent: any) {
  const chart = getChart();
  if (!chart) return;
  if (!chart.containPixel('grid', [zrEvent?.offsetX, zrEvent?.offsetY])) return;
  resetZoom();
}

function resetZoom() {
  const chart = getChart();
  if (!chart) return;
  chart.dispatchAction({ type: 'dataZoom', start: 0, end: 100 });
}

// ---- 随选中区间自动收放纵向振幅刻度 ----
let yRangeFrame = 0;
let currentYLimit = initialYLimit.value;

function scheduleYRangeUpdate() {
  if (yRangeFrame) return;
  yRangeFrame = requestAnimationFrame(() => {
    yRangeFrame = 0;
    updateYRange();
  });
}

function updateYRange() {
  const chart = getChart();
  if (!chart) return;

  // 从图表状态读取当前缩放百分比（inside / slider 始终同步）
  const dz = (chart.getOption() as any)?.dataZoom?.[0] ?? {};
  let sPct: number | undefined = dz.start;
  let ePct: number | undefined = dz.end;
  if (sPct == null && dz.startValue != null) sPct = (dz.startValue / props.duration) * 100;
  if (ePct == null && dz.endValue != null) ePct = (dz.endValue / props.duration) * 100;
  sPct = sPct ?? viewStartPct.value;
  ePct = ePct ?? viewEndPct.value;
  viewStartPct.value = sPct;
  viewEndPct.value = ePct;

  const n = props.samples.length;
  if (n === 0) return;
  const i0 = Math.max(0, Math.min(n - 1, Math.floor((sPct / 100) * n)));
  const i1 = Math.max(i0 + 1, Math.min(n, Math.ceil((ePct / 100) * n)));

  let min = Infinity;
  let max = -Infinity;
  for (let i = i0; i < i1; i++) {
    const v = props.samples[i];
    if (v < min) min = v;
    if (v > max) max = v;
  }
  if (!Number.isFinite(min) || !Number.isFinite(max)) return;

  // 以零基线为中心对称收放：刻度变化时波形不漂移
  const limit = Math.max(Math.abs(min), Math.abs(max), 0.05) * 1.15;
  if (Math.abs(limit - currentYLimit) < 1e-9) return;
  currentYLimit = limit;
  chart.setOption({ yAxis: { min: -limit, max: limit } });
}

// 订阅 ECharts 事件（原始实例在首帧 option 提交后才创建，必要时重试）
let chartBound: any = null;
function bindChartEvents(retries = 0) {
  const chart = getChart();
  const rawReady = chart && chartRef.value && (chartRef.value as any).chart === chart;
  if (!rawReady) {
    if (retries < 20) nextTick(() => bindChartEvents(retries + 1));
    return;
  }
  if (chart === chartBound) return;
  chartBound = chart;
  chart.on('dataZoom', scheduleYRangeUpdate);
  updateGridRect();
}
watch(chartRef, () => bindChartEvents(), { immediate: true });

// 切换导联或重新出图（新数据）后，缩放回到初始整段区间，不沿用上次
watch(
  () => [props.samples, props.leadName, props.samplingRate, props.duration],
  () => {
    viewStartPct.value = 0;
    viewEndPct.value = 100;
    currentYLimit = initialYLimit.value;
    nextTick(() => {
      const chart = getChart();
      chart?.dispatchAction({ type: 'dataZoom', start: 0, end: 100 });
      updateGridRect();
    });
  }
);
</script>

<style scoped>
.ecg-waveform-container {
  width: 100%;
  background: #0a0a0a;
  border: 1px solid rgba(16, 185, 129, 0.2);
  border-radius: 8px;
  padding: 16px;
}

.ecg-chart-wrapper {
  width: 100%;
}

.ecg-chart {
  width: 100%;
  height: 320px;
  cursor: crosshair;
}

.brush-rect {
  position: absolute;
  pointer-events: none;
  background: rgba(16, 185, 129, 0.18);
  border: 1px dashed rgba(16, 185, 129, 0.9);
  z-index: 5;
}
</style>
