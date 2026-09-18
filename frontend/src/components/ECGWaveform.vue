<template>
  <div class="ecg-waveform-container">
    <div class="flex items-center justify-between mb-2">
      <h3 class="text-lg font-semibold text-emerald-400">
        心电图 - 导联 {{ leadName }}
      </h3>
      <div class="flex items-center gap-2">
        <span class="text-xs text-gray-400">{{ samplingRate }} Hz</span>
        <span class="text-xs text-gray-400">{{ duration }}s</span>
        <span class="text-xs px-1.5 py-0.5 rounded bg-emerald-500/10 border border-emerald-500/30 text-emerald-300">
          {{ rangeLabel }}
        </span>
        <button
          class="text-xs px-2 py-0.5 rounded border transition-all"
          :class="
            isZoomed
              ? 'border-emerald-500/50 text-emerald-300 hover:bg-emerald-500/10'
              : 'border-gray-700 text-gray-600 cursor-not-allowed'
          "
          :disabled="!isZoomed"
          title="恢复整段浏览（纵向振幅恢复为全程范围）"
          @click="resetView"
        >
          重置缩放
        </button>
      </div>
    </div>
    <v-chart
      ref="chartRef"
      class="ecg-chart"
      :option="chartOption"
      autoresize
      @datazoom="onDataZoom"
      @restore="onRestore"
    />
    <p class="mt-1 text-[11px] text-gray-500">
      工具栏可框选一段时间放大；图表内支持滚轮缩放、按住拖动平移；纵向振幅刻度随所选区间自动收放。
    </p>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, ref, shallowRef, watch } from 'vue';
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
  ToolboxComponent,
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
  ToolboxComponent,
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

// 当前缩放区间（秒），仅用于界面提示；图表交互本身不经过 Vue 重渲染
const viewStart = ref(0);
const viewEnd = ref(0);
const isZoomed = ref(false);

// 上一次下发给 y 轴的范围，避免拖动时重复 setOption
let lastYMin = NaN;
let lastYMax = NaN;

/**
 * 密集取点时不再预先丢弃采样点：
 * - 数据以 [时间, 振幅] 二元组一次性下发，交给 ECharts 内置的 LTTB 降采样渲染，
 *   缩放越深入自动还原越细的波形，峰值不会被跳点削平；
 * - tooltip / 框选 / 平移全部基于原始坐标，hover 读数与放大后的时间电压坐标一致。
 * shallowRef 避免大数据量下的深层响应式开销。
 */
const seriesData = shallowRef<[number, number][]>([]);

function buildSeriesData(samples: number[], sr: number): [number, number][] {
  const data: [number, number][] = new Array(samples.length);
  for (let i = 0; i < samples.length; i++) {
    data[i] = [i / sr, samples[i]];
  }
  return data;
}

/** 计算全程数据的纵向范围，作为整段浏览时的初始 / 复位刻度 */
function fullDataRange(): { min: number; max: number } {
  const samples = props.samples;
  let min = Infinity;
  let max = -Infinity;
  for (let i = 0; i < samples.length; i++) {
    const v = samples[i];
    if (v < min) min = v;
    if (v > max) max = v;
  }
  if (!isFinite(min) || !isFinite(max)) {
    min = -1.5;
    max = 1.8;
  }
  return niceBounds(min, max);
}

/**
 * 生成稳定、整齐的纵向刻度范围：
 * 取"好看"的步长（1/2/5 × 10^n）并向外取整。
 * 拖动平移时只要数据极值落在同一档刻度内，min/max 就不变，
 * 刻度不会抖动，波形也不会因为刻度变化而纵向漂走。
 */
function niceBounds(rawMin: number, rawMax: number): { min: number; max: number } {
  let min = rawMin;
  let max = rawMax;
  if (max - min < 1e-6) {
    min -= 0.1;
    max += 0.1;
  }
  // 预留约 12% 的边距，避免 R 峰贴顶
  const span = max - min;
  min -= span * 0.12;
  max += span * 0.12;

  const roughStep = (max - min) / 6;
  const mag = Math.pow(10, Math.floor(Math.log10(roughStep)));
  const norm = roughStep / mag;
  let step: number;
  if (norm <= 1) step = 1;
  else if (norm <= 2) step = 2;
  else if (norm <= 5) step = 5;
  else step = 10;
  step *= mag;

  return {
    min: Math.floor(min / step) * step,
    max: Math.ceil(max / step) * step,
  };
}

