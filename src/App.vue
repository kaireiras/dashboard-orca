<script setup>
import { ref, computed } from 'vue'
import axios from 'axios'
import * as echarts from 'echarts'
import VChart from 'vue-echarts'

// Hubungkan keempat grafik dalam satu grup sinkronisasi ('attitudeGroup')
echarts.connect('attitudeGroup')

const isLoading = ref(false)
const fileInfo = ref(null)
const errorMsg = ref('')

// Konfigurasi opsi untuk 4 grafik
const combinedOption = ref({})
const rollOption = ref({})
const pitchOption = ref({})
const yawOption = ref({})
const depthOption = ref({})
const pressureOption = ref({})
const controlOption = ref({})
const modeOption = ref({})

// Tab per jenis pesan; tab Orca nonaktif kalau topic-nya tidak ada di .mcap
const tab = ref('attitude')
const tabs = computed(() => [
  { id: 'attitude', label: 'Attitude', ok: true },
  { id: 'depth', label: 'OrcaDepth', ok: !!depthOption.value.series },
  { id: 'control', label: 'OrcaControl', ok: !!controlOption.value.series }
])
const orcaCharts = computed(() => ({
  depth: [depthOption.value, pressureOption.value],
  control: [controlOption.value, modeOption.value]
})[tab.value] ?? [])

// Fungsi pembantu pembuat template opsi ECharts
const createChartOption = (title, timeData, seriesList, showSlider = false, yName = 'Derajat (°)') => ({
  backgroundColor: '#18181b',
  title: {
    text: title,
    left: '15px',
    top: '8px',
    textStyle: { color: '#e4e4e7', fontSize: 13, fontWeight: '600' }
  },
  tooltip: {
    trigger: 'axis',
    axisPointer: { type: 'cross' },
    backgroundColor: 'rgba(24, 24, 27, 0.9)',
    borderColor: '#3f3f46',
    textStyle: { color: '#f4f4f5', fontSize: 12 }
  },
  legend: {
    top: '8px',
    right: '20px',
    textStyle: { color: '#a1a1aa' }
  },
  grid: {
    left: '55px',
    right: '25px',
    top: '40px',
    bottom: showSlider ? '65px' : '28px'
  },
  xAxis: {
    type: 'value',
    min: 0,
    max: fileInfo.value.duration_sec,
    axisLabel: { color: '#a1a1aa', formatter: '{value} s' },
    axisLine: { lineStyle: { color: '#3f3f46' } }
  },
  yAxis: {
    type: 'value',
    name: yName,
    nameTextStyle: { color: '#a1a1aa' },
    axisLabel: { color: '#a1a1aa' },
    splitLine: { lineStyle: { color: '#27272a' } }
  },
  dataZoom: [
    { type: 'inside', xAxisIndex: 0 },
    ...(showSlider
      ? [{ type: 'slider', xAxisIndex: 0, bottom: 10, height: 22, textStyle: { color: '#a1a1aa' } }]
      : [])
  ],
  series: seriesList.map(s => ({ ...s, data: s.data.map((v, i) => [timeData[i], v]) }))
})

// Pembuat series garis untuk grafik baru
const line = (name, data, color) => ({ name, type: 'line', data, showSymbol: false, itemStyle: { color }, lineStyle: { width: 1.5 } })

