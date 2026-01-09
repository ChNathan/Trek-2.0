<template>
  <div class="app-container">

    <!-- PAGE 1 : BLE CONNECTION -->
    <div v-if="currentPage === 'ble'">
      <h1 class="title">TREK 2.0 : Bilateral Gait Monitoring</h1>

      <div class="cards-container">
        <div class="card">
          <h2 class="card-title">Nicla 1</h2>
          <p class="status">{{ connected1 ? "Connected" : "Waiting..." }}</p>
          <button
            @click="connectToBLE1"
            :disabled="connected1"
            class="connect-btn"
          >
            {{ connected1 ? "Connected" : "Connect Nicla 1" }}
          </button>
        </div>

        <div class="card">
          <h2 class="card-title">Nicla 2</h2>
          <p class="status">{{ connected2 ? "Connected" : "Waiting..." }}</p>
          <button
            @click="connectToBLE2"
            :disabled="connected2"
            class="connect-btn cyan"
          >
            {{ connected2 ? "Connected" : "Connect Nicla 2" }}
          </button>
        </div>
      </div>

      <div v-if="!connected1 || !connected2" class="waiting-message">
        Waiting for both cards connection...
      </div>

      <button class="navigate-btn" @click="currentPage='dashboard'">
        Go to Dashboard
      </button>
    </div>

    <!--  PAGE 2 : DASHBOARD  -->
    <div v-else-if="currentPage === 'dashboard'">
      <h1 class="title">Dashboard</h1>

      <button class="navigate-btn" @click="currentPage='ble'">
        Back to BLE Page
      </button>

      <div class="mode-buttons">
        <button :class="{ active: mode==='minimal' }" @click="activateMinimal">
          Minimal Mode
        </button>
        <button :class="{ active: mode==='debugged' }" @click="activateDebugged">
          Debugged Mode
        </button>
      </div>

      <!--  MINIMAL MODE  -->
      <div v-if="mode==='minimal'" class="chart-container">

        <div class="time-frames">
          <button
            v-for="time in minimalTimes"
            :key="time.value"
            :class="['time-btn', { 'active-time': selectedTimeFrame === time.value }]"
            @click="selectTimeFrame(time.value)"
          >
            {{ time.label }}
          </button>
        </div>

        <!--  General metrics block  -->
        <div class="block general-block">
          <div class="mean-strides">
            <div class="mean-card">
              <h3>Mean Stride L</h3>
              <p>{{ meanStrideL.toFixed(2) }}</p>
            </div>

            <div class="mean-card">
              <h3>Mean Stride R</h3>
              <p>{{ meanStrideR.toFixed(2) }}</p>
            </div>
          </div>

          <div class="mean-strides">
            <div class="mean-card">
              <h3>Nb of Steps</h3>
              <p>{{ nbSteps }}</p>
            </div>
            <div class="mean-card">
              <h3>Steps / Min</h3>
              <p>{{ stepsPerMin.toFixed(2) }}</p>
            </div>
          </div>

          <div class="stride-chart-container">
            <h3>Stride Length</h3>
            <canvas id="strideChartMinimal" height="150"></canvas>
          </div>
        </div>

        <!-- visual separator -->
        <div class="separator"></div>

        <!-- ===== Symmetry metrics block ===== -->
        <div class="block symmetry-block">
          <div class="mean-strides">
            <div class="mean-card">
              <h3>RI</h3>
              <p>{{ RI.toFixed(2) }}</p>
            </div>
            <div class="mean-card">
              <h3>SI</h3>
              <p>{{ SI.toFixed(2) }}%</p>
            </div>
            <div class="mean-card nsi-card">
              <h3>NSI</h3>
              <p>{{ currentNSI.toFixed(2) }}</p>
            </div>
          </div>

          <div class="nsi-chart-container">
            <h3>NSI over time</h3>
            <canvas id="nsiChartMinimal" height="150"></canvas>
          </div>
        </div>

      </div>

      <!--  DEBUGGED MODE  -->
      <div v-else-if="mode==='debugged'" class="chart-container">

        <div class="time-frames">
          <button
            v-for="time in debuggedTimes"
            :key="time.value"
            :class="['time-btn', { 'active-time': selectedTimeFrame === time.value }]"
            @click="selectTimeFrame(time.value)"
          >
            {{ time.label }}
          </button>
        </div>

        <div class="debug-controls">
          <button class="navigate-btn" @click="sendReset">Reset</button>
        </div>

        <!--  General metrics block (debug)  -->
        <div class="block general-block">
          <div class="mean-strides">
            <div class="mean-card">
              <h3>Mean Stride L</h3>
              <p>{{ meanStrideL.toFixed(2) }}</p>
            </div>

            <div class="mean-card">
              <h3>Mean Stride R</h3>
              <p>{{ meanStrideR.toFixed(2) }}</p>
            </div>
          </div>

          <div class="mean-strides">
            <div class="mean-card">
              <h3>Nb of Steps</h3>
              <p>{{ nbSteps }}</p>
            </div>
            <div class="mean-card">
              <h3>Steps / Min</h3>
              <p>{{ stepsPerMin.toFixed(2) }}</p>
            </div>
          </div>

          <div class="stride-chart-container">
            <h3>Stride Length (debug)</h3>
            <canvas id="strlgtChart" height="200"></canvas>
          </div>
        </div>

        <div class="separator"></div>

        <!--  Symmetry metrics block (debug)  -->
        <div class="block symmetry-block">
          <div class="mean-strides">
            <div class="mean-card">
              <h3>RI</h3>
              <p>{{ RI.toFixed(2) }}</p>
            </div>
            <div class="mean-card">
              <h3>SI</h3>
              <p>{{ SI.toFixed(2) }}%</p>
            </div>
            <div class="mean-card nsi-card">
              <h3>NSI</h3>
              <p>{{ currentNSI.toFixed(2) }}</p>
            </div>
          </div>

          <div class="nsi-chart-container">
            <h3>NSI over time (debug)</h3>
            <canvas id="nsiChartDebug" height="150"></canvas>
          </div>
        </div>

      </div>

    </div>

  </div>
