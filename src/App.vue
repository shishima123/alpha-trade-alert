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
          <h3 class="font-semibold text-lg">{{ symbol }}</h3>
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

        <div class="overflow-y-auto max-h-[240px]">
          <table class="min-w-full text-sm">
            <thead>
              <tr class="border-b border-gray-700">
                <th class="text-left py-1 font-medium">#</th>
                <th class="text-left py-1 font-medium">Giá</th>
                <th class="text-left py-1 font-medium">Δ</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(item, idx) in priceTracker[symbol].slice(0, settings.MAX_RECORD)"
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
            <label class="block text-sm mb-1">📊 Số điểm biến động tối đa</label>
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
import { ref, reactive, computed, onMounted } from 'vue'
import { useStorage } from '@vueuse/core'
import { toast } from 'vue3-toastify'
import 'vue3-toastify/dist/index.css'
import validSound from '@/assets/notification.mp3'
import invalidSound from '@/assets/alert.mp3'

interface CoinInfo {
  symbol: string
  alphaId: string
  volume24h: number
}
interface PriceEntry {
  price: number
  diff: number | null
}

// =================== CONFIG (dùng useStorage) ===================
const settings = useStorage('stability-settings', {
  MAX_RECORD: 20,
  maxInvalid: 2,
  priceThreshold: 10,
  stableTime: 10000,
})
const showSettings = ref(false)

// =================== CORE STATE ===================
const BINANCE_STREAM = 'wss://nbstream.binance.com/w3w/wsa/stream'
const API_TOP =
  'https://www.binance.com/bapi/defi/v1/public/alpha-trade/aggTicker24?dataType=aggregate'

const priceTracker = reactive<Record<string, PriceEntry[]>>({})
const alphaToSymbol = reactive<Record<string, string>>({})
const coinStatus = reactive<Record<string, 'valid' | 'low' | 'invalid'>>({})
const flashClass = reactive<Record<string, string>>({})
const stableSince = reactive<Record<string, number>>({})
const defaultOrder = ref<string[]>([])
const loading = ref(true)
const soundEnabled = ref(false)
const now = ref(Date.now())
const wasValid = reactive<Record<string, boolean>>({})

setInterval(() => {
  now.value = Date.now()
}, 1000)

const validAudio = new Audio(validSound)
const invalidAudio = new Audio(invalidSound)

// ===== Sorted: valid > low > invalid =====
const sortedSymbols = computed(() => {
  const validArr: string[] = []
  const lowArr: string[] = []
  const invalidArr: string[] = []
  defaultOrder.value.forEach((sym) => {
    const status = coinStatus[sym]
    if (status === 'valid') validArr.push(sym)
    else if (status === 'low') lowArr.push(sym)
    else invalidArr.push(sym)
  })
  return [...validArr, ...lowArr, ...invalidArr]
})

// ===== Helper =====
function playSound(type: 'valid' | 'invalid') {
  if (!soundEnabled.value) return
  const audio = type === 'valid' ? validAudio : invalidAudio
  audio.currentTime = 0
  audio.play().catch(() => {})
}

function flash(symbol: string, type: 'valid' | 'low' | 'invalid') {
  flashClass[symbol] =
    type === 'valid' ? 'flash-green' : type === 'low' ? 'flash-yellow' : 'flash-red'
  setTimeout(() => {
    flashClass[symbol] = ''
  }, 400)
}

function showToast(message: string, type: 'valid' | 'invalid') {
  toast[type === 'valid' ? 'success' : 'error'](message, {
    position: 'bottom-right',
    autoClose: 2000,
    hideProgressBar: false,
    closeOnClick: true,
    pauseOnHover: true,
  })
}

// ===== Logic tính trạng thái =====
function getStatus(arr: number[]): 'valid' | 'low' | 'invalid' {
  if (arr.length < 2) return 'low'
  let countInvalid = 0
  for (let i = 1; i < arr.length; i++) {
    if (Math.abs((arr[i] - arr[i - 1]) * 1e8) > settings.value.priceThreshold) countInvalid++
  }
  if (countInvalid === 0) return 'valid'
  if (countInvalid <= settings.value.maxInvalid) return 'low'
  return 'invalid'
}