// Fungsi saat pengguna memilih file .mcap
const handleFileUpload = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  isLoading.value = true
  errorMsg.value = ''

  const formData = new FormData()
  formData.append('file', file)

  try {
    // Kirim file ke server FastAPI di port 8000
    const res = await axios.post('http://127.0.0.1:8000/api/upload-mcap', formData, {
      headers: { 'Content-Type': 'multipart/form-data' }
    })

    const { filename, message_count, duration_sec, series, depth, control } = res.data
    fileInfo.value = { filename, message_count, duration_sec }

    // 1. Grafik Gabungan (Roll, Pitch, Yaw)
    combinedOption.value = createChartOption(
      '1. Perbandingan Attitude Gabungan (/sensor/attitude)',
      series.time,
      [
        { name: 'Roll (deg)', type: 'line', data: series.roll, showSymbol: false, itemStyle: { color: '#3b82f6' }, lineStyle: { width: 1.5 } },
        { name: 'Pitch (deg)', type: 'line', data: series.pitch, showSymbol: false, itemStyle: { color: '#f97316' }, lineStyle: { width: 1.5 } },
        { name: 'Yaw (deg)', type: 'line', data: series.yaw, showSymbol: false, itemStyle: { color: '#22c55e' }, lineStyle: { width: 1.5 } }
      ]
    )

    // 2. Grafik Terpisah: Roll
    rollOption.value = createChartOption(
      '2. Roll (deg)',
      series.time,
      [{ name: 'Roll', type: 'line', data: series.roll, showSymbol: false, itemStyle: { color: '#3b82f6' }, lineStyle: { width: 1.5 } }]
    )

    // 3. Grafik Terpisah: Pitch
    pitchOption.value = createChartOption(
      '3. Pitch (deg)',
      series.time,
      [{ name: 'Pitch', type: 'line', data: series.pitch, showSymbol: false, itemStyle: { color: '#f97316' }, lineStyle: { width: 1.5 } }]
    )

    // 4. Grafik Terpisah: Yaw (Dilengkapi Slider Waktu di Bawah)
    yawOption.value = createChartOption(
      '4. Yaw (deg)',
      series.time,
      [{ name: 'Yaw', type: 'line', data: series.yaw, showSymbol: false, itemStyle: { color: '#22c55e' }, lineStyle: { width: 1.5 } }],
      true
    )

    depthOption.value = pressureOption.value = controlOption.value = modeOption.value = {}

    // 5-6. OrcaDepth
    if (depth.time.length) {
      depthOption.value = createChartOption('5. Depth', depth.time, [line('Depth', depth.depth, '#06b6d4')], false, 'Depth')
      pressureOption.value = createChartOption('6. Pressure', depth.time, [line('Pressure', depth.pressure, '#e879f9')], false, 'Pressure')
    }

    // 7-8. OrcaControl: perintah numerik + mode (string, digambar sebagai garis step)
    if (control.time.length) {
      const cmd = ['throttle', 'lateral', 'forward', 'yaw']
      const cmdColor = ['#3b82f6', '#f97316', '#22c55e', '#eab308']
      controlOption.value = createChartOption('7. Control Command (OrcaControl)', control.time,
        cmd.map((k, i) => line(k, control[k], cmdColor[i])), false, 'Command')

      const modes = ['arm_mode', 'control_mode', 'gripper_mode', 'source']
      const modeColor = ['#a78bfa', '#facc15', '#f472b6', '#2dd4bf']
      const mo = createChartOption('8. Mode & Source (OrcaControl)', control.time,
        modes.map((k, i) => ({ ...line(k, control[k], modeColor[i]), step: 'end' })), false, '')
      mo.yAxis = { ...mo.yAxis, type: 'category', data: [...new Set(modes.flatMap(k => control[k]))] }
      modeOption.value = mo
    }
    tab.value = series.time.length ? 'attitude' : depth.time.length ? 'depth' : 'control'
  } catch (err) {
    errorMsg.value = err.response?.data?.detail || 'Gagal terhubung ke server FastAPI (Pastikan backend menyala di port 8000)'
  } finally {
    isLoading.value = false
  }
}
</script>

<template>
  <div class="dashboard">
    <header class="topbar">
      <div>
        <h1>ROS 2 MCAP Attitude Viewer</h1>
        <p v-if="fileInfo" class="meta">
          File: <strong>{{ fileInfo.filename }}</strong> | 
          Pesan: <strong>{{ fileInfo.message_count }}</strong> | 
          Durasi: <strong>{{ fileInfo.duration_sec }}s</strong>
        </p>
      </div>

      <label class="upload-btn">
        <span>{{ isLoading ? 'Memproses MCAP...' : 'Upload File .mcap' }}</span>
        <input type="file" accept=".mcap" @change="handleFileUpload" :disabled="isLoading" hidden />
      </label>
    </header>

    <div v-if="errorMsg" class="error-box">{{ errorMsg }}</div>

    <nav v-if="fileInfo" class="tabs" role="tablist">
      <button v-for="t in tabs" :key="t.id" role="tab" class="tab" :class="{ active: tab === t.id }"
        :aria-selected="tab === t.id" :disabled="!t.ok" @click="tab = t.id">{{ t.label }}</button>
    </nav>

    <!-- Tampilan Subplot Vertikal Tersinkronisasi, per tab -->
    <main v-if="fileInfo" class="charts-grid">
      <template v-if="tab === 'attitude'">
        <div class="chart-card main-chart">
          <v-chart :option="combinedOption" group="attitudeGroup" autoresize />
        </div>
        <div class="chart-card sub-chart">
          <v-chart :option="rollOption" group="attitudeGroup" autoresize />
        </div>
        <div class="chart-card sub-chart">
          <v-chart :option="pitchOption" group="attitudeGroup" autoresize />
        </div>
        <div class="chart-card sub-chart-bottom">
          <v-chart :option="yawOption" group="attitudeGroup" autoresize />
        </div>
      </template>
      <div v-for="(o, i) in orcaCharts" :key="tab + i" class="chart-card sub-chart">
        <v-chart :option="o" group="attitudeGroup" autoresize />
      </div>
    </main>

    <div v-else class="empty-state">
      <p>Silakan unggah file <code>.mcap</code> untuk menampilkan grafik perbandingan dan subplot Roll, Pitch, Yaw.</p>
    </div>
  </div>
