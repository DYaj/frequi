<script setup lang="ts">
import type { ChartSliderPosition, PairHistory, Trade } from '@/types';
import { LoadingStatus } from '@/types';

interface SetupRow {
  pair: string;
  side: 'LONG' | 'SHORT';
  progress: string;
  progress_value: number | null;
  status: string;
  result: string;
}

interface SetupPayload {
  rows: SetupRow[];
  top_setup: SetupRow | null;
  updated_at?: string;
  source?: string;
}

interface ScannerAlert {
  timestamp: string;
  pair: string;
  side: string | null;
  scanner_stage: string;
  context_status?: string | null;
  context_reason?: string | null;
  near_ready?: boolean;
  fvg_entry_price?: number | null;
  stop_price?: number | null;
  target_price?: number | null;
  rr?: number | null;
}

interface ScannerPayload {
  ok: boolean;
  scanner: string;
  mode: string;
  alerts: ScannerAlert[];
  latest_alert: ScannerAlert | null;
  counts: Record<string, number>;
  rejection_breakdown?: Record<string, number>;
  updated_at?: string;
  error?: string;
}

const props = withDefaults(
  defineProps<{
    trades?: Trade[];
    availablePairs: string[];
    timeframe: string;
    historicView?: boolean;
    pair?: string;
    scannerMode?: string;
    sliderPosition?: ChartSliderPosition;
    isSinglePairView?: boolean;
  }>(),
  {
    trades: () => [],
    historicView: false,
    pair: '',
    scannerMode: 'off',
    sliderPosition: undefined,
    isSinglePairView: true,
  },
);

const emit = defineEmits<{
  refreshData: [pair: string, columns: string[]];
}>();

const settingsStore = useSettingsStore();
const colorStore = useColorStore();
const botStore = useBotStore();
const plotStore = usePlotConfigStore();

const API_BASE = 'http://localhost:5001';
const setupsPayload = ref<SetupPayload>({
  rows: [],
  top_setup: null,
});
const scannerPayload = ref<ScannerPayload | null>(null);
const scannerLoading = ref(false);
let setupPollInterval: number | undefined;
let scannerPollInterval: number | undefined;

const dataset = computed((): PairHistory => {
  if (props.historicView) {
    return botStore.activeBot.history[`${props.pair}__${props.timeframe}`]?.data;
  }
  return botStore.activeBot.candleData[`${props.pair}__${props.timeframe}`]?.data;
});

const chartDataset = computed(() => {
  if (!dataset.value || !dataset.value.data?.length || !dataset.value.columns?.length) {
    return dataset.value;
  }

  const highIndex = dataset.value.columns.findIndex((c) => c === 'high');
  const lowIndex = dataset.value.columns.findIndex((c) => c === 'low');
  const volumeIndex = dataset.value.columns.findIndex((c) => c === 'volume');

  if (highIndex < 0 || lowIndex < 0 || volumeIndex < 0) {
    return dataset.value;
  }

  const recentCandles = dataset.value.data.slice(-50);

  const highs = recentCandles.map((row) => Number(row[highIndex])).filter((v) => !Number.isNaN(v));
  const lows = recentCandles.map((row) => Number(row[lowIndex])).filter((v) => !Number.isNaN(v));

  if (!highs.length || !lows.length) {
    return dataset.value;
  }

  const recentHigh = Math.max(...highs);
  const recentLow = Math.min(...lows);

  const volumeCandles = dataset.value.data.slice(-20);
  const volumes = volumeCandles
    .map((row) => Number(row[volumeIndex]))
    .filter((v) => !Number.isNaN(v));

  const lastRow = dataset.value.data[dataset.value.data.length - 1];
  const currentVolume = Number(lastRow?.[volumeIndex]);

  const avgVolume20 =
    volumes.length > 0 ? volumes.reduce((sum, v) => sum + v, 0) / volumes.length : null;

  const volumeRatio =
    avgVolume20 && avgVolume20 > 0 && !Number.isNaN(currentVolume)
      ? currentVolume / avgVolume20
      : null;

  const highVolumeEvent = volumeRatio !== null ? volumeRatio >= 1.5 : false;

  return {
    ...dataset.value,
    zones: {
      support: [{ low: recentLow * 0.998, high: recentLow * 1.002 }],
      resistance: [{ low: recentHigh * 0.998, high: recentHigh * 1.002 }],
    },
    levels: {
      breakout_above: recentHigh * 1.001,
      breakout_below: recentLow * 0.999,
    },
    volumeContext: {
      current_volume: !Number.isNaN(currentVolume) ? currentVolume : null,
      avg_volume_20: avgVolume20,
      volume_ratio: volumeRatio,
      high_volume_event: highVolumeEvent,
    },
  };
});

