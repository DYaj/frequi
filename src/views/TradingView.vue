<script setup lang="ts">
const botStore = useBotStore();
const chartStore = useChartConfigStore();
const settingsStore = useSettingsStore();

interface LiveStrategyOption {
  value: string;
  label: string;
}

interface LiveStrategyProfile {
  research_profile: string;
  label: string;
  leverage?: number;
  leverage_options?: number[];
  observed_profile?: string | null;
  observed_timeframe?: string | null;
  available: LiveStrategyOption[];
}

interface ChartScannerOption {
  value: string;
  label: string;
  description: string;
}

const API_BASE = 'http://localhost:5001';
const chartTimeframes = [
  '1s',
  '5s',
  '10s',
  '15s',
  '30s',
  '45s',
  '1m',
  '3m',
  '5m',
  '15m',
  '30m',
  '1h',
];
const chartTimeframe = computed(
  () => chartStore.selectedTimeframe || botStore.activeBot.timeframe || '1m',
);

const hasOpenTrades = computed(() => botStore.activeBot.openTrades.length > 0);
const hasClosedTrades = computed(() => botStore.activeBot.closedTrades.length > 0);
const selectedTrade = computed(() => botStore.activeBot.tradeDetail);
const selectedPairsLabel = computed(() => {
  const pairs = botStore.activeBot.plotMultiPairs.filter(Boolean);
  if (pairs.length === 0) return 'No pair';
  if (pairs.length === 1) return pairs[0];
  return `${pairs.length} pairs`;
});
const botStatusLabel = computed(() => (botStore.activeBot.isBotOnline ? 'Online' : 'Offline'));
const runModeLabel = computed(() =>
  String(botStore.activeBot.botState?.runmode || 'Unknown').replaceAll('_', ' '),
);
const strategyLabel = computed(() => botStore.activeBot.strategy?.strategy || 'Strategy loading');
const liveStrategyProfile = ref<LiveStrategyProfile | null>(null);
const selectedLiveStrategy = ref('');
const selectedLeverage = ref(4);
const liveStrategyStatus = ref('');
const isSavingLiveStrategy = ref(false);
const isSavingLeverage = ref(false);
let liveStrategyPollInterval: number | undefined;
const chartScannerStorageKey = 'ftTradeChartScanner';
const chartScannerOptions: ChartScannerOption[] = [
  {
    value: 'off',
    label: 'Scanner off',
    description: 'Chart scanner disabled.',
  },
  {
    value: 'v1_htf_ltf_sweep_fvg',
    label: 'v1 HTF→LTF Sweep FVG',
    description: 'Scanner-only alerts. Does not place trades.',
  },
];
const selectedChartScanner = ref('off');
const selectedChartScannerLabel = computed(
  () =>
    chartScannerOptions.find((scanner) => scanner.value === selectedChartScanner.value)?.label ??
    'Scanner off',
);
const liveStrategyLabel = computed(
  () => liveStrategyProfile.value?.label || strategyLabel.value || 'Strategy loading',
);
const liveStrategyTimeframe = computed(
  () => liveStrategyProfile.value?.observed_timeframe || botStore.activeBot.timeframe || '5m',
);
const chartTimeframeMismatch = computed(
  () => !!liveStrategyTimeframe.value && chartTimeframe.value !== liveStrategyTimeframe.value,
);
const tradeChartHeightStorageKey = 'ftTradeChartHeight';
const sidePanelStorageKey = 'ftTradeSidePanelOpen';
const tradeChartHeight = ref(430);
const tradeChartResizeStart = ref<{ y: number; height: number } | null>(null);
const sidePanelOpen = ref(true);
const isMobileViewport = ref(false);

function clampTradeChartHeight(height: number) {
  return Math.min(1000, Math.max(220, Math.round(height)));
}

function refreshOHLCV(pair: string, columns: string[]) {
  botStore.activeBot.getPairCandles({
    pair: pair,
    timeframe: chartTimeframe.value,
    columns: columns,
  });
}

async function fetchLiveStrategyProfile() {
  try {
    const res = await fetch(`${API_BASE}/api/v1/live-strategy-profile`);
    if (!res.ok) return;
    const data = await res.json();
    liveStrategyProfile.value = data;
    if (!selectedLiveStrategy.value || selectedLiveStrategy.value !== data.research_profile) {
      selectedLiveStrategy.value = data.research_profile;
    }
    if (Number.isFinite(Number(data.leverage))) {
      selectedLeverage.value = Number(data.leverage);
    }
  } catch (error) {
    console.error('Failed to fetch live strategy profile', error);
  }
}

