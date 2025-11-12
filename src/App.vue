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
          coinStatus[symbol] === 'valid'
            ? 'ring-4 ring-green-400 shadow-xl scale-[1.02]'
            : coinStatus[symbol] === 'low'
              ? 'border-yellow-500 ring-2 ring-yellow-400 shadow-lg scale-[1.01]'
              : '',
          flashClass[symbol],
        ]"
      >
        <div class="flex items-center justify-between mb-3">
          <div>
            <h3 class="font-semibold text-lg">{{ symbol }}</h3>
            <!-- 👇 thêm dòng này ngay bên dưới -->
            <span class="text-xs text-blue-400 block mb-3">
              (x4: {{ getDaysLeft(symbol) }} ngày)
            </span>
          </div>

          <div class="flex flex-col items-end">
            <span
              class="text-sm font-medium px-2 py-1 rounded-full"
              :class="{
                'bg-green-700 text-green-100': coinStatus[symbol] === 'valid',
                'bg-yellow-700 text-yellow-100': coinStatus[symbol] === 'low',
                'bg-red-700 text-red-100': coinStatus[symbol] === 'invalid',
              }"
            >
              {{
                coinStatus[symbol] === 'valid'
                  ? 'Ổn định'
                  : coinStatus[symbol] === 'low'
                    ? 'Ổn định thấp'
                    : 'Không ổn định'
              }}
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
            <label class="block text-sm mb-1">📊 Số bản ghi biến động tối đa</label>
            <input
              type="number"
              v-model.number="settings.maxInvalid"
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
  maxInvalid: 2,
  priceThreshold: 10,
  stableTime: 10000,
})
const showSettings = ref(false)
const soundEnabled = ref(false)
const loading = ref(true)
const now = ref(Date.now())

// =================== 📊 CORE STATE ===================
const priceTracker = reactive<Record<string, PriceEntry[]>>({})
const coinStatus = reactive<Record<string, 'valid' | 'low' | 'invalid'>>({})
const flashClass = reactive<Record<string, string>>({})
const stableSince = reactive<Record<string, number>>({})
const wasValid = reactive<Record<string, boolean>>({})
const alphaToSymbol: Record<string, string> = {}
const listingTimes: Record<string, number> = {} // ⏱️ lưu ngày list

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

// =================== 🧮 STATUS CALC ===================
function getStatus(arr: number[]): 'valid' | 'low' | 'invalid' {
  if (arr.length < 2) return 'low'
  let invalids = 0
  const threshold = settings.value.priceThreshold
  const limit = settings.value.maxInvalid
  for (let i = 1; i < arr.length; i++) {
    if (Math.abs((arr[i] - arr[i - 1]) * 1e8) > threshold) invalids++
  }
  if (invalids === 0) return 'valid'
  if (invalids <= limit) return 'low'
  return 'invalid'
}

// ⏳ Tính ngày còn lại x4
function getDaysLeft(symbol: string): number | null {
  const t = listingTimes[symbol]
  if (!t) return null
  const end = t + 30 * 24 * 60 * 60 * 1000
  const remaining = Math.floor((end - now.value) / (24 * 60 * 60 * 1000))
  return remaining < 0 ? 0 : remaining
}