const currentPairSetup = computed(() => {
  const pair = props.pair;
  if (!pair) return null;
  return setupsPayload.value.rows.find((row) => row.pair === pair) ?? null;
});

const scannerEnabled = computed(() => props.scannerMode === 'v1_htf_ltf_sweep_fvg');
const latestScannerAlert = computed(() => scannerPayload.value?.latest_alert ?? null);
const latestScannerStage = computed(() => latestScannerAlert.value?.scanner_stage ?? 'Waiting');
const scannerReadyCount = computed(() => scannerPayload.value?.counts?.SETUP_READY ?? 0);
const scannerNearReadyCount = computed(() => {
  const counts = scannerPayload.value?.counts ?? {};
  return Object.entries(counts)
    .filter(([stage]) => stage.startsWith('NEAR_READY_'))
    .reduce((sum, [, value]) => sum + Number(value || 0), 0);
});

const datasetColumns = computed(() =>
  dataset.value ? (dataset.value.all_columns ?? dataset.value.columns) : [],
);

const datasetLoadedColumns = computed(() =>
  dataset.value ? (dataset.value.columns ?? dataset.value.all_columns) : [],
);

const hasDataset = computed(() => !!chartDataset.value && chartDataset.value.data.length > 0);

const isLoadingDataset = computed((): boolean => {
  if (props.historicView) {
    return botStore.activeBot.historyStatus === LoadingStatus.loading;
  }

  return botStore.activeBot.candleDataStatus === LoadingStatus.loading;
});

const noDatasetText = computed((): string => {
  const status = props.historicView
    ? botStore.activeBot.historyStatus
    : botStore.activeBot.candleDataStatus;

  switch (status) {
    case LoadingStatus.not_loaded:
      return 'Not loaded yet.';
    case LoadingStatus.loading:
      return 'Loading...';
    case LoadingStatus.success:
      return 'No data available';
    case LoadingStatus.error:
      return 'Failed to load data';
    default:
      return 'Unknown';
  }
});

function bannerStatusClass(status: string): string {
  if (status.startsWith('Waiting:')) return 'text-orange-600';
  if (status === 'Entered') return 'text-green-600';
  return 'text-surface-900 dark:text-surface-100';
}

function bannerResultClass(result: string): string {
  if (!result) return 'text-surface-900 dark:text-surface-100';
  if (result.startsWith('Open ')) return 'text-green-600';
  if (result.includes('+')) return 'text-blue-600';
  if (result.includes('-')) return 'text-red-600';
  return 'text-surface-900 dark:text-surface-100';
}

async function fetchSetups() {
  try {
    const res = await fetch(`${API_BASE}/api/v1/setups`);
    if (!res.ok) return;
    const data = await res.json();
    setupsPayload.value = {
      rows: Array.isArray(data?.rows) ? data.rows : [],
      top_setup: data?.top_setup ?? null,
      updated_at: data?.updated_at,
      source: data?.source,
    };
  } catch (error) {
    console.error('SingleCandleChartContainer setups fetch failed', error);
  }
}

async function fetchScanner() {
  if (!scannerEnabled.value || !props.pair) {
    scannerPayload.value = null;
    return;
  }

  scannerLoading.value = true;
  try {
    const params = new URLSearchParams({
      pair: props.pair,
      limit: '30',
    });
    const res = await fetch(`${API_BASE}/api/v1/chart-scanner/v1-htf-ltf?${params}`);
    if (!res.ok) throw new Error(`Scanner failed with ${res.status}`);
    scannerPayload.value = await res.json();
  } catch (error) {
    scannerPayload.value = {
      ok: false,
      scanner: props.scannerMode,
      mode: 'scanner_only',
      alerts: [],
      latest_alert: null,
      counts: {},
      error: error instanceof Error ? error.message : 'Scanner fetch failed',
    };
    console.error('SingleCandleChartContainer scanner fetch failed', error);
  } finally {
    scannerLoading.value = false;
  }
}