</template>

<script setup>
import { ref, nextTick, watch, computed } from "vue";
import {
  Chart,
  LineController,
  LineElement,
  PointElement,
  LinearScale,
  Title,
  CategoryScale,
  Legend
} from "chart.js";

Chart.register(LineController, LineElement, PointElement, LinearScale, Title, CategoryScale, Legend);

/* BLE UUIDs */
const SERVICE_UUID = "12345678-1234-1234-1234-123456781234";
const CHARACTERISTIC_UUID = "12345679-1234-1234-1234-123456781234";
const CMD_CHARACTERISTIC_UUID = "12345671-1234-1234-1234-123456781234";

/* BLE refs */
const deviceRef1 = ref(null), deviceRef2 = ref(null);
const serverRef1 = ref(null), serverRef2 = ref(null);
const connected1 = ref(false), connected2 = ref(false);
const writeChar1 = ref(null), writeChar2 = ref(null);
const cmdChar1 = ref(null), cmdChar2 = ref(null);

/* UI & data */
const currentPage = ref("ble");
const mode = ref("");
const selectedTimeFrame = ref(300);
const meanStrideL = ref(0), meanStrideR = ref(0);
const nbSteps = ref(0), stepsPerMin = ref(0);
const denom_EMA = ref(0), EMA_ALPHA = 0.2;
const RI = ref(0), SI = ref(0);

const minimalTimes = [{ value: 60, label: "1m" }, { value: 300, label: "5m" }, { value: 600, label: "10m" }, { value: 1800, label: "30m" }, { value: -1, label: "all" }];
const debuggedTimes = [{ value: 300, label: "5m" }, { value: 900, label: "15m" }, { value: 1800, label: "30m" }, { value: 3600, label: "1h" }, { value: -1, label: "all" }];