// ===== Lấy top coin =====
async function getTopCoins(): Promise<CoinInfo[]> {
  const { data } = await axios.get(API_TOP)
  const tokens = data?.data ?? []
  const filtered = tokens.filter((c: any) => c.mulPoint === 4)
  const sorted = filtered.sort(
    (a: any, b: any) => parseFloat(b.volume24h || '0') - parseFloat(a.volume24h || '0'),
  )
  return sorted.slice(0, 10).map((c: any) => ({
    symbol: c.symbol,
    alphaId: c.alphaId,
    volume24h: parseFloat(c.volume24h || '0'),
  }))
}

// ===== Collect initial data =====
async function collectInitialData(symbol: string, alphaId: string) {
  try {
    const url = `https://www.binance.com/bapi/defi/v1/public/alpha-trade/agg-trades?limit=${settings.value.MAX_RECORD}&symbol=${alphaId.toUpperCase()}USDT`
    const { data } = await axios.get(url)
    const trades: any[] = data?.data ?? []
    const arr: PriceEntry[] = trades.map((t) => ({ price: parseFloat(t.p), diff: null }))
    for (let i = 1; i < arr.length; i++) arr[i].diff = (arr[i].price - arr[i - 1].price) * 1e8

    priceTracker[symbol] = arr
    const status = getStatus(arr.map((t) => t.price))
    coinStatus[symbol] = status
    stableSince[symbol] = status === 'valid' ? Date.now() : 0
  } catch (err) {
    console.error(`Collect data ${symbol} lỗi`, err)
  }
}
function resetSettings() {
  settings.value = {
    MAX_RECORD: 20,
    maxInvalid: 2,
    priceThreshold: 10,
    stableTime: 10000,
  }
  toast.info('Đã khôi phục cài đặt mặc định ⚙️', {
    position: 'bottom-right',
    autoClose: 2000,
  })
}

// ===== Mounted =====
onMounted(async () => {
  const topCoins = await getTopCoins()
  defaultOrder.value = topCoins.map((c) => c.symbol)

  topCoins.forEach((c) => {
    priceTracker[c.symbol] = []
    coinStatus[c.symbol] = 'low'
    flashClass[c.symbol] = ''
    alphaToSymbol[`${c.alphaId.toUpperCase()}USDT`] = c.symbol
    stableSince[c.symbol] = 0
    wasValid[c.symbol] = false
  })

  loading.value = false
  await Promise.all(topCoins.map((c) => collectInitialData(c.symbol, c.alphaId)))

  const ws = new WebSocket(BINANCE_STREAM)
  ws.onopen = () => {
    const params = topCoins.map((c) => `${c.alphaId.toLowerCase()}usdt@aggTrade`)
    ws.send(JSON.stringify({ method: 'SUBSCRIBE', params, id: 1 }))
  }

  ws.onmessage = (event) => {
    const res = JSON.parse(event.data)
    if (!res.data || res.data.e !== 'aggTrade') return

    const alphaSymbol = res.data.s
    const symbol = alphaToSymbol[alphaSymbol]
    if (!symbol) return

    const price = parseFloat(res.data.p)
    const arr = priceTracker[symbol]
    const prev = arr.length > 0 ? arr[0].price : null
    const diff = prev !== null ? (price - prev) * 1e8 : null
    arr.unshift({ price, diff })
    if (arr.length > settings.value.MAX_RECORD) arr.pop()

    const currentStatus = getStatus(arr.map((t) => t.price))
    const prevState = coinStatus[symbol]
    const tNow = Date.now()

    if (currentStatus === 'valid') {
      if (!stableSince[symbol]) stableSince[symbol] = tNow

      const alreadyHasValid = Object.values(coinStatus).includes('valid')

      if (tNow - stableSince[symbol] >= settings.value.stableTime && prevState !== 'valid') {
        coinStatus[symbol] = 'valid'
        wasValid[symbol] = true

        // ✅ Nếu chưa có coin nào valid thì mới phát âm thanh valid
        if (!alreadyHasValid) {
          playSound('valid')
        }

        flash(symbol, 'valid')
        showToast(`${symbol} vừa đạt trạng thái ổn định ✅`, 'valid')
      }
    } else {
      stableSince[symbol] = 0
      if (currentStatus === 'low' && prevState !== 'low') {
        coinStatus[symbol] = 'low'
      } else if (currentStatus === 'invalid' && prevState !== 'invalid') {
        coinStatus[symbol] = 'invalid'
        if (wasValid[symbol]) {
          playSound('invalid')
          showToast(`${symbol} vừa mất ổn định ❌`, 'invalid')
        }
        wasValid[symbol] = false
      }
    }
  }
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
</style>