function refresh() {
  emit('refreshData', props.pair, plotStore.usedColumns);
}

function refreshIfNecessary() {
  if (!hasDataset.value) {
    refresh();
  }
}

function assignFirstPair() {
  const [firstPair] = props.availablePairs;
  if (firstPair) {
    // props.pair = firstPair;
  }
}

watch(
  () => props.availablePairs,
  () => {
    if (!props.availablePairs.find((p) => p === props.pair)) {
      assignFirstPair();
      refresh();
    }
  },
);

watch(
  () => plotStore.plotConfig,
  () => {
    const hasAllColumns = plotStore.usedColumns.some(
      (c) => datasetColumns.value.includes(c) && !datasetLoadedColumns.value.includes(c),
    );

    if (settingsStore.useReducedPairCalls && hasAllColumns) {
      refresh();
    }
  },
);

watch(
  () => props.timeframe,
  () => {
    refreshIfNecessary();
  },
);

watch(
  () => [props.pair, props.scannerMode],
  () => {
    fetchScanner();
  },
);

onMounted(() => {
  fetchSetups();
  fetchScanner();
  setupPollInterval = window.setInterval(fetchSetups, 3000);
  scannerPollInterval = window.setInterval(fetchScanner, 15000);
});

onBeforeUnmount(() => {
  if (setupPollInterval) {
    window.clearInterval(setupPollInterval);
  }
  if (scannerPollInterval) {
    window.clearInterval(scannerPollInterval);
  }
});
</script>

