<template>
  <div class="min-h-screen bg-gray-50 p-6">
    <!-- Header và checkbox trên 1 dòng -->
    <div class="flex items-center justify-between mb-6 gap-4 flex-wrap">
      <h1 class="text-2xl font-bold text-gray-800 flex items-center gap-2">
        📊 Kiểm tra độ ổn định
      </h1>
      <div class="flex items-center gap-2">
        <input type="checkbox" id="enableSound" v-model="soundEnabled" class="w-4 h-4" />
        <label for="enableSound" class="text-gray-700">🔊 Nhấn để bật âm thanh</label>
      </div>
    </div>

    <div v-if="loading" class="text-gray-500 text-center py-10">
      ⏳ Đang tải dữ liệu từ Binance...
    </div>

    <div v-else class="grid gap-6 grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
      <div
        v-for="(symbol, index) in sortedSymbols"
        :key="symbol"
        class="bg-white rounded-2xl shadow-md border border-gray-100 transition-all duration-300 p-5"
        :class="[
          coinStatus[symbol] === 'valid'
            ? 'ring-4 ring-green-400 shadow-xl scale-[1.02]'
            : coinStatus[symbol] === 'low'
              ? 'border-yellow-400 ring-2 ring-yellow-300 shadow-lg scale-[1.01]'
              : '',
          flashClass[symbol],
        ]"
      >
        <div class="flex items-center justify-between mb-3">
          <h3 class="font-semibold text-lg text-gray-800">{{ symbol }}</h3>
          <span
            class="text-sm font-medium px-2 py-1 rounded-full"
            :class="{
              'bg-green-100 text-green-700': coinStatus[symbol] === 'valid',
              'bg-yellow-100 text-yellow-700': coinStatus[symbol] === 'low',
              'bg-red-100 text-red-700': coinStatus[symbol] === 'invalid',
              'bg-gray-100 text-gray-500': coinStatus[symbol] === 'collecting',
            }"
          >
            {{
              coinStatus[symbol] === 'valid'
                ? 'Ổn định'
                : coinStatus[symbol] === 'low'
                  ? 'Ổn định thấp'
                  : coinStatus[symbol] === 'invalid'
                    ? 'Không ổn định'
                    : 'Đang thu thập...'
            }}
          </span>
        </div>

        <div class="overflow-y-auto max-h-[240px]">
          <table class="min-w-full text-sm">
            <thead>
              <tr class="border-b border-gray-200">
                <th class="text-left py-1 text-gray-500 font-medium">#</th>
                <th class="text-left py-1 text-gray-500 font-medium">Giá</th>
                <th class="text-left py-1 text-gray-500 font-medium">Δ</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="(item, idx) in priceTracker[symbol].slice(0, maxLength)"
                :key="idx"
                class="border-b last:border-0"
              >
                <td class="py-1">{{ idx + 1 }}</td>
                <td class="py-1 font-mono">{{ item.price.toFixed(8) }}</td>
                <td
                  class="py-1"
                  :class="
                    item.diff !== null
                      ? item.diff > 0
                        ? 'text-green-600'
                        : item.diff < 0
                          ? 'text-red-500'
                          : 'text-gray-600'
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
const API_URL =
  'https://www.binance.com/bapi/defi/v1/public/alpha-trade/aggTicker24?dataType=aggregate'

const maxLength = 20
const maxCoins = 10

const priceTracker = reactive<Record<string, PriceEntry[]>>({})
const alphaToSymbol = reactive<Record<string, string>>({})
const coinStatus = reactive<Record<string, 'collecting' | 'valid' | 'invalid'>>({})
const flashClass = reactive<Record<string, string>>({})
const loading = ref(true)
const defaultOrder = ref<string[]>([])

const soundEnabled = ref(false)

// Audio instances
const validAudio = new Audio(validSound)
const invalidAudio = new Audio(invalidSound)

// ==== Sorted coin: Valid lên đầu, còn lại theo thứ tự mặc định ====
const sortedSymbols = computed(() => {
  const top: string[] = []
  const rest: string[] = []

  defaultOrder.value.forEach((sym) => {
    if (coinStatus[sym] === 'valid' || coinStatus[sym] === 'low') top.push(sym)
    else rest.push(sym)
  })

  return [...top, ...rest]
})

// ==== Helper phát âm thanh ====
function playSound(type: 'valid' | 'invalid') {
  if (!soundEnabled.value) return
  const audio = type === 'valid' ? validAudio : invalidAudio
  audio.currentTime = 0
  audio.play().catch(() => {})
}

// ==== Flash hiệu ứng ====
function flash(symbol: string, type: 'valid' | 'invalid') {
  flashClass[symbol] = type === 'valid' ? 'flash-green' : 'flash-red'
  setTimeout(() => {
    flashClass[symbol] = ''
  }, 400)
}

// ==== Toast Vue3-Toastify ====
function showToast(message: string, type: 'valid' | 'invalid') {
  toast[type === 'valid' ? 'success' : 'error'](message, {
    position: 'bottom-right',
    autoClose: 2000,
    hideProgressBar: false,
    closeOnClick: true,
    pauseOnHover: true,
    draggable: true,
  })
}

