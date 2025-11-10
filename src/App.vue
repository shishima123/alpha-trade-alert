<template>
  <div class="min-h-screen bg-gray-900 p-6 text-gray-200">
    <!-- Header và checkbox -->
    <div class="flex items-center justify-between mb-6 gap-4 flex-wrap">
      <h1 class="text-2xl font-bold flex items-center gap-2">📊 Kiểm tra độ ổn định</h1>
      <div class="flex items-center gap-2">
        <input type="checkbox" id="enableSound" v-model="soundEnabled" class="w-4 h-4" />
        <label for="enableSound">🔊 Nhấn để bật âm thanh</label>
      </div>
    </div>

    <div v-if="loading" class="text-gray-400 text-center py-10">
      ⏳ Đang tải dữ liệu từ Binance...
    </div>

    <div v-else class="grid gap-6 grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
      <div
        v-for="(symbol, index) in sortedSymbols"
        :key="symbol"
        class="bg-gray-800 rounded-2xl shadow-md border border-gray-700 transition-all duration-300 p-5"
        :class="[
          coinStatus[symbol] === 'valid'
            ? 'ring-4 ring-green-400 shadow-xl scale-[1.02]'
            : coinStatus[symbol] === 'low'
              ? 'border-yellow-500 ring-2 ring-yellow-400 shadow-lg scale-[1.01]'
              : '',
          flashClass[symbol],
        ]"
      >
        <!-- Header + badge + thời gian ổn định -->
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

        <!-- Table giá -->
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
                v-for="(item, idx) in priceTracker[symbol].slice(0, maxLength)"
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
  </div>
</template>

<script setup lang="ts">
import axios from 'axios'
import { ref, reactive, computed, onMounted } from 'vue'
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

const BINANCE_STREAM = 'wss://nbstream.binance.com/w3w/wsa/stream'
const API_TOP =
  'https://www.binance.com/bapi/defi/v1/public/alpha-trade/aggTicker24?dataType=aggregate'
const MAX_RECORD = 20

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

// Cập nhật thời gian mỗi giây
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

function getStatus(arr: number[]): 'valid' | 'low' | 'invalid' {
  if (arr.length < 2) return 'low'
  let countInvalid = 0
  for (let i = 1; i < arr.length; i++) {
    if (Math.abs((arr[i] - arr[i - 1]) * 1e8) > 10) countInvalid++
  }
  if (countInvalid === 0) return 'valid'
  if (countInvalid <= 2) return 'low'
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

// ===== Collect initial data từ API =====
async function collectInitialData(symbol: string, alphaId: string) {
  try {
    const url = `https://www.binance.com/bapi/defi/v1/public/alpha-trade/agg-trades?limit=20&symbol=${alphaId.toUpperCase()}USDT`
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

// ===== Mounted =====
onMounted(async () => {
  const topCoins = await getTopCoins()
  defaultOrder.value = topCoins.map((c) => c.symbol)

  // Khởi tạo state
  topCoins.forEach((c) => {
    priceTracker[c.symbol] = []
    coinStatus[c.symbol] = 'low'
    flashClass[c.symbol] = ''
    alphaToSymbol[`${c.alphaId.toUpperCase()}USDT`] = c.symbol
    stableSince[c.symbol] = 0
    wasValid[c.symbol] = false
  })

  loading.value = false

  // Collect initial data
  await Promise.all(topCoins.map((c) => collectInitialData(c.symbol, c.alphaId)))

  // Mở WebSocket
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
    if (arr.length > MAX_RECORD) arr.pop()

    const currentStatus = getStatus(arr.map((t) => t.price))
    const prevState = coinStatus[symbol]
    const tNow = Date.now()

    if (currentStatus === 'valid') {
      if (!stableSince[symbol]) stableSince[symbol] = tNow
      if (tNow - stableSince[symbol] >= 10000 && prevState !== 'valid') {
        coinStatus[symbol] = 'valid'
        wasValid[symbol] = true // Lưu trạng thái từng valid
        playSound('valid')
        flash(symbol, 'valid')
        showToast(`${symbol} vừa đạt trạng thái hợp lệ ✅`, 'valid')
      }
    } else {
      stableSince[symbol] = 0

      if (currentStatus === 'low' && prevState !== 'low') {
        coinStatus[symbol] = 'low'
        flash(symbol, 'low')
        showToast(`${symbol} đang ổn định thấp ⚠️`, 'valid')
      } else if (currentStatus === 'invalid' && prevState !== 'invalid') {
        coinStatus[symbol] = 'invalid'
        flash(symbol, 'invalid')
        // Chỉ phát âm thanh nếu coin từng đạt trạng thái valid
        if (wasValid[symbol]) {
          playSound('invalid')
          showToast(`${symbol} vừa mất trạng thái hợp lệ ❌`, 'invalid')
        }
        wasValid[symbol] = false // reset trạng thái valid
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
</style>