async function setLiveStrategyProfile(profile: string) {
  if (!profile || profile === liveStrategyProfile.value?.research_profile) return;

  isSavingLiveStrategy.value = true;
  liveStrategyStatus.value = 'Applying...';
  try {
    const res = await fetch(`${API_BASE}/api/v1/live-strategy-profile`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ research_profile: profile }),
    });
    const data = await res.json();
    if (!res.ok || data.ok === false) {
      throw new Error(data.error || `Failed with status ${res.status}`);
    }
    liveStrategyProfile.value = data;
    selectedLiveStrategy.value = data.research_profile;
    liveStrategyStatus.value = 'Applied';
    window.setTimeout(() => {
      if (liveStrategyStatus.value === 'Applied') liveStrategyStatus.value = '';
    }, 2500);
  } catch (error) {
    liveStrategyStatus.value = error instanceof Error ? error.message : 'Apply failed';
    selectedLiveStrategy.value = liveStrategyProfile.value?.research_profile || selectedLiveStrategy.value;
  } finally {
    isSavingLiveStrategy.value = false;
  }
}

async function setLiveLeverage(leverage: number) {
  const nextLeverage = Number(leverage);
  if (!Number.isFinite(nextLeverage) || nextLeverage < 1) return;

  isSavingLeverage.value = true;
  liveStrategyStatus.value = 'Applying leverage...';
  try {
    const res = await fetch(`${API_BASE}/api/v1/live-trade-controls`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        research_profile: selectedLiveStrategy.value || liveStrategyProfile.value?.research_profile,
        leverage: nextLeverage,
      }),
    });
    const data = await res.json();
    if (!res.ok || data.ok === false) {
      throw new Error(data.error || `Failed with status ${res.status}`);
    }
    liveStrategyProfile.value = data;
    selectedLiveStrategy.value = data.research_profile;
    selectedLeverage.value = Number(data.leverage);
    liveStrategyStatus.value = `Leverage ${selectedLeverage.value}x`;
    window.setTimeout(() => {
      if (liveStrategyStatus.value === `Leverage ${selectedLeverage.value}x`) liveStrategyStatus.value = '';
    }, 2500);
  } catch (error) {
    liveStrategyStatus.value = error instanceof Error ? error.message : 'Leverage apply failed';
    selectedLeverage.value = Number(liveStrategyProfile.value?.leverage || selectedLeverage.value);
  } finally {
    isSavingLeverage.value = false;
  }
}

function startTradeChartResize(event: PointerEvent) {
  tradeChartResizeStart.value = {
    y: event.clientY,
    height: tradeChartHeight.value,
  };
  window.addEventListener('pointermove', resizeTradeChart);
  window.addEventListener('pointerup', stopTradeChartResize, { once: true });
}

function resizeTradeChart(event: PointerEvent) {
  if (!tradeChartResizeStart.value) return;

  tradeChartHeight.value = clampTradeChartHeight(
    tradeChartResizeStart.value.height + event.clientY - tradeChartResizeStart.value.y,
  );
}

function stopTradeChartResize() {
  if (!tradeChartResizeStart.value) return;

  tradeChartResizeStart.value = null;
  window.removeEventListener('pointermove', resizeTradeChart);
  localStorage.setItem(tradeChartHeightStorageKey, String(tradeChartHeight.value));
}

function selectDefaultTrade() {
  if (botStore.activeBot.detailTradeId || !hasOpenTrades.value) return;

  const [firstOpenTrade] = botStore.activeBot.openTrades;
  if (firstOpenTrade) {
    botStore.activeBot.setDetailTrade(firstOpenTrade);
  }
}

watch(
  () => botStore.activeBot.openTrades,
  () => selectDefaultTrade(),
  { deep: true, immediate: true },
);

function syncViewportState() {
  isMobileViewport.value = window.innerWidth < 768;
}