const initialY = computed(() => fullDataRange());

const rangeLabel = computed(() => {
  if (!isZoomed.value) return `整段 0.00 - ${props.duration.toFixed(2)}s`;
  return `${viewStart.value.toFixed(2)} - ${viewEnd.value.toFixed(2)}s`;
});

const chartOption = computed(() => {
  seriesData.value = buildSeriesData(props.samples, props.samplingRate);

  // R 峰标注使用真实 [时间, 振幅] 数据坐标，任何缩放级别都与曲线上的峰值吻合
  const rPeakMarkers = props.rPeaks.map((rp) => ({
    name: 'R',
    coord: [rp.time, rp.amplitude],
    value: `${rp.amplitude.toFixed(2)} mV`,
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
  }));

  const y0 = initialY.value;
  lastYMin = y0.min;
  lastYMax = y0.max;

  return {
    backgroundColor: '#0a0a0a',
    animation: false,
    grid: {
      left: 60,
      right: 30,
      top: 36,
      bottom: 56,
    },
    toolbox: {
      right: 10,
      top: 0,
      itemSize: 14,
      iconStyle: { borderColor: '#9ca3af' },
      emphasis: { iconStyle: { borderColor: '#10b981' } },
      feature: {
        // 框选放大：仅沿时间轴框选，纵向范围由选中区间数据自动决定
        dataZoom: {
          title: { zoom: '框选放大', back: '撤销框选' },
          xAxisIndex: 'all',
          yAxisIndex: false,
        },
        restore: { title: '重置缩放' },
      },
    },
    tooltip: {
      trigger: 'axis',
      triggerOn: 'mousemove',
      // 吸附到最近的真实采样点，密点悬停不卡
      axisPointer: {
        type: 'line',
        snap: true,
        lineStyle: { color: 'rgba(16, 185, 129, 0.5)' },
        label: {
          backgroundColor: '#10b981',
          formatter: (p: any) => `${Number(p.value).toFixed(3)}s`,
        },
      },
      backgroundColor: 'rgba(0, 0, 0, 0.85)',
      borderColor: '#10b981',
      textStyle: { color: '#fff', fontSize: 12 },
      formatter: (params: any) => {
        const p = Array.isArray(params) ? params[0] : params;
        const point = p?.value as [number, number] | undefined;
        if (!point) return '';
        return `时间: ${point[0].toFixed(3)}s<br/>振幅: ${point[1].toFixed(3)} mV`;
      },
    },
    xAxis: {
      type: 'value',
      name: '时间 (s)',
      nameLocation: 'middle',
      nameGap: 30,
      min: 0,
      max: props.duration,
      nameTextStyle: { color: '#9ca3af', fontSize: 11 },
      axisLine: { lineStyle: { color: '#374151' } },
      axisLabel: {
        color: '#9ca3af',
        fontSize: 10,
        formatter: (v: number) => v.toFixed(1),
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
      min: y0.min,
      max: y0.max,
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
        type: 'inside',
        xAxisIndex: 0,
        start: 0,
        end: 100,
        zoomOnMouseWheel: true,
        moveOnMouseMove: true,
        moveOnMouseWheel: false,
        filterMode: 'none',
      },
      {
        type: 'slider',
        xAxisIndex: 0,
        height: 20,
        bottom: 8,
        start: 0,
        end: 100,
        borderColor: '#374151',
        backgroundColor: 'rgba(17, 24, 39, 0.6)',
        fillerColor: 'rgba(16, 185, 129, 0.15)',
        handleStyle: { color: '#10b981' },
        moveHandleStyle: { color: '#10b981' },
        selectedDataBackground: {
          lineStyle: { color: '#10b981', opacity: 0.5 },
          areaStyle: { color: '#10b981', opacity: 0.15 },
        },
        textStyle: { color: '#9ca3af' },
        labelFormatter: (v: number) => `${Number(v).toFixed(1)}s`,
        filterMode: 'none',
      },
    ],
    series: [
      {
        name: 'ECG',
        type: 'line',
        data: seriesData.value,
        showSymbol: false,
        sampling: 'lttb',
        clip: true,
        lineStyle: {
          color: '#10b981',
          width: 1.5,
        },
        markPoint: {
          data: rPeakMarkers,
          animation: false,
        },
        z: 10,
      },
    ],
  };
});

