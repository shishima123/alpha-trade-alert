<template>
  <div class="min-h-screen bg-gray-900 p-6 text-gray-200">
    <!-- Header -->
    <div class="flex items-center justify-between mb-6 gap-4 flex-wrap">
      <h1 class="text-2xl font-bold flex items-center gap-2">📊 Kiểm tra độ ổn định</h1>
      <div class="flex items-center gap-4">
        <button
          class="px-3 py-1 bg-gray-700 rounded-lg hover:bg-gray-600 transition"
          @click="showSettings = true"
        >
          ⚙️ Cài đặt
        </button>
        <div class="flex items-center gap-2">
          <input type="checkbox" id="enableSound" v-model="soundEnabled" class="w-4 h-4" />
          <label for="enableSound">🔊 Bật âm thanh</label>
        </div>
      </div>
    </div>

    <div v-if="loading" class="text-gray-400 text-center py-10">
      ⏳ Đang tải dữ liệu từ Binance...
    </div>

    <div v-else class="grid gap-6 grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
      <div
        v-for="(symbol, index) in sortedSymbols"
        :key="symbol"
        class="coin-card bg-gray-800 rounded-2xl shadow-md border border-gray-700 transition-all duration-300 p-5"
        :class="[
          coinStatus[symbol] === 'valid' ? 'ring-4 ring-green-400 shadow-xl scale-[1.02]' : '',
          flashClass[symbol],
        ]"
      >
        <div class="flex items-center justify-between mb-3">
          <div>
            <h3 class="font-semibold text-lg">{{ symbol }}</h3>

            <span class="text-xs text-blue-400 block mb-3">
              (x4: {{ getDaysLeft(symbol) }} ngày)
            </span>
          </div>

          <div class="flex flex-col items-end">
            <span
              class="text-sm font-medium px-2 py-1 rounded-full"
              :class="{
                'bg-green-700 text-green-100': coinStatus[symbol] === 'valid',
                'bg-red-700 text-red-100': coinStatus[symbol] === 'invalid',
              }"
            >
              {{ coinStatus[symbol] === 'valid' ? 'Ổn định' : 'Không ổn định' }}
            </span>

            <span v-if="coinStatus[symbol] === 'valid'" class="text-xs text-gray-400 mt-1">
              Đã ổn định: {{ Math.floor((now - stableSince[symbol]) / 1000) }}s
            </span>
          </div>
        </div>

        <div class="custom-scroll overflow-y-auto max-h-[310px]">
          <table class="min-w-full text-sm">
            <thead>
              <tr class="border-b border-gray-700">
                <th class="text-left py-1 font-medium w-[15%]">#</th>
                <th class="text-left py-1 font-medium w-[55%]">Giá</th>
                <th class="text-left py-1 font-medium w-[30%]">Δ</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(item, idx) in priceTracker[symbol]"
                :key="idx"
                class="border-b last:border-0 border-gray-700"
              >
                <td class="py-1">{{ idx + 1 }}</td>
                <td class="py-1 font-mono">{{ item.price.toFixed(8) }}</td>
                <td
                  class="py-1"
                  :class="
                    item.diff !== null
                      ? item.diff > 0
                        ? 'text-green-400'
                        : item.diff < 0
                          ? 'text-red-400'
                          : 'text-gray-400'
                      : ''
                  "
                >
                  {{ item.diff !== null ? item.diff.toFixed(2) : '—' }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- ⚙️ Modal Setting -->
    <div
      v-if="showSettings"
      class="fixed inset-0 bg-black bg-opacity-60 flex items-center justify-center z-50"
    >
      <div class="bg-gray-800 rounded-2xl p-6 w-full max-w-md shadow-lg">
        <h2 class="text-xl font-semibold mb-4 text-center">⚙️ Cài đặt</h2>

        <div class="space-y-4">
          <div>
            <label class="block text-sm mb-1">📈 Số bản ghi tối đa</label>
            <input
              type="number"
              v-model.number="settings.MAX_RECORD"
              class="w-full bg-gray-700 rounded p-2"
            />
          </div>

          <div>
            <label class="block text-sm mb-1">💥 Độ chênh lệch cho phép (×1e8)</label>
            <input
              type="number"
              v-model.number="settings.priceThreshold"
              class="w-full bg-gray-700 rounded p-2"
            />
          </div>

          <div>
            <label class="block text-sm mb-1">⚠ Giới hạn biến động 1/2 đầu</label>
            <input
              type="number"
              v-model.number="settings.halfInvalidLimit"
              class="w-full bg-gray-700 rounded p-2"
            />
          </div>

          <div>
            <label class="block text-sm mb-1">⚠ Tổng giới hạn biến động</label>
            <input
              type="number"
              v-model.number="settings.totalInvalidLimit"
              class="w-full bg-gray-700 rounded p-2"
            />
          </div>

          <div>
            <label class="block text-sm mb-1">⏱️ Thời gian ổn định tối thiểu (ms)</label>
            <input
              type="number"
              v-model.number="settings.stableTime"
              class="w-full bg-gray-700 rounded p-2"
            />
          </div>
        </div>

        <div class="flex justify-end gap-2 mt-6">
          <button
            class="px-4 py-2 bg-yellow-600 rounded-lg hover:bg-yellow-500"
            @click="resetSettings"
          >
            🔄 Reset mặc định
          </button>
          <button
            class="px-4 py-2 bg-gray-700 rounded-lg hover:bg-gray-600"
            @click="showSettings = false"
          >
            Đóng
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import axios from 'axios'
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue'
import { useStorage } from '@vueuse/core'
import { toast } from 'vue3-toastify'
import 'vue3-toastify/dist/index.css'
import validSound from '@/assets/notification.mp3'
import invalidSound from '@/assets/alert.mp3'

interface PriceEntry {
  price: number
  diff: number | null
}

// =================== ⚙️ SETTINGS ===================
const settings = useStorage('stability-settings', {
  MAX_RECORD: 20,
  priceThreshold: 10,
  stableTime: 10000,
  halfInvalidLimit: 2, // NEW
  totalInvalidLimit: 3, // NEW
})

const defaultSettings = {
  MAX_RECORD: 20,
  priceThreshold: 10,
  stableTime: 10000,
  halfInvalidLimit: 2,
  totalInvalidLimit: 3,
}

// Bổ sung các field thiếu trong localStorage
settings.value = {
  ...defaultSettings,
  ...settings.value,
}

const showSettings = ref(false)
const soundEnabled = ref(false)
const loading = ref(true)
const now = ref(Date.now())

// =================== 📊 CORE STATE ===================
const priceTracker = reactive<Record<string, PriceEntry[]>>({})
const coinStatus = reactive<Record<string, 'valid' | 'invalid'>>({})
const flashClass = reactive<Record<string, string>>({})
const stableSince = reactive<Record<string, number>>({})
const wasValid = reactive<Record<string, boolean>>({})
const alphaToSymbol: Record<string, string> = {}
const listingTimes: Record<string, number> = {}

const BINANCE_STREAM = 'wss://nbstream.binance.com/w3w/wsa/stream'
const API_TOP =
  'https://www.binance.com/bapi/defi/v1/public/alpha-trade/aggTicker24?dataType=aggregate'

let ws: WebSocket | null = null
let clockInterval: number | null = null
const debounceTimers: Record<string, number> = {}

// =================== ⏱️ UPDATE CLOCK ===================
clockInterval = window.setInterval(() => (now.value = Date.now()), 1000)

// =================== 🔊 SOUND ===================
const audioMap = {
  valid: new Audio(validSound),
  invalid: new Audio(invalidSound),
}
function playSound(type: 'valid' | 'invalid') {
  if (!soundEnabled.value) return
  const audio = audioMap[type]
  audio.currentTime = 0
  audio.play().catch(() => {})
}

// STATUS ONLY: VALID or INVALID (dựa theo diff + limit)
function getStatus(diffs: Array<number | null>): 'valid' | 'invalid' {
  const threshold = settings.value.priceThreshold
  const halfLimit = settings.value.halfInvalidLimit
  const totalLimit = settings.value.totalInvalidLimit
  const maxRecord = settings.value.MAX_RECORD

  // Chưa đủ dữ liệu thì coi như valid
  if (diffs.length < 2) return 'valid'

  const half = Math.floor(maxRecord / 2)

  let firstHalfInvalid = 0
  let totalInvalid = 0

  for (let i = 0; i < diffs.length; i++) {
    const d = diffs[i]
    if (d === null) continue

    if (Math.abs(d) > threshold) {
      totalInvalid++
      if (i < half) firstHalfInvalid++
    }
  }

  if (firstHalfInvalid > halfLimit || totalInvalid > totalLimit) {
    return 'invalid'
  }

  return 'valid'
}

// ⏳ Tính ngày còn lại x4
function getDaysLeft(symbol: string): number | null {
  const t = listingTimes[symbol]
  if (!t) return null

  const end = t + 30 * 24 * 60 * 60 * 1000
  const utcNow = Date.now() - new Date().getTimezoneOffset() * 60 * 1000

  // Dùng Math.ceil để còn vài giờ vẫn đếm 1 ngày
  const remaining = Math.ceil((end - utcNow) / (24 * 60 * 60 * 1000))

  return remaining < 0 ? 0 : remaining
}

// UI effects
function flash(symbol: string, type: 'valid' | 'invalid') {
  flashClass[symbol] = type === 'valid' ? 'flash-green' : 'flash-red'
  setTimeout(() => (flashClass[symbol] = ''), 300)
}
function notify(symbol: string, type: 'valid' | 'invalid') {
  toast[type === 'valid' ? 'success' : 'error'](
    `${symbol} ${type === 'valid' ? 'ổn định ✅' : 'mất ổn định ❌'}`,
    { position: 'bottom-right', autoClose: 1500 },
  )
}

// =================== 🧭 INVALID ORDER STATE ===================
const invalidOrder = ref<string[]>([])

function pushFrontUnique(arr: string[], s: string) {
  const i = arr.indexOf(s)
  if (i !== -1) arr.splice(i, 1)
  arr.unshift(s)
}
function removeItem(arr: string[], s: string) {
  const i = arr.indexOf(s)
  if (i !== -1) arr.splice(i, 1)
}

// =================== 🧠 STATUS SORTING ===================
const sortedSymbols = computed(() => {
  const valid: string[] = []

  for (const s in coinStatus) {
    if (coinStatus[s] === 'valid') valid.push(s)
  }

  return [...valid, ...invalidOrder.value]
})

// =================== 📈 FETCH INIT DATA ===================
async function getTopCoins() {
  const { data } = await axios.get(API_TOP)
  return (data?.data ?? [])
    .filter((c: any) => c.mulPoint === 4)
    .sort((a: any, b: any) => b.volume24h - a.volume24h)
    .slice(0, 10)
    .map((c: any) => ({
      symbol: c.symbol,
      alphaId: c.alphaId.toUpperCase(),
      listingTime: c.listingTime,
    }))
}

async function collectInitialData(symbol: string, alphaId: string) {
  try {
    const url = `https://www.binance.com/bapi/defi/v1/public/alpha-trade/agg-trades?limit=${settings.value.MAX_RECORD}&symbol=${alphaId}USDT`
    const { data } = await axios.get(url)
    const trades = (data?.data ?? []).map((t: any) => parseFloat(t.p))
    const entries: PriceEntry[] = trades.map((p, i) => ({
      price: p,
      diff: i === 0 ? null : (p - trades[i - 1]) * 1e8,
    }))
    priceTracker[symbol] = entries

    const diffs = entries.map((e) => e.diff)
    const st = getStatus(diffs)
    coinStatus[symbol] = st
    stableSince[symbol] = st === 'valid' ? Date.now() : 0
  } catch (err) {
    console.error('collectInitialData error', symbol, err)
  }
}

// =================== ♻️ UPDATE EACH TICK ===================
function handlePriceUpdate(symbol: string, price: number) {
  const arr = priceTracker[symbol]
  const prev = arr[0]?.price
  const diff = prev != null ? (price - prev) * 1e8 : null

  arr.unshift({ price, diff })
  if (arr.length > settings.value.MAX_RECORD) arr.length = settings.value.MAX_RECORD

  const diffs = arr.map((x) => x.diff)

  // getStatus giờ đã dùng threshold + halfInvalidLimit + totalInvalidLimit
  const newStatus = getStatus(diffs)
  const oldStatus = coinStatus[symbol]

  // ❗ CHẶN MỌI CASE KHÔNG ĐỔI TRẠNG THÁI
  if (newStatus === oldStatus) return

  // ============================================================
  // 🟢 VALID LOGIC
  // ============================================================
  if (newStatus === 'valid') {
    const t = Date.now()
    if (!stableSince[symbol]) stableSince[symbol] = t

    if (t - stableSince[symbol] >= settings.value.stableTime) {
      if (oldStatus !== 'valid') {
        // 🔍 Kiểm tra xem hiện tại đã có coin nào khác đang valid chưa
        const anyOtherValid = Object.entries(coinStatus).some(
          ([s, st]) => s !== symbol && st === 'valid',
        )

        // 🔔 Chỉ phát âm khi ĐÂY là coin valid đầu tiên trong hệ thống
        if (!anyOtherValid) {
          playSound('valid')
        }

        coinStatus[symbol] = 'valid'
        removeItem(invalidOrder.value, symbol)

        wasValid[symbol] = true
        flash(symbol, 'valid')
        notify(symbol, 'valid')
      }
    }
    return
  }

  // ============================================================
  // 🔴 INVALID LOGIC
  // ============================================================
  if (oldStatus !== 'invalid') {
    coinStatus[symbol] = 'invalid'
    stableSince[symbol] = 0
    pushFrontUnique(invalidOrder.value, symbol)

    wasValid[symbol] = false
    flash(symbol, 'invalid')
    notify(symbol, 'invalid')
    playSound('invalid')
  }
}

function resetSettings() {
  settings.value = { ...defaultSettings }
  toast.info('⚙️ Đã khôi phục cài đặt mặc định', { position: 'bottom-right' })
}

// INIT
onMounted(async () => {
  const topCoins = await getTopCoins()

  for (const c of topCoins) {
    priceTracker[c.symbol] = []
    coinStatus[c.symbol] = 'invalid'
    flashClass[c.symbol] = ''
    stableSince[c.symbol] = 0
    wasValid[c.symbol] = false
    alphaToSymbol[`${c.alphaId}USDT`] = c.symbol
    listingTimes[c.symbol] = c.listingTime // ⏱️ gán thời gian list
  }

  await Promise.all(topCoins.map((c) => collectInitialData(c.symbol, c.alphaId)))

  // Điền invalidOrder theo thứ tự hiện có (giữ nguyên thứ tự ban đầu)
  invalidOrder.value = topCoins.map((c) => c.symbol).filter((s) => coinStatus[s] === 'invalid')

  loading.value = false

  ws = new WebSocket(BINANCE_STREAM)
  ws.onopen = () => {
    const params = topCoins.map((c) => `${c.alphaId.toLowerCase()}usdt@aggTrade`)
    ws?.send(JSON.stringify({ method: 'SUBSCRIBE', params, id: 1 }))
  }

  ws.onmessage = (e) => {
    const res = JSON.parse(e.data)
    const d = res?.data
    if (!d || d.e !== 'aggTrade') return
    const symbol = alphaToSymbol[d.s]
    if (!symbol) return
    const price = parseFloat(d.p)

    clearTimeout(debounceTimers[symbol])
    debounceTimers[symbol] = window.setTimeout(() => {
      handlePriceUpdate(symbol, price)
    }, 200)
  }
})

// =================== 🧹 CLEANUP ===================
onBeforeUnmount(() => {
  if (ws) {
    ws.close()
    ws = null
  }
  if (clockInterval) clearInterval(clockInterval)
  Object.values(debounceTimers).forEach((t) => clearTimeout(t))
})
</script>

<style scoped>
/* 💫 Flash khi đổi trạng thái */
@keyframes smoothValid {
  0% {
    transform: scale(1);
    box-shadow: 0 0 0 rgba(34, 197, 94, 0);
  }
  40% {
    transform: scale(1.04);
    box-shadow: 0 0 20px rgba(34, 197, 94, 0.6);
  }
  100% {
    transform: scale(1);
    box-shadow: none;
  }
}
@keyframes smoothInvalid {
  0% {
    transform: scale(1);
    box-shadow: 0 0 0 rgba(239, 68, 68, 0);
  }
  40% {
    transform: scale(1.04);
    box-shadow: 0 0 20px rgba(239, 68, 68, 0.6);
  }
  100% {
    transform: scale(1);
    box-shadow: none;
  }
}

.flash-green {
  animation: smoothValid 0.8s ease-out;
}
.flash-red {
  animation: smoothInvalid 0.8s ease-out;
}

.coin-card {
  border-radius: 1rem;
  padding: 1.25rem;
  transition:
    transform 0.3s ease,
    box-shadow 0.3s ease;
}
table {
  width: 100%;
  table-layout: fixed;
}
td,
th {
  white-space: nowrap;
}
</style>

<style>
/* Chrome/Edge/Safari */
.custom-scroll {
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: transparent transparent;
  transition: scrollbar-color 0.3s ease;
}

/* Hiệu ứng hover cho Firefox */
.custom-scroll:hover {
  scrollbar-color: rgba(125, 125, 125, 0.6) transparent;
}

.custom-scroll::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

/* Track trong suốt */
.custom-scroll::-webkit-scrollbar-track {
  background: transparent;
}

/* Thumb (phần thanh cuộn chính) */
.custom-scroll::-webkit-scrollbar-thumb {
  background: transparent;
  border-radius: 9999px;
  transition:
    background 0.3s ease,
    opacity 0.3s ease;
}

/* Hover: hiện thanh cuộn */
.custom-scroll:hover::-webkit-scrollbar-thumb {
  background: rgba(125, 125, 125, 0.6);
}
.custom-scroll::-webkit-scrollbar-thumb:hover {
  background: rgba(125, 125, 125, 0.8);
}

/* 🚫 Ẩn hoàn toàn mũi tên hai đầu */
.custom-scroll::-webkit-scrollbar-button {
  display: none;
}
</style>