onMounted(() => {
  syncViewportState();
  window.addEventListener('resize', syncViewportState);
  fetchLiveStrategyProfile();
  liveStrategyPollInterval = window.setInterval(fetchLiveStrategyProfile, 5000);

  const storedTradeChartHeight = Number(localStorage.getItem(tradeChartHeightStorageKey));
  if (Number.isFinite(storedTradeChartHeight) && storedTradeChartHeight > 0) {
    tradeChartHeight.value = clampTradeChartHeight(storedTradeChartHeight);
  }

  const storedSidePanelOpen = localStorage.getItem(sidePanelStorageKey);
  const storedChartScanner = localStorage.getItem(chartScannerStorageKey);
  if (storedChartScanner && chartScannerOptions.some((scanner) => scanner.value === storedChartScanner)) {
    selectedChartScanner.value = storedChartScanner;
  }

  if (isMobileViewport.value) {
    sidePanelOpen.value = false;
  } else if (storedSidePanelOpen !== null) {
    sidePanelOpen.value = storedSidePanelOpen === '1';
  }
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', syncViewportState);
  if (liveStrategyPollInterval) {
    window.clearInterval(liveStrategyPollInterval);
  }
});

function setSidePanelOpen(value: boolean) {
  sidePanelOpen.value = value;
  localStorage.setItem(sidePanelStorageKey, value ? '1' : '0');
}

function setChartScanner(scanner: string) {
  selectedChartScanner.value = scanner;
  localStorage.setItem(chartScannerStorageKey, scanner);
}
</script>