const maxPoints = 100;
let strideChartMinimal = null;
let nsiChartMinimal = null;
let chart = null;
let nsiChartDebug = null;

const strlgtData = { x1: [], x2: [], t: [] };
const NSI_store = ref([]);
const dataStore1 = ref([]), dataStore2 = ref([]);

/* Time frame */
function selectTimeFrame(time) {
  selectedTimeFrame.value = time;
  updateMetrics();
}

/* Send commands */
function sendIntToBothCards(value) {
  const buffer = new Uint8Array([value]);
  [
    { c: cmdChar1.value, name: "NiclaBLE1" },
    { c: cmdChar2.value, name: "NiclaBLE2" }
  ].forEach(dev => {
    if (!dev.c) return console.warn(`${dev.name} cmdChar non défini`);
    dev.c.writeValue(buffer).catch(err => console.error(`Erreur écriture ${dev.name}:`, err));
  });
}

function sendReset() { sendIntToBothCards(2); }

/* Watch for entering dashboard */
watch(currentPage, async (newPage) => {
  if (newPage === "dashboard") {
    mode.value = "minimal";
    await nextTick();
    sendIntToBothCards(0);
    createMinimalCharts();
  }
});

/* Chart creation helpers */
function safeDestroyChart(c) {
  try { if (c && c.destroy) c.destroy(); } catch(e){ /* ignore */ }
}

/* Minimal mode chart creation */
async function createMinimalCharts() {
  await nextTick();

  // destroy previous minimal charts if present
  safeDestroyChart(strideChartMinimal);
  safeDestroyChart(nsiChartMinimal);

  const ctxStride = document.getElementById("strideChartMinimal");
  const ctxNSI = document.getElementById("nsiChartMinimal");
  if (ctxStride) {
    strideChartMinimal = new Chart(ctxStride, {
      type: "line",
      data: {
        labels: strlgtData.t.map(ts => new Date(ts).toLocaleTimeString()),
        datasets: [
          { label: "Nicla 1 Stride", data: strlgtData.x1, borderColor: "#ff6384", fill: false, tension: 0.2, pointRadius: 1 },
          { label: "Nicla 2 Stride", data: strlgtData.x2, borderColor: "#8a2be2", fill: false, tension: 0.2, pointRadius: 1 }
        ]
      },
      options: { responsive: true, animation: false, scales: { x: { display: false }, y: { beginAtZero: false } }, plugins: { legend: { position: "bottom" } } }
    });
  }
  if (ctxNSI) {
    nsiChartMinimal = new Chart(ctxNSI, {
      type: "line",
      data: { labels: NSI_store.value.map(d => new Date(d.time).toLocaleTimeString()), datasets: [{ label: "NSI", data: NSI_store.value.map(d => d.value), borderColor: "#00ff00", fill: false, tension: 0.2, pointRadius: 1 }] },
      options:{responsive:true,animation:false,scales:{x:{display:false},y:{min:-1,max:1}},plugins:{legend:{position:"bottom"}}}
    });
  }
}

