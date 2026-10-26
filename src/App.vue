<script setup>
import { ref } from 'vue'
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

// Fungsi pembantu pembuat template opsi ECharts
const createChartOption = (title, timeData, seriesList, showSlider = false) => ({
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
    type: 'category',
    boundaryGap: false,
    data: timeData,
    axisLabel: { color: '#a1a1aa', formatter: '{value} s' },
    axisLine: { lineStyle: { color: '#3f3f46' } }
  },
  yAxis: {
    type: 'value',
    name: 'Derajat (°)',
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
  series: seriesList
})

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

    const { filename, message_count, duration_sec, series } = res.data
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

    <!-- Tampilan 4 Subplot Vertikal Tersinkronisasi -->
    <main v-if="fileInfo" class="charts-grid">
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