<template>
  <div class="trade-page flex min-h-full w-full flex-col gap-3 p-3">
    <section class="trade-command-center">
      <div class="trade-command-main">
        <div>
          <div class="trade-kicker">Live Trading Desk</div>
          <h1>{{ selectedPairsLabel }}</h1>
        </div>
        <div class="trade-command-meta">
          <span class="desk-chip" :class="botStore.activeBot.isBotOnline ? 'positive' : 'neutral'">
            {{ botStatusLabel }}
          </span>
          <span class="desk-chip">{{ runModeLabel }}</span>
          <span class="desk-chip">TF {{ chartTimeframe }}</span>
          <span class="desk-chip">Open {{ botStore.activeBot.openTrades.length }}</span>
          <span class="desk-chip">Closed {{ botStore.activeBot.closedTrades.length }}</span>
        </div>
      </div>
      <div class="trade-command-side">
        <div class="trade-control-bar">
          <label class="live-strategy-control strategy-select">
            <span>Strategy</span>
            <select
              v-model="selectedLiveStrategy"
              :disabled="isSavingLiveStrategy || !liveStrategyProfile?.available?.length"
              @change="setLiveStrategyProfile(selectedLiveStrategy)"
            >
              <option
                v-for="strategy in liveStrategyProfile?.available ?? []"
                :key="strategy.value"
                :value="strategy.value"
              >
                {{ strategy.label }}
              </option>
            </select>
          </label>
          <label class="live-strategy-control scanner-select">
            <span>Scanner</span>
            <select
              v-model="selectedChartScanner"
              @change="setChartScanner(selectedChartScanner)"
            >
              <option
                v-for="scanner in chartScannerOptions"
                :key="scanner.value"
                :value="scanner.value"
              >
                {{ scanner.label }}
              </option>
            </select>
          </label>
          <label class="live-strategy-control leverage-select">
            <span>Leverage</span>
            <select
              v-model.number="selectedLeverage"
              :disabled="isSavingLeverage"
              @change="setLiveLeverage(selectedLeverage)"
            >
              <option
                v-for="leverage in liveStrategyProfile?.leverage_options ?? [1, 2, 3, 4, 5, 6, 8, 10]"
                :key="leverage"
                :value="leverage"
              >
                {{ leverage }}x
              </option>
            </select>
          </label>
          <button
            type="button"
            class="desk-action"
            :aria-pressed="sidePanelOpen"
            @click="setSidePanelOpen(!sidePanelOpen)"
          >
            <i-mdi-view-dashboard-outline />
            <span>{{ sidePanelOpen ? 'Hide Panels' : 'Show Panels' }}</span>
          </button>
        </div>
        <div class="trade-command-status">
          <span class="trade-strategy-label">
            Active: {{ liveStrategyLabel }}
            <span v-if="liveStrategyStatus"> · {{ liveStrategyStatus }}</span>
          </span>
          <span class="trade-strategy-label">Leverage {{ selectedLeverage }}x</span>
          <span class="trade-strategy-label">Scanner {{ selectedChartScannerLabel }}</span>
        </div>
        <div v-if="chartTimeframeMismatch" class="trade-timeframe-warning">
          Chart {{ chartTimeframe }} · bot trades {{ liveStrategyTimeframe }}
        </div>
      </div>
    </section>

    <div class="trade-workspace flex min-h-0 w-full flex-1 gap-3">
      <aside class="mix-panel" :class="{ closed: !sidePanelOpen }">
      <button
        v-if="!sidePanelOpen"
        type="button"
        class="mix-panel-tab"
        title="Open Multi Pane"
        @click="setSidePanelOpen(true)"
      >
        <i-mdi-chevron-right />
      </button>

      <DraggableContainer v-else header="Multi Pane" class="h-full" content-overflow="auto">
        <template #header>
          <div class="mix-panel-header">
            <span>Multi Pane</span>
            <button
              type="button"
              class="mix-panel-toggle"
              title="Close Multi Pane"
              @click.stop="setSidePanelOpen(false)"
            >
              <i-mdi-chevron-left />
            </button>
          </div>
        </template>

        <div class="mt-1 flex justify-center">
          <BotControls class="mt-1 mb-2" />
        </div>

        <Tabs value="0" scrollable lazy>
          <TabList>
            <Tab value="0" severity="secondary">
              <div title="Pairs combined">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">
                  Pairs combined
                </span>
                <i-mdi-view-list v-else />
              </div>
            </Tab>

            <Tab value="1" severity="secondary">
              <div title="General">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">General</span>
                <i-mdi-information v-else />
              </div>
            </Tab>

            <Tab value="2" severity="secondary">
              <div title="Performance">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">
                  Performance
                </span>
                <i-mdi-chart-line v-else />
              </div>
            </Tab>

            <Tab value="3" severity="secondary">
              <div title="Setups">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">Setups</span>
                <i-mdi-radar v-else />
              </div>
            </Tab>

            <Tab value="4" severity="secondary">
              <div title="Balance">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">Balance</span>
                <i-mdi-bank v-else />
              </div>
            </Tab>

            <Tab value="5" severity="secondary">
              <div title="Time Breakdown">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">
                  Time Breakdown
                </span>
                <i-mdi-folder-clock v-else />
              </div>
            </Tab>

            <Tab value="6" severity="secondary">
              <div title="Pairlist">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">Pairlist</span>
                <i-mdi-format-list-group v-else />
              </div>
            </Tab>

            <Tab value="7" severity="secondary">
              <div title="Pair Locks">
                <span v-if="settingsStore.multiPaneButtonsShowText" class="ms-1">Pair Locks</span>
                <i-mdi-lock-alert v-else />
              </div>
            </Tab>
          </TabList>

          <TabPanels>
            <TabPanel value="0">
              <PairSummary
                :pairlist="botStore.activeBot.whitelist"
                :current-locks="botStore.activeBot.activeLocks"
                :trades="botStore.activeBot.openTrades"
              />
            </TabPanel>

            <TabPanel value="1">
              <BotStatus />
            </TabPanel>

            <TabPanel value="2" lazy>
              <BotPerformance />
            </TabPanel>

            <TabPanel value="3" lazy>
              <TradeReadinessPanel :pairlist="botStore.activeBot.whitelist" />
            </TabPanel>

            <TabPanel value="4" lazy>
              <BotBalance />
            </TabPanel>

            <TabPanel value="5" lazy>
              <PeriodBreakdown />
            </TabPanel>

            <TabPanel value="6" lazy>
              <PairListLive />
            </TabPanel>

            <TabPanel value="7" lazy>
              <PairLockList />
            </TabPanel>
          </TabPanels>
        </Tabs>
      </DraggableContainer>
      </aside>

      <div class="trade-main flex min-w-0 flex-1 flex-col gap-2">
      <div
        class="trade-section trade-section-chart"
        :style="{ flexBasis: `${tradeChartHeight}px` }"
      >
        <DraggableContainer header="Chart" class="h-full" content-overflow="hidden">
          <div class="trade-chart-content">
            <div class="chart-timeframe-bar">
              <button
                v-for="timeframe in chartTimeframes"
                :key="timeframe"
                type="button"
                class="timeframe-button"
                :class="{ active: chartTimeframe === timeframe }"
                @click="chartStore.selectedTimeframe = timeframe"
              >
                {{ timeframe }}
              </button>
            </div>
            <div class="trade-chart-body">
              <CandleChartContainer
                :available-pairs="botStore.activeBot.whitelist"
                :historic-view="!!false"
                :timeframe="chartTimeframe"
                :trades="botStore.activeBot.allTrades"
                :scanner-mode="selectedChartScanner"
                @refresh-data="refreshOHLCV"
              />
            </div>
            <button
              class="trade-chart-height-handle"
              type="button"
              title="Drag to resize chart height"
              @pointerdown.prevent="startTradeChartResize"
            >
              <span></span>
            </button>
          </div>
        </DraggableContainer>
      </div>

      <div class="trade-section trade-section-open">
        <DraggableContainer header="Open Trades" content-overflow="visible">
          <TradeList
            v-if="hasOpenTrades"
            class="open-trades"
            :trades="botStore.activeBot.openTrades"
            title="Open trades"
            :active-trades="true"
            :page-scroll="true"
            empty-text="Currently no open trades."
          />
          <div v-else class="empty-section">Currently no open trades.</div>
        </DraggableContainer>
      </div>

      <div class="trade-section trade-section-detail">
        <DraggableContainer header="Trade Detail" content-overflow="visible">
          <TradeDetail
            v-if="selectedTrade"
            :trade="selectedTrade"
            :stake-currency="botStore.activeBot.stakeCurrency"
          />
          <div v-else class="empty-section">Select a trade to show details.</div>
        </DraggableContainer>
      </div>

      <div class="trade-section trade-section-closed">
        <DraggableContainer header="Closed Trades" content-overflow="visible">
          <TradeList
            v-if="hasClosedTrades"
            class="trade-history"
            :trades="botStore.activeBot.closedTrades"
            title="Trade history"
            :show-filter="true"
            :page-scroll="true"
            empty-text="No closed trades so far."
          />
          <div v-else class="empty-section">No closed trades so far.</div>
        </DraggableContainer>
      </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.trade-page {
  background:
    linear-gradient(180deg, rgba(15, 23, 42, 0.72), rgba(2, 6, 23, 0.2)),
    #020617;
}