/* Debugged mode chart creation */
async function createDebuggedCharts() {
  await nextTick();

  // destroy previous debug charts
  safeDestroyChart(chart);
  safeDestroyChart(nsiChartDebug);

  const ctx = document.getElementById("strlgtChart");
  const ctxNSI = document.getElementById("nsiChartDebug");

  if (ctx) {
    chart = new Chart(ctx, {
      type: "line",
      data: {
        labels: strlgtData.t.map(ts => new Date(ts).toLocaleTimeString()),
        datasets: [
          { label: "Nicla 1 Stride", data: strlgtData.x1, borderColor: "#ff6384", fill: false, tension: 0.15, pointRadius: 2 },
          { label: "Nicla 2 Stride", data: strlgtData.x2, borderColor: "#8a2be2", fill: false, tension: 0.15, pointRadius: 2 }
        ]
      },
      options: {
        responsive: true,
        animation: false,
        scales: { x: { display: true }, y: { beginAtZero: false } },
        plugins: { legend: { position: "bottom" } }
      }
    });
  }

  if (ctxNSI) {
    nsiChartDebug = new Chart(ctxNSI, {
      type: "line",
      data: { labels: NSI_store.value.map(d => new Date(d.time).toLocaleTimeString()), datasets: [{ label: "NSI", data: NSI_store.value.map(d => d.value), borderColor: "#00ff00", fill: false, tension: 0.2, pointRadius: 1 }] },
      options:{responsive:true,animation:false,scales:{x:{display:true},y:{min:-1,max:1}},plugins:{legend:{position:"bottom"}}}
    });
  }
}

/* Mode switch */
function activateMinimal() {
  mode.value = "minimal";
  sendIntToBothCards(0);
  createMinimalCharts();
}
function activateDebugged() {
  mode.value = "debugged";
  sendIntToBothCards(1);
  createDebuggedCharts();
}

/* Metrics */
function updateMetrics() {
  const now = Date.now();
  const interval = selectedTimeFrame.value > 0 ? selectedTimeFrame.value * 1000 : Infinity;

  const recent1 = dataStore1.value.filter(d => now - d.time <= interval);
  const recent2 = dataStore2.value.filter(d => now - d.time <= interval);

  meanStrideL.value = recent1.length ? recent1.reduce((s, d) => s + d.strideLength, 0)/recent1.length : 0;
  meanStrideR.value = recent2.length ? recent2.reduce((s, d) => s + d.strideLength, 0)/recent2.length : 0;

  // RI: ratio (guard division by zero)
  RI.value = meanStrideR.value !== 0 ? (1- meanStrideL.value / meanStrideR.value)*100 : 0;

  // SI: percent symmetry index: ((2*(L-R))/(L+R))*100
  SI.value = (meanStrideL.value + meanStrideR.value) !== 0 ? ((2 * (meanStrideL.value - meanStrideR.value)) / (meanStrideL.value + meanStrideR.value)) * 100 : 0;

  let NSI_current = 0;
  if (recent1.length && recent2.length) {
    const X1 = recent1.map(d => d.strideLength);
    const X2 = recent2.map(d => d.strideLength);
    const denom_current = Math.max(...X1, ...X2) - Math.min(...X1, ...X2);
    denom_EMA.value = EMA_ALPHA * denom_current + (1 - EMA_ALPHA) * denom_EMA.value;
    // guard denom_EMA
    NSI_current = denom_EMA.value !== 0 ? (meanStrideL.value - meanStrideR.value)/denom_EMA.value : 0;
  }
  NSI_store.value.push({ time: now, value: NSI_current });

  nbSteps.value = recent1.length + recent2.length;
  stepsPerMin.value = interval < Infinity ? (nbSteps.value / (interval / 1000))*60 : 0;

  // Update minimal charts (labels/data arrays are references to strlgtData / NSI_store)
  if (strideChartMinimal) {
    strideChartMinimal.data.labels = strlgtData.t.map(ts => new Date(ts).toLocaleTimeString());
    strideChartMinimal.data.datasets[0].data = strlgtData.x1;
    strideChartMinimal.data.datasets[1].data = strlgtData.x2;
    strideChartMinimal.update("none");
  }
  if (nsiChartMinimal) {
    nsiChartMinimal.data.labels = NSI_store.value.map(d => new Date(d.time).toLocaleTimeString());
    nsiChartMinimal.data.datasets[0].data = NSI_store.value.map(d => d.value);
    nsiChartMinimal.update("none");
  }

  // Update debugged charts
  if (chart && mode.value === "debugged") {
    chart.data.labels = strlgtData.t.map(ts => new Date(ts).toLocaleTimeString());
    chart.data.datasets[0].data = strlgtData.x1;
    chart.data.datasets[1].data = strlgtData.x2;
    const vals = [...strlgtData.x1, ...strlgtData.x2];
    if (vals.length) { chart.options.scales.y.min = Math.min(...vals)-0.1; chart.options.scales.y.max = Math.max(...vals)+0.1; }
    chart.update("none");
  }
  if (nsiChartDebug && mode.value === "debugged") {
    nsiChartDebug.data.labels = NSI_store.value.map(d => new Date(d.time).toLocaleTimeString());
    nsiChartDebug.data.datasets[0].data = NSI_store.value.map(d => d.value);
    nsiChartDebug.update("none");
  }
}

