<template>
  <client-only>
    <div class="container mx-auto p-4">
      <h1 class="text-2xl font-bold mb-4">まるっとボイス エディター</h1>
      <input type="file" accept="audio/*" @change="onFileChange" class="mb-4" />
      <div v-if="audioBuffer">
        <h2 class="font-medium mb-2">調整パラメーター</h2>
        <div class="mb-4">
          <label class="block mb-1">ノイズ除去ブレンド: {{ blend.toFixed(2) }}
            <input type="range" min="0" max="1" step="0.01" v-model.number="blend" class="w-full" />
          </label>
          <label class="block mb-1">音量 (Gain): {{ volume.toFixed(2) }}
            <input type="range" min="0.1" max="2" step="0.01" v-model.number="volume" class="w-full" />
          </label>
          <label class="block mb-1">コンプレッサー Threshold: {{ compressorThreshold }} dB
            <input type="range" min="-60" max="0" step="1" v-model.number="compressorThreshold" class="w-full" />
          </label>
        </div>
        <h3 class="font-medium mb-2">EQ</h3>
        <div v-for="(band, i) in eqBands" :key="i" class="mb-2">
          <label class="block">{{ band.label }}: {{ band.gain }} dB
            <input type="range" :min="-12" :max="12" step="0.5" v-model.number="band.gain" class="w-full" />
          </label>
        </div>
        <label class="block mb-4">リミッター閾値 (dB): {{ limiterThreshold }}
          <input type="range" min="-10" max="0" step="0.5" v-model.number="limiterThreshold" class="w-full" />
        </label>
        <p class="mb-4">Voice Prob: {{ voiceProb.toFixed(2) }}</p>
        <canvas ref="waveformCanvas" width="600" height="100" class="border mb-4"></canvas>
        <button @click="processAndDownload" class="px-4 py-2 bg-blue-600 text-white rounded">処理してダウンロード</button>
      </div>
    </div>
  </client-only>
</template>

<script setup>
import { ref, reactive, onMounted, watch, nextTick } from 'vue'
import initRnnoise from '@shiguredo/rnnoise-wasm'

// 音声パラメータ
const blend = ref(0.7)
const volume = ref(1.0)
const compressorThreshold = ref(-24)
const limiterThreshold = ref(-1)
const eqBands = reactive([
  { freq: 100, type: 'lowshelf', gain: 0, label: 'Low' },
  { freq: 1000, type: 'peaking', gain: 0, label: 'Mid' },
  { freq: 5000, type: 'highshelf', gain: 0, label: 'High' }
])
const voiceProb = ref(0)

// 内部状態
const audioBuffer = ref(null)
let context = null
let rn = null
let analyser = null
let limiterNode = null

// ファイル読み込み
const onFileChange = async (e) => {
  const file = e.target.files[0]
  if (!file) return
  const arrayBuffer = await file.arrayBuffer()
  audioBuffer.value = await context.decodeAudioData(arrayBuffer)
}

// AudioContext / ノード設定
async function setupAudioGraph() {
  context = new AudioContext()

  // ソースノード
  const sourceNode = context.createBufferSource()
  sourceNode.buffer = audioBuffer.value

  // RNNoise
  const frameSize = 480
  const channelData = audioBuffer.value.getChannelData(0)
  const denoisedData = new Float32Array(channelData.length)
  for (let i = 0; i < channelData.length; i += frameSize) {
    const slice = channelData.subarray(i, i + frameSize)
    const out = rn.process(slice)
    denoisedData.set(out, i)
  }
  const denBuf = context.createBuffer(1, denoisedData.length, context.sampleRate)
  denBuf.copyToChannel(denoisedData, 0)
  const denSource = context.createBufferSource()
  denSource.buffer = denBuf

  // ミックス
  const gainOrig = context.createGain()
  const gainDeno = context.createGain()
  gainOrig.gain.value = 1 - blend.value
  gainDeno.gain.value = blend.value
  sourceNode.connect(gainOrig)
  denSource.connect(gainDeno)

  // コンプレッサー
  const compressor = context.createDynamicsCompressor()
  compressor.threshold.setValueAtTime(compressorThreshold.value, context.currentTime)

  // EQ
  let lastNode = compressor
  eqBands.forEach(b => {
    const f = context.createBiquadFilter()
    f.type = b.type; f.frequency.value = b.freq; f.gain.value = b.gain
    lastNode.connect(f)
    lastNode = f
  })

  // リミッター
  limiterNode = context.createDynamicsCompressor()
  limiterNode.threshold.setValueAtTime(limiterThreshold.value, context.currentTime)
  limiterNode.knee.setValueAtTime(0, context.currentTime)
  limiterNode.ratio.setValueAtTime(20, context.currentTime)
  limiterNode.attack.setValueAtTime(0.001, context.currentTime)
  limiterNode.release.setValueAtTime(0.05, context.currentTime)
  lastNode.connect(limiterNode)

  // Analyser for waveform
  analyser = context.createAnalyser()
  analyser.fftSize = 2048
  limiterNode.connect(analyser)
  analyser.connect(context.destination)

  return { sourceNode, denSource }
}

// 波形描画
function drawWaveform() {
  nextTick(() => {
    const canvas = document.querySelector('canvas')
    const ctx = canvas.getContext('2d')
    const bufferLength = analyser.fftSize
    const dataArray = new Uint8Array(bufferLength)

    const draw = () => {
      requestAnimationFrame(draw)
      analyser.getByteTimeDomainData(dataArray)
      ctx.fillStyle = '#fff'
      ctx.fillRect(0, 0, canvas.width, canvas.height)
      ctx.lineWidth = 2
      ctx.strokeStyle = '#007bff'
      ctx.beginPath()
      const sliceWidth = canvas.width / bufferLength
      let x = 0
      for (let i = 0; i < bufferLength; i++) {
        const v = dataArray[i] / 128.0
        const y = v * (canvas.height / 2)
        i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y)
        x += sliceWidth
      }
      ctx.lineTo(canvas.width, canvas.height / 2)
      ctx.stroke()
    }
    draw()
  })
}

// 処理＆ダウンロード
async function processAndDownload() {
  const { sourceNode, denSource } = await setupAudioGraph()
  const dest = context.createMediaStreamDestination()
  analyser.connect(dest)

  const recorder = new MediaRecorder(dest.stream)
  const chunks = []
  recorder.ondataavailable = e => chunks.push(e.data)
  recorder.onstop = () => {
    const blob = new Blob(chunks, { type: 'audio/webm' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url; a.download = 'processed.webm'; a.click()
  }
  recorder.start()
  sourceNode.start()
  denSource.start()
  sourceNode.onended = () => recorder.stop()
}

// リアルタイム更新
watch(blend, val => {})  // mixは次プロセスで反映
watch(volume, val => {})
watch(compressorThreshold, val => {})
watch(limiterThreshold, val => {
  if (limiterNode) limiterNode.threshold.setValueAtTime(val, context.currentTime)
})
watch(eqBands, bands => {})

// 初期化
onMounted(async () => {
  if (process.client) {
    rn = await initRnnoise()
    context = new AudioContext()
    drawWaveform()
  }
})
</script>

<style scoped>
.container { max-width: 800px; }
label { display: block; margin-bottom: .5rem; }
button { margin-top: 1rem; }
</style>