// =================== 🔔 NOTIFY & FLASH ===================
function flash(symbol: string, type: 'valid' | 'low' | 'invalid') {
  flashClass[symbol] =
    type === 'valid' ? 'flash-green' : type === 'low' ? 'flash-yellow' : 'flash-red'
  setTimeout(() => (flashClass[symbol] = ''), 300)
}
function notify(symbol: string, type: 'valid' | 'invalid') {
  if (type === 'invalid') playSound('invalid')
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
  const low: string[] = []

  for (const s in coinStatus) {
    const st = coinStatus[s]
    if (st === 'valid') valid.push(s)
    else if (st === 'low') low.push(s)
    // nhóm invalid KHÔNG sort ở đây, dùng invalidOrder để giữ thứ tự ổn định
  }

  return [...valid, ...low, ...invalidOrder.value]
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
    const st = getStatus(trades)
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

  const prices = arr.map((x) => x.price)
  const newStatus = getStatus(prices)
  const oldStatus = coinStatus[symbol]
  if (newStatus === oldStatus) return

  const t = Date.now()

  if (newStatus === 'valid') {
    if (!stableSince[symbol]) stableSince[symbol] = t
    if (t - stableSince[symbol] >= settings.value.stableTime) {
      // Kiểm tra trước khi đổi trạng thái
      const anyOtherValid = Object.entries(coinStatus).some(
        ([s, st]) => s !== symbol && st === 'valid',
      )

      // Chưa có coin nào valid → phát âm thanh
      if (!anyOtherValid) playSound('valid')

      coinStatus[symbol] = 'valid'
      removeItem(invalidOrder.value, symbol)

      wasValid[symbol] = true
      flash(symbol, 'valid')
      toast.success(`${symbol} ổn định ✅`, {
        position: 'bottom-right',
        autoClose: 1500,
      })
    }

    // Nếu chưa đủ thời gian ổn định thì KHÔNG đổi state, cũng không đụng invalidOrder
  } else {
    // Bất kỳ trạng thái nào khác “valid” => reset ổn định
    stableSince[symbol] = 0
    coinStatus[symbol] = newStatus

    if (newStatus === 'invalid') {
      // Vừa chuyển sang invalid -> đẩy lên đầu
      pushFrontUnique(invalidOrder.value, symbol)

      if (wasValid[symbol]) {
        wasValid[symbol] = false
        flash(symbol, 'invalid')
        notify(symbol, 'invalid')
      }
    } else {
      // rời invalid (sang low) -> loại khỏi invalidOrder
      if (oldStatus === 'invalid') removeItem(invalidOrder.value, symbol)
    }
  }
}

// =================== 🧩 RESET SETTINGS ===================
function resetSettings() {
  settings.value = { MAX_RECORD: 20, maxInvalid: 2, priceThreshold: 10, stableTime: 10000 }
  toast.info('⚙️ Đã khôi phục cài đặt mặc định', { position: 'bottom-right', autoClose: 1500 })
}

// =================== 🚀 INIT ===================
onMounted(async () => {
  const topCoins = await getTopCoins()

  for (const c of topCoins) {
    priceTracker[c.symbol] = []
    coinStatus[c.symbol] = 'low'
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
.flash-green {
  animation: flashGreen 0.4s ease-in-out;
}
.flash-red {
  animation: flashRed 0.4s ease-in-out;
}
.flash-yellow {
  animation: flashYellow 0.4s ease-in-out;
}

@keyframes flashGreen {
  0% {
    box-shadow: 0 0 25px rgba(34, 197, 94, 0.8);
    transform: scale(1.05);
  }
  100% {
    box-shadow: none;
    transform: scale(1);
  }
}
@keyframes flashRed {
  0% {
    box-shadow: 0 0 25px rgba(239, 68, 68, 0.8);
    transform: scale(1.05);
  }
  100% {
    box-shadow: none;
    transform: scale(1);
  }
}
@keyframes flashYellow {
  0% {
    box-shadow: 0 0 25px rgba(234, 179, 8, 0.8);
    transform: scale(1.05);
  }
  100% {
    box-shadow: none;
    transform: scale(1);
  }
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
  width: 0;
  height: 0;
  display: none;
  -webkit-appearance: none;
  background: none;
  content: none;
}

/* Safari đặc biệt đôi khi vẫn hiển thị — fix thêm */
.custom-scroll::-webkit-scrollbar-button:start:decrement,
.custom-scroll::-webkit-scrollbar-button:end:increment {
  display: none;
  width: 0;
  height: 0;
  -webkit-appearance: none;
  background: none;
  content: none;
}
</style>