.trade-command-center {
  display: flex;
  align-items: stretch;
  justify-content: space-between;
  gap: 14px;
  padding: 14px;
  border: 1px solid rgba(51, 65, 85, 0.82);
  border-radius: 12px;
  background:
    linear-gradient(135deg, rgba(15, 23, 42, 0.98), rgba(8, 13, 23, 0.96)),
    #0b111a;
  box-shadow: 0 18px 48px rgba(0, 0, 0, 0.24);
}

.trade-command-main {
  display: flex;
  min-width: 0;
  flex: 1 1 auto;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.trade-kicker {
  color: var(--p-primary-300);
  font-size: 0.68rem;
  font-weight: 800;
  letter-spacing: 0.18em;
  text-transform: uppercase;
}

.trade-command-main h1 {
  margin: 2px 0 0;
  color: var(--p-surface-50);
  font-size: clamp(1.15rem, 2vw, 1.65rem);
  font-weight: 760;
  letter-spacing: 0;
}

.trade-command-meta {
  display: flex;
  flex-wrap: wrap;
  justify-content: flex-end;
  gap: 7px;
}

.desk-chip {
  display: inline-flex;
  min-height: 28px;
  align-items: center;
  border: 1px solid rgba(71, 85, 105, 0.78);
  border-radius: 999px;
  padding: 0 10px;
  background: rgba(15, 23, 42, 0.86);
  color: var(--p-surface-300);
  font-size: 0.78rem;
  font-weight: 700;
  text-transform: capitalize;
  white-space: nowrap;
}

.desk-chip.positive {
  border-color: rgba(20, 184, 166, 0.45);
  background: rgba(20, 184, 166, 0.12);
  color: #5eead4;
}

.desk-chip.neutral {
  color: var(--p-surface-400);
}

.trade-command-side {
  display: flex;
  min-width: min(46rem, 58%);
  flex-direction: column;
  align-items: flex-end;
  justify-content: center;
  gap: 6px;
  text-align: right;
}

.trade-control-bar {
  display: grid;
  width: 100%;
  grid-template-columns: minmax(190px, 1.25fr) minmax(170px, 1fr) minmax(92px, 0.42fr) auto;
  align-items: end;
  gap: 8px;
}

.trade-command-status {
  display: flex;
  max-width: 100%;
  flex-wrap: wrap;
  justify-content: flex-end;
  gap: 6px 12px;
}

.trade-strategy-label {
  max-width: 100%;
  overflow: hidden;
  color: var(--p-surface-400);
  font-size: 0.74rem;
  font-weight: 650;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.live-strategy-control {
  display: grid;
  min-width: 0;
  width: 100%;
  gap: 4px;
  color: var(--p-surface-400);
  font-size: 0.68rem;
  font-weight: 800;
  letter-spacing: 0.12em;
  text-align: left;
  text-transform: uppercase;
}

.live-strategy-control.leverage-select {
  min-width: 92px;
}

.live-strategy-control select {
  min-height: 34px;
  max-width: 100%;
  border: 1px solid rgba(71, 85, 105, 0.85);
  border-radius: 8px;
  padding: 0 10px;
  background: rgba(2, 6, 23, 0.72);
  color: var(--p-surface-100);
  font-size: 0.82rem;
  font-weight: 700;
  letter-spacing: 0;
  text-transform: none;
}

.live-strategy-control select:disabled {
  cursor: wait;
  opacity: 0.65;
}

.trade-timeframe-warning {
  max-width: 100%;
  border: 1px solid rgba(245, 158, 11, 0.36);
  border-radius: 6px;
  padding: 4px 8px;
  background: rgba(245, 158, 11, 0.1);
  color: #fbbf24;
  font-size: 0.74rem;
  font-weight: 700;
  text-align: left;
}

.desk-action {
  display: inline-flex;
  min-height: 34px;
  align-items: center;
  align-self: end;
  gap: 8px;
  border: 1px solid rgba(0, 137, 161, 0.52);
  border-radius: 8px;
  padding: 0 12px;
  background: rgba(0, 137, 161, 0.13);
  color: var(--p-primary-100);
  font-size: 0.82rem;
  font-weight: 760;
  cursor: pointer;
  white-space: nowrap;
}

.desk-action:hover {
  border-color: rgba(46, 224, 255, 0.72);
  background: rgba(0, 137, 161, 0.22);
}

@media (max-width: 1120px) {
  .trade-command-center {
    flex-direction: column;
  }

  .trade-command-side {
    min-width: 0;
    align-items: stretch;
    text-align: left;
  }

  .trade-command-status {
    justify-content: flex-start;
  }
}

@media (min-width: 768px) and (max-width: 980px) {
  .trade-control-bar {
    grid-template-columns: minmax(220px, 1fr) minmax(180px, 0.8fr) minmax(92px, 0.36fr) auto;
  }
}

.trade-workspace {
  min-height: 0;
}

.mix-panel {
  position: relative;
  flex: 0 0 316px;
  min-width: 316px;
  transition:
    flex-basis 0.16s ease,
    min-width 0.16s ease;
}

.mix-panel.closed {
  flex-basis: 42px;
  min-width: 42px;
}

.mix-panel-tab {
  position: sticky;
  top: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 96px;
  border: 1px solid rgba(51, 65, 85, 0.9);
  border-radius: 10px;
  background: rgba(15, 23, 42, 0.95);
  color: var(--p-surface-200);
  cursor: pointer;
}

.mix-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.mix-panel-toggle {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border: 0;
  border-radius: 4px;
  background: transparent;
  color: var(--p-surface-300);
  cursor: pointer;
}

.mix-panel-toggle:hover,
.mix-panel-tab:hover {
  background: var(--p-surface-800);
  color: var(--p-surface-50);
}

.trade-section {
  position: relative;
  min-height: 0;
}

.trade-section-chart {
  flex: 0 0 auto;
}

.trade-section-open {
  flex: 0 0 auto;
  min-height: 40px;
}

.trade-section-detail {
  flex: 0 0 auto;
  min-height: 40px;
}

.trade-section-closed {
  flex: 0 0 auto;
  min-height: 40px;
}

.empty-section {
  display: flex;
  min-height: 32px;
  align-items: center;
  justify-content: center;
  color: var(--p-surface-500);
  font-size: 0.875rem;
}

.chart-timeframe-bar {
  flex: 0 0 auto;
  display: flex;
  align-items: center;
  gap: 4px;
  height: 36px;
  min-width: 0;
  overflow-x: auto;
  overflow-y: hidden;
  padding: 5px 10px;
  border-bottom: 1px solid rgba(30, 41, 59, 0.95);
  background: rgba(2, 6, 23, 0.34);
  scrollbar-width: thin;
}

.timeframe-button {
  flex: 0 0 auto;
  min-width: 34px;
  height: 25px;
  border: 0;
  border-radius: 6px;
  background: transparent;
  color: var(--p-surface-400);
  font-size: 0.78rem;
  font-weight: 700;
  line-height: 1;
  cursor: pointer;
}

.timeframe-button:hover {
  background: var(--p-surface-800);
  color: var(--p-surface-100);
}

.timeframe-button.active {
  background: linear-gradient(180deg, var(--p-primary-500), var(--p-primary-700));
  color: var(--p-primary-contrast-color);
}

@media (max-width: 640px) {
  .chart-timeframe-bar {
    height: auto;
    max-height: 64px;
    flex-wrap: wrap;
    overflow-y: auto;
  }
}

.trade-chart-content {
  position: relative;
  display: flex;
  height: 100%;
  min-height: 0;
  flex-direction: column;
}

.trade-chart-body {
  min-height: 0;
  flex: 1 1 auto;
  overflow: hidden;
}

.trade-chart-body :deep(> .flex) {
  height: 100%;
  min-height: 0;
}

.trade-chart-body :deep(.flex-fill) {
  min-height: 0;
}

.trade-chart-body :deep(.tradingview-chart-shell) {
  height: 100%;
}

.trade-chart-height-handle {
  position: absolute;
  left: 50%;
  bottom: 1px;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 34px;
  height: 8px;
  border: 1px solid var(--p-surface-700);
  border-radius: 2px;
  background: color-mix(in srgb, var(--p-surface-900) 92%, transparent);
  cursor: ns-resize;
  opacity: 0.74;
  transform: translateX(-50%);
}

.trade-chart-height-handle:hover {
  opacity: 1;
}

.trade-chart-height-handle span,
.trade-chart-height-handle span::before,
.trade-chart-height-handle span::after {
  display: block;
  width: 18px;
  height: 1px;
  background: var(--p-surface-400);
  content: '';
}

.trade-chart-height-handle span::before {
  transform: translateY(-3px);
}

.trade-chart-height-handle span::after {
  transform: translateY(2px);
}

@media (max-width: 767px) {
  .trade-page {
    gap: 8px;
    padding: 8px;
  }

  .trade-command-center {
    flex-direction: column;
    gap: 10px;
    padding: 12px;
  }

  .trade-command-main {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }

  .trade-command-meta {
    width: 100%;
    flex-wrap: nowrap;
    justify-content: flex-start;
    overflow-x: auto;
    padding-bottom: 2px;
  }

  .trade-command-side {
    width: 100%;
    min-width: 0;
    align-items: stretch;
    text-align: left;
  }

  .trade-control-bar {
    display: flex;
    align-items: end;
    gap: 8px;
    overflow-x: auto;
    padding-bottom: 2px;
    scrollbar-width: thin;
  }

  .live-strategy-control.strategy-select,
  .live-strategy-control.scanner-select {
    grid-column: auto;
    flex: 0 0 230px;
  }

  .live-strategy-control.scanner-select {
    flex-basis: 210px;
  }

  .live-strategy-control.leverage-select {
    flex: 0 0 92px;
  }

  .trade-command-status {
    flex-wrap: nowrap;
    overflow-x: auto;
    padding-bottom: 1px;
    scrollbar-width: none;
  }

  .trade-command-status {
    justify-content: flex-start;
  }

  .desk-action {
    flex: 0 0 auto;
    justify-content: center;
  }

  .trade-workspace {
    flex-direction: column;
    gap: 8px;
  }

  .mix-panel {
    flex: 0 0 auto;
    min-width: 0;
    width: 100%;
    max-height: 42vh;
    overflow: hidden;
  }

  .mix-panel.closed {
    flex-basis: 32px;
    min-width: 0;
    height: 32px;
  }

  .mix-panel-tab {
    position: relative;
    top: 0;
    width: 100%;
    height: 30px;
    border-radius: 8px;
  }

  .trade-main {
    width: 100%;
  }

  .trade-section-chart {
    flex-basis: clamp(420px, 62vh, 680px) !important;
  }

  .trade-chart-height-handle {
    display: none;
  }
}
</style>