// ==== Kiểm tra valid ====
function isValidArray(arr: number[]): boolean {
  if (arr.length < 2) return false
  for (let i = 1; i < arr.length; i++) {
    const diff = Math.abs((arr[i] - arr[i - 1]) * 1e8)
    if (diff > 10) return false
  }
  return true
}

async function collectInitialData(symbol: string, alphaId: string) {
  try {
    const url = `https://www.binance.com/bapi/defi/v1/public/alpha-trade/agg-trades?limit=20&symbol=${alphaId.toUpperCase()}USDT`
    const { data } = await axios.get(url)
    const trades: any[] = data?.data ?? []

    const arr: PriceEntry[] = trades.map((t) => ({
      price: parseFloat(t.p),
      diff: null,
    }))

    // Tính diff
    for (let i = 1; i < arr.length; i++) {
      arr[i].diff = (arr[i].price - arr[i - 1].price) * 1e8
    }

    // Lưu vào reactive
    priceTracker[symbol] = arr

    // Cập nhật trạng thái ngay
    const status = getStatus(arr.map((t) => t.price))
    coinStatus[symbol] = status
  } catch (err) {
    console.error(`Lỗi collectInitialData cho ${symbol}:`, err)
  }
}

// ==== Lấy top coins ====
async function getTopCoins(): Promise<CoinInfo[]> {
  const { data } = await axios.get(API_URL)
  const tokens = data?.data ?? []
  const filtered = tokens.filter((c: any) => c.mulPoint === 4)
  const sorted = filtered.sort(
    (a: any, b: any) => parseFloat(b.volume24h || '0') - parseFloat(a.volume24h || '0'),
  )
  return sorted.slice(0, maxCoins).map((c: any) => ({
    symbol: c.symbol,
    alphaId: c.alphaId,
    volume24h: parseFloat(c.volume24h || '0'),
  }))
}

function getStatus(arr: number[]): 'valid' | 'low' | 'invalid' | 'collecting' {
  if (arr.length < 2) return 'collecting'

  let countInvalid = 0
  for (let i = 1; i < arr.length; i++) {
    const diff = Math.abs((arr[i] - arr[i - 1]) * 1e8)
    if (diff > 10) countInvalid++
  }

  if (countInvalid === 0) return 'valid'
  if (countInvalid <= 2) return 'low' // Ổn định thấp
  return 'invalid'
}

// ==== Mounted ====
onMounted(async () => {
  const topCoins = await getTopCoins()
  defaultOrder.value = topCoins.map((c) => c.symbol)

  // Khởi tạo state
  topCoins.forEach((coin) => {
    priceTracker[coin.symbol] = []
    coinStatus[coin.symbol] = 'collecting'
    flashClass[coin.symbol] = ''
    alphaToSymbol[`${coin.alphaId.toUpperCase()}USDT`] = coin.symbol
  })

  loading.value = false

  // Collect initial data từ REST API
  await Promise.all(topCoins.map((coin) => collectInitialData(coin.symbol, coin.alphaId)))

  // Mở WebSocket
  const ws = new WebSocket(BINANCE_STREAM)
  ws.onopen = () => {
    console.log('✅ WebSocket connected')
    const params = topCoins.map((coin) => `${coin.alphaId.toLowerCase()}usdt@aggTrade`)
    ws.send(JSON.stringify({ method: 'SUBSCRIBE', params, id: 1 }))
  }

  ws.onmessage = (event) => {
    const res = JSON.parse(event.data)
    if (!res.data || res.data.e !== 'aggTrade') return

    const alphaSymbol = res.data.s
    const symbol = alphaToSymbol[alphaSymbol]
    const price = parseFloat(res.data.p)
    if (!symbol) return

    const arr = priceTracker[symbol]
    const prev = arr.length > 0 ? arr[0].price : null
    const diff = prev !== null ? (price - prev) * 1e8 : null

    arr.unshift({ price, diff })
    if (arr.length > maxLength) arr.pop()

    if (arr.length >= maxLength) {
      const newStatus = getStatus(arr.map((t) => t.price))
      const prevState = coinStatus[symbol]

      if (newStatus === 'valid' && prevState !== 'valid') {
        coinStatus[symbol] = 'valid'
        playSound('valid')
        flash(symbol, 'valid')
        showToast(`${symbol} vừa đạt trạng thái ổn định ✅`, 'valid')
      } else if (newStatus === 'low' && prevState !== 'low') {
        coinStatus[symbol] = 'low'
        flash(symbol, 'low')
        showToast(`${symbol} đang ổn định thấp ⚠️`, 'valid')
      } else if (newStatus === 'invalid' && prevState !== 'invalid') {
        coinStatus[symbol] = 'invalid'
        playSound('invalid')
        flash(symbol, 'invalid')
        showToast(`${symbol} vừa mất trạng thái ổn định ❌`, 'invalid')
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