const currentNSI = computed(() => NSI_store.value.length ? NSI_store.value[NSI_store.value.length-1].value : 0);

/* BLE Connection */
async function connectToBLE(deviceName, isFirst) {
  if (!navigator.bluetooth) { alert("Web Bluetooth not supported"); return; }
  try {
    const dev = await navigator.bluetooth.requestDevice({ filters:[{name:deviceName}], optionalServices:[SERVICE_UUID] });
    if (isFirst) deviceRef1.value=dev; else deviceRef2.value=dev;

    const server = await dev.gatt.connect();
    if (isFirst) serverRef1.value=server; else serverRef2.value=server;

    dev.addEventListener("gattserverdisconnected", ()=>autoReconnect(deviceName,isFirst));

    const service = await server.getPrimaryService(SERVICE_UUID);
    const characteristic = await service.getCharacteristic(CHARACTERISTIC_UUID);
    const cmdCharacteristic = await service.getCharacteristic(CMD_CHARACTERISTIC_UUID);

    if (isFirst) { connected1.value=true; writeChar1.value=characteristic; cmdChar1.value=cmdCharacteristic; }
    else { connected2.value=true; writeChar2.value=characteristic; cmdChar2.value=cmdCharacteristic; }

    await characteristic.startNotifications();
    characteristic.addEventListener("characteristicvaluechanged", e=>{
      const dv = e.target.value; if(dv.byteLength<8)return;
      const strideLength=dv.getFloat32(0,true), strideTime=dv.getFloat32(4,true);
      const now=Date.now();
      const store = isFirst?dataStore1.value:dataStore2.value;
      store.push({time:now,strideLength,strideTime});
      if(store.length>maxPoints)store.shift();

      // update shared strlgtData for plotting
      const arr = isFirst?strlgtData.x1:strlgtData.x2;
      arr.push(strideLength);
      // keep times aligned (use push timestamp)
      strlgtData.t.push(now);
      // trim to maxPoints
      if (strlgtData.x1.length > maxPoints) {
        ["x1","x2","t"].forEach(k => { if(strlgtData[k].length>maxPoints) strlgtData[k].shift(); });
      }

      // update debug chart scales if needed
      if(chart && mode.value==="debugged"){
        const vals=[...strlgtData.x1,...strlgtData.x2];
        if(vals.length){ chart.options.scales.y.min=Math.min(...vals)-0.1; chart.options.scales.y.max=Math.max(...vals)+0.1; }
        chart.update("none");
      }

      updateMetrics();
    });

    // small init command write after connect
    setTimeout(()=>{ if(isFirst) cmdChar1.value?.writeValue(new Uint8Array([0])); else cmdChar2.value?.writeValue(new Uint8Array([0])); },200);

  } catch(err){ if(err.name!=="NotFoundError") console.error(`Erreur BLE ${deviceName}:`,err); }
}

function connectToBLE1(){ connectToBLE("NiclaBLE1",true); }
function connectToBLE2(){ connectToBLE("NiclaBLE2",false); }

/* Auto reconnect */
async function autoReconnect(deviceName,isFirst){
  let retries=0, maxRetries=5;
  while(retries<maxRetries){
    try{ console.log(`Reconnect ${deviceName} (${retries+1}/${maxRetries})`); await connectToBLE(deviceName,isFirst); return; }
    catch(err){ retries++; await new Promise(r=>setTimeout(r,3000)); }
  }
}