</template>


<style>
*,
*::before,
*::after {
  box-sizing: border-box;
}

html,
body {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  background-color: #09090b;
}

#app {
  width: 100%;
  min-height: 100vh;
  margin: 0;
  padding: 0;
  display: block;
  max-width: none;
  text-align: left;
}

.dashboard {
  display: flex;
  flex-direction: column;
  width: 100%;
  max-width: none;
  min-height: 100vh;
  min-height: 100dvh;
  margin: 0;
  padding: 16px 24px;
  background-color: #09090b;
  color: #f4f4f5;
  font-family: system-ui, -apple-system, sans-serif;
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
  width: 100%;
  padding: 14px 20px;
  margin-bottom: 16px;
  background: #18181b;
  border: 1px solid #27272a;
  border-radius: 10px;
}

.topbar h1 {
  margin: 0;
  font-size: 18px;
}

.meta {
  margin: 4px 0 0;
  font-size: 13px;
  color: #a1a1aa;
  overflow-wrap: anywhere;
}

.upload-btn {
  flex-shrink: 0;
  padding: 9px 16px;
  background: #2563eb;
  color: white;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.upload-btn:hover {
  background: #1d4ed8;
}

.error-box {
  width: 100%;
  padding: 12px;
  margin-bottom: 16px;
  background: #450a0a;
  border: 1px solid #dc2626;
  border-radius: 8px;
  color: #fca5a5;
  overflow-wrap: anywhere;
}

.tabs {
  display: flex;
  gap: 4px;
  margin-bottom: 12px;
  border-bottom: 1px solid #27272a;
}

.tab {
  flex-shrink: 0;
  padding: 8px 18px;
  margin-bottom: -1px;
  background: transparent;
  border: 1px solid transparent;
  border-radius: 8px 8px 0 0;
  color: #a1a1aa;
  font: inherit;
  font-size: 14px;
  cursor: pointer;
}

.tab:hover:not(:disabled) {
  background: #18181b;
  color: #f4f4f5;
}

.tab.active {
  background: #18181b;
  border-color: #27272a #27272a #18181b;
  color: #f4f4f5;
  font-weight: 600;
}

.tab:disabled {
  color: #52525b;
  cursor: not-allowed;
}

.charts-grid {
  display: flex;
  flex-direction: column;
  gap: 12px;
  width: 100%;
  max-width: none;
  min-width: 0;
}

.chart-card {
  width: 100%;
  min-width: 0;
  overflow: hidden;
  background: #18181b;
  border: 1px solid #27272a;
  border-radius: 8px;
}

.chart-card > x-vue-echarts,
.chart-card > div,
.chart-card canvas,
.chart-card .echarts {
  width: 100% !important;
  height: 100% !important;
  max-width: 100%;
}

.main-chart {
  height: 320px;
}

.sub-chart {
  height: 260px;
}

.sub-chart-bottom {
  height: 300px;
}

.empty-state {
  width: 100%;
  padding: 80px 20px;
  text-align: center;
  color: #71717a;
  border: 2px dashed #27272a;
  border-radius: 10px;
}

@media (max-width: 640px) {
  .dashboard {
    padding: 10px;
  }

  .topbar {
    flex-direction: column;
    align-items: stretch;
    gap: 14px;
    padding: 14px;
  }

  .upload-btn {
    width: 100%;
    text-align: center;
  }
}
</style>