<template>
  <div
    class="flex-fill w-full flex-col align-items-stretch flex"
    :class="{
      'h-full': isSinglePairView,
      'h-150 border border-r border-b border-surface-300 dark:border-surface-700':
        !isSinglePairView,
    }"
  >
    <div
      v-if="currentPairSetup"
      class="mx-1 mt-1 mb-2 rounded border border-surface-300 dark:border-surface-700 px-3 py-2 bg-surface-50 dark:bg-surface-900"
    >
      <div class="text-xs uppercase tracking-wide text-surface-500 dark:text-surface-400">
        Trade Readiness
      </div>
      <div class="mt-1 flex flex-wrap items-center gap-x-3 gap-y-1 text-sm">
        <span class="font-semibold">{{ currentPairSetup.pair }}</span>
        <span>{{ currentPairSetup.side }}</span>
        <span>{{ currentPairSetup.progress }}</span>
        <span :class="bannerStatusClass(currentPairSetup.status)">{{
          currentPairSetup.status
        }}</span>
        <span v-if="currentPairSetup.result" :class="bannerResultClass(currentPairSetup.result)">
          {{ currentPairSetup.result }}
        </span>
      </div>
    </div>

    <div
      v-if="scannerEnabled"
      class="scanner-banner mx-1 mb-2 rounded border px-3 py-2"
      :class="{
        ready: latestScannerAlert?.scanner_stage === 'SETUP_READY',
        near: latestScannerAlert?.scanner_stage?.startsWith('NEAR_READY_'),
        rejected: latestScannerAlert?.scanner_stage === 'CONTEXT_REJECTED',
      }"
    >
      <div class="scanner-banner-top">
        <span class="scanner-title">v1 HTF→LTF Scanner</span>
        <span class="scanner-mode">scanner only</span>
        <span v-if="scannerLoading" class="scanner-muted">refreshing</span>
      </div>
      <div v-if="scannerPayload?.ok" class="scanner-banner-body">
        <span>{{ latestScannerStage }}</span>
        <span v-if="latestScannerAlert?.side">{{ latestScannerAlert.side }}</span>
        <span>Ready {{ scannerReadyCount }}</span>
        <span>Near {{ scannerNearReadyCount }}</span>
        <span v-if="latestScannerAlert?.context_reason">
          {{ latestScannerAlert.context_reason }}
        </span>
        <span v-if="latestScannerAlert?.rr">RR {{ Number(latestScannerAlert.rr).toFixed(2) }}</span>
      </div>
      <div v-else class="scanner-banner-body scanner-error">
        {{ scannerPayload?.error || 'Scanner waiting for data.' }}
      </div>
    </div>

    <div class="flex me-0 w-full items-center justify-between">
      <div class="ms-1 md:ms-2 flex flex-wrap md:flex-nowrap items-center gap-1">
        <div class="flex flex-col md:flex-row md:gap-2">
          <div class="flex flex-row flex-wrap gap-2">
            <small v-if="dataset" class="text-sm text-nowrap">
              Long entries: {{ dataset.enter_long_signals || dataset.buy_signals }}
            </small>
            <small v-if="dataset" class="text-sm text-nowrap">
              Long exit: {{ dataset.exit_long_signals || dataset.sell_signals }}
            </small>
          </div>
          <div class="flex flex-row flex-wrap gap-2">
            <small v-if="dataset && dataset.enter_short_signals" class="text-sm text-nowrap">
              Short entries: {{ dataset.enter_short_signals }}
            </small>
            <small v-if="dataset && dataset.exit_short_signals" class="text-sm text-nowrap">
              Short exits: {{ dataset.exit_short_signals }}
            </small>
          </div>
        </div>

        <div class="flex flex-row flex-wrap gap-2">
          <small v-if="(chartDataset as any)?.volumeContext">
            Vol Ratio:
            {{ (chartDataset as any).volumeContext.volume_ratio?.toFixed?.(2) ?? 'n/a' }}
          </small>

          <small v-if="(chartDataset as any)?.volumeContext">
            High Vol:
            {{ (chartDataset as any).volumeContext.high_volume_event ? 'YES' : 'NO' }}
          </small>
        </div>
      </div>

      <div>
        {{ pair || 'Pair' }}
      </div>

      <div v-if="isLoadingDataset">
        <ProgressSpinner class="w-4 h-4" stroke-width="4" small />
      </div>
      <div v-else class="w-4 h-4"></div>
    </div>

    <div class="h-full flex">
      <div class="min-w-0 w-full flex-1">
        <CandleChart
          v-if="hasDataset"
          :dataset="chartDataset"
          :trades="trades"
          :plot-config="plotStore.plotConfig"
          :heikin-ashi="settingsStore.useHeikinAshiCandles"
          :show-mark-area="settingsStore.showMarkArea"
          :show-zones="true"
          :use-u-t-c="settingsStore.timezone === 'UTC'"
          :theme="settingsStore.chartTheme"
          :slider-position="sliderPosition"
          :color-up="colorStore.colorUp"
          :color-down="colorStore.colorDown"
          :start-candle-count="settingsStore.chartDefaultCandleCount"
          :label-side="settingsStore.chartLabelSide"
        />

        <div v-else class="m-auto">
          <ProgressSpinner v-if="isLoadingDataset" class="w-5 h-5" />
          <div v-else class="text-lg">
            {{ noDatasetText }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.scanner-banner {
  border-color: rgba(71, 85, 105, 0.8);
  background: rgba(15, 23, 42, 0.92);
  color: var(--p-surface-200);
}

.scanner-banner.ready {
  border-color: rgba(34, 197, 94, 0.65);
  background: rgba(20, 83, 45, 0.42);
}

.scanner-banner.near {
  border-color: rgba(245, 158, 11, 0.65);
  background: rgba(120, 53, 15, 0.38);
}

.scanner-banner.rejected {
  border-color: rgba(148, 163, 184, 0.55);
}

.scanner-banner-top,
.scanner-banner-body {
  display: flex;
  min-width: 0;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.35rem 0.65rem;
}

.scanner-title {
  font-size: 0.72rem;
  font-weight: 800;
  letter-spacing: 0.1em;
  text-transform: uppercase;
}

.scanner-mode,
.scanner-muted {
  color: var(--p-surface-400);
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
}

.scanner-banner-body {
  margin-top: 0.25rem;
  font-size: 0.82rem;
  font-weight: 700;
}

.scanner-error {
  color: #fca5a5;
}
</style>