/**
 * 框选 / 滚轮 / 拖动条导致时间范围变化时：
 * 1. inside 与 slider 两个 dataZoom 由 ECharts 原生联动，图表与拖动条始终是同一范围；
 * 2. 只扫描可见窗口内的数据，纵向刻度随选中区间的高低自动收放；
 * 3. 只做 y 轴的局部 setOption（lazyUpdate），不触碰 x 轴缩放，
 *    所以刻度变化时心电图波形不会跟着横向漂走；
 * 4. 刻度按整齐步长取整，平移过程中同一档位不重复下发，避免抖动。
 */
function onDataZoom() {
  const chart = chartRef.value?.chart;
  if (!chart) return;

  const option = chart.getOption() as any;
  const dz = option.dataZoom?.[0];
  if (!dz) return;

  // 优先使用实际起止值（秒），百分比模式下按 duration 换算
  let startSec: number;
  let endSec: number;
  if (dz.startValue !== undefined && dz.endValue !== undefined) {
    startSec = Number(dz.startValue);
    endSec = Number(dz.endValue);
  } else {
    startSec = ((dz.start ?? 0) / 100) * props.duration;
    endSec = ((dz.end ?? 100) / 100) * props.duration;
  }
  startSec = Math.max(0, startSec);
  endSec = Math.min(props.duration, endSec);

  viewStart.value = startSec;
  viewEnd.value = endSec;
  isZoomed.value = endSec - startSec < props.duration - 1e-6;

  const sr = props.samplingRate;
  let i0 = Math.floor(startSec * sr);
  let i1 = Math.ceil(endSec * sr);
  i0 = Math.max(0, Math.min(i0, props.samples.length - 1));
  i1 = Math.max(i0 + 1, Math.min(i1, props.samples.length));

  let min = Infinity;
  let max = -Infinity;
  for (let i = i0; i < i1; i++) {
    const v = props.samples[i];
    if (v < min) min = v;
    if (v > max) max = v;
  }
  if (!isFinite(min) || !isFinite(max)) return;

  const bounds = niceBounds(min, max);
  if (bounds.min === lastYMin && bounds.max === lastYMax) return;
  lastYMin = bounds.min;
  lastYMax = bounds.max;

  chart.setOption(
    {
      yAxis: { min: bounds.min, max: bounds.max },
    },
    { lazyUpdate: true }
  );
}

function onRestore() {
  const y = fullDataRange();
  lastYMin = y.min;
  lastYMax = y.max;
  viewStart.value = 0;
  viewEnd.value = props.duration;
  isZoomed.value = false;
}

/** 手动重置：时间轴回到整段，纵向刻度回到全程范围 */
function resetView() {
  const chart = chartRef.value?.chart;
  if (!chart) return;
  chart.dispatchAction({
    type: 'dataZoom',
    start: 0,
    end: 100,
  });
  const y = fullDataRange();
  lastYMin = y.min;
  lastYMax = y.max;
  chart.setOption({ yAxis: { min: y.min, max: y.max } }, { lazyUpdate: true });
  viewStart.value = 0;
  viewEnd.value = props.duration;
  isZoomed.value = false;
}

/**
 * 切换导联或重新出图（samples / leadName / 采样率变化）后，
 * 缩放范围必须回到初始整段，不沿用上一次的区间。
 * notMerge 更新会重建 dataZoom，这里再显式复位并同步纵向刻度。
 */
watch(
  () => [props.samples, props.leadName, props.samplingRate, props.duration] as const,
  () => {
    viewStart.value = 0;
    viewEnd.value = props.duration;
    isZoomed.value = false;
    const y = fullDataRange();
    lastYMin = y.min;
    lastYMax = y.max;
    nextTick(() => {
      const chart = chartRef.value?.chart;
      if (!chart || chart.isDisposed()) return;
      chart.dispatchAction({ type: 'dataZoom', start: 0, end: 100 });
      chart.setOption({ yAxis: { min: y.min, max: y.max } }, { lazyUpdate: true });
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

.ecg-chart {
  width: 100%;
  height: 320px;
}
</style>