</script>

<style scoped>
.app-container { min-height:100vh; background: linear-gradient(to bottom right,#1e3a8a,#3b82f6,#06b6d4); display:flex; flex-direction:column; align-items:center; padding:2rem; color:white; }
.title { font-size:2.5rem; font-weight:800; margin-bottom:2rem; }
.cards-container { display:flex; flex-direction:column; gap:1.5rem; width:100%; max-width:900px; }
@media(min-width:768px){ .cards-container{ flex-direction:row; } }
.card { flex:1; background: rgba(255,255,255,0.15); border-radius:1rem; padding:1.5rem; }
.card-title { font-size:1.25rem; font-weight:600; }
.status { margin-top:0.5rem; font-style:italic; opacity:0.9; }
.connect-btn { margin-top:1rem; width:100%; background:linear-gradient(to right,#3b82f6,#2563eb); padding:0.5rem 1rem; border-radius:0.5rem; border:none; color:white; cursor:pointer; }
.connect-btn.cyan { background:linear-gradient(to right,#06b6d4,#0ea5e9); }
.waiting-message { margin-top:1.5rem; padding:0.75rem 1rem; background: rgba(255,255,255,0.15); border-radius:0.5rem; border:1px solid rgba(255,255,255,0.25); }
.navigate-btn { margin-top:1.5rem; padding:0.5rem 1rem; background:#36a2eb; border:none; border-radius:0.5rem; color:white; cursor:pointer; }
.mode-buttons { display:flex; gap:1rem; margin-top:1.5rem; margin-bottom:1rem; }
.mode-buttons button { padding:0.5rem 1rem; border:none; border-radius:0.5rem; cursor:pointer; background:#2563eb; color:white; }
.mode-buttons button.active { background:#0ea5e9; }
.chart-container { margin-top:2rem; width:100%; max-width:900px; background: rgba(255,255,255,0.08); padding:1.5rem; border-radius:1rem; text-align:center; }

/* Blocks */
.block { padding: 0.5rem 0; }
.general-block { }
.symmetry-block { }

/* separator */
.separator { height:1px; width:90%; background: rgba(255,255,255,0.12); margin: 1rem auto; border-radius:2px; }

/* Titles */
.chart-title, .stride-chart-container h3, .nsi-chart-container h3 { text-align:center; font-size:1.2rem; margin-bottom:0.8rem; }
canvas { max-width:100%; }

/* Time frames */
.time-frames { display:flex; flex-wrap:wrap; gap:0.5rem; justify-content:center; margin-bottom:1rem; }
.time-btn { padding:0.4rem 0.8rem; border-radius:0.4rem; border:none; background:#2563eb; color:white; cursor:pointer; transition:0.2s; }
.time-btn:hover { background:#1e40af; }
.time-btn.active-time { background:#0ea5e9; color:white; font-weight:600; }

/* Metric cards */
.mean-strides { display:flex; justify-content:center; gap:1rem; flex-wrap:wrap; margin-bottom:1rem; }
.mean-card { background: rgba(255,255,255,0.18); padding:0.6rem 0.8rem; border-radius:0.6rem; min-width: 120px; text-align:center; }
.mean-card h3 { margin-bottom:0.15rem; font-size:0.75rem; font-weight:600; }
.mean-card p { font-size:1rem; font-weight:700; }
.nsi-card { min-width:150px; }

/* layout tweaks for debug */
.debug-controls { display:flex; justify-content:center; gap:1rem; margin-bottom:1rem; }

@media(min-width:900px){
  .mean-strides { gap:2rem; }
}

.dashboard-container {
  display: flex;
  flex-direction: column; 
  justify-content: center; 
  align-items: center; 
  min-height: 100vh; 
  text-align: center;
}

</style>
