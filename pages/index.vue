<template>
  <client-only>
    <div class="container mx-auto p-4">
      <h1 class="text-2xl font-bold mb-4">まるっとボイス エディター</h1>
      
      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">音声入力</h2>
        <div class="flex flex-col space-y-2">
          <div>
            <label class="block mb-1">マイク入力</label>
            <button @click="startMicrophone" class="btn btn-primary mr-2" :disabled="isProcessing">マイクを有効化</button>
            <button @click="stopAudio" class="btn btn-secondary" :disabled="!isProcessing">停止</button>
          </div>
          <div>
            <label class="block mb-1">ファイル入力</label>
            <input type="file" accept="audio/*" @change="onFileChange" class="mb-2" />
            <button @click="playAudioFile" class="btn btn-primary" :disabled="!audioFile || isProcessing">再生</button>
          </div>
        </div>
      </div>
      
      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">ノイズ除去設定</h2>
        <div class="flex items-center mb-2">
          <input type="checkbox" id="enableDenoise" v-model="enableDenoise" class="mr-2" />
          <label for="enableDenoise">ノイズ除去を有効化</label>
        </div>
        <div v-if="enableDenoise" class="mb-2">
          <label for="noiseReductionBlend" class="block mb-1">ノイズ除去ブレンド: {{ noiseReductionBlend.toFixed(2) }}</label>
          <input type="range" id="noiseReductionBlend" v-model="noiseReductionBlend" min="0" max="1" step="0.01" class="w-full" />
        </div>
      </div>

      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">音量調整</h2>
        <div class="mb-2">
          <label for="gainValue" class="block mb-1">ゲイン: {{ gainValue.toFixed(1) }}</label>
          <input type="range" id="gainValue" v-model="gainValue" min="0.1" max="2" step="0.1" class="w-full" />
        </div>
      </div>

      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">EQ設定</h2>
        <div class="mb-2">
          <label for="eqLow" class="block mb-1">Low (100Hz): {{ eqLow }} dB</label>
          <input type="range" id="eqLow" v-model="eqLow" min="-12" max="12" step="1" class="w-full" />
        </div>
        <div class="mb-2">
          <label for="eqLowMid" class="block mb-1">Low-Mid (300Hz): {{ eqLowMid }} dB</label>
          <input type="range" id="eqLowMid" v-model="eqLowMid" min="-12" max="12" step="1" class="w-full" />
        </div>
        <div class="mb-2">
          <label for="eqMid" class="block mb-1">Mid (1kHz): {{ eqMid }} dB</label>
          <input type="range" id="eqMid" v-model="eqMid" min="-12" max="12" step="1" class="w-full" />
        </div>
        <div class="mb-2">
          <label for="eqHighMid" class="block mb-1">High-Mid (3kHz): {{ eqHighMid }} dB</label>
          <input type="range" id="eqHighMid" v-model="eqHighMid" min="-12" max="12" step="1" class="w-full" />
        </div>
        <div class="mb-2">
          <label for="eqHigh" class="block mb-1">High (10kHz): {{ eqHigh }} dB</label>
          <input type="range" id="eqHigh" v-model="eqHigh" min="-12" max="12" step="1" class="w-full" />
        </div>
        <div class="mb-2">
          <label for="eqQ" class="block mb-1">Q値: {{ eqQ.toFixed(1) }}</label>
          <input type="range" id="eqQ" v-model="eqQ" min="0.1" max="10" step="0.1" class="w-full" />
        </div>
      </div>

      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">コンプレッサー設定</h2>
        <div class="mb-2">
          <label for="compressorThreshold" class="block mb-1">閾値: {{ compressorThreshold }} dB</label>
          <input type="range" id="compressorThreshold" v-model="compressorThreshold" min="-60" max="0" step="1" class="w-full" />
        </div>
      </div>

      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">リミッター設定</h2>
        <div class="mb-2">
          <label for="limiterThreshold" class="block mb-1">閾値: {{ limiterThreshold }} dB</label>
          <input type="range" id="limiterThreshold" v-model="limiterThreshold" min="-20" max="0" step="1" class="w-full" />
        </div>
      </div>
      
      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">音声波形</h2>
        <div class="flex space-x-4">
          <div class="w-1/2">
            <h3 class="text-lg mb-1">入力</h3>
            <canvas ref="inputCanvas" class="w-full h-32 bg-gray-100 border"></canvas>
          </div>
          <div class="w-1/2">
            <h3 class="text-lg mb-1">出力</h3>
            <canvas ref="outputCanvas" class="w-full h-32 bg-gray-100 border"></canvas>
          </div>
        </div>
      </div>
      
      <div class="mb-4">
        <h2 class="text-xl font-semibold mb-2">EQ波形</h2>
        <div class="flex space-x-4">
          <div class="w-1/2">
            <h3 class="text-lg mb-1">EQ前</h3>
            <canvas ref="eqBeforeCanvas" class="w-full h-32 bg-gray-100 border"></canvas>
          </div>
          <div class="w-1/2">
            <h3 class="text-lg mb-1">EQ後</h3>
            <canvas ref="eqAfterCanvas" class="w-full h-32 bg-gray-100 border"></canvas>
          </div>
        </div>
      </div>
      
      <div class="mb-4">
        <button @click="processAndDownload" class="btn btn-primary" :disabled="!isProcessing">処理してダウンロード</button>
      </div>
      
      <div v-if="errorMessage" class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
        {{ errorMessage }}
      </div>
      
      <div v-if="loadingMessage" class="bg-blue-100 border border-blue-400 text-blue-700 px-4 py-3 rounded mb-4">
        {{ loadingMessage }}
      </div>
    </div>
  </client-only>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watchEffect } from 'vue';

// RNNoise WASM module will be imported dynamically
import { Rnnoise } from '@shiguredo/rnnoise-wasm';
let rnnoiseInstance = null;
let denoiseState = null;

// Audio context and nodes
let audioContext = null;
let sourceNode = null;
let scriptNode = null;
let analyserInputNode = null;
let analyserOutputNode = null;
let gainNode = null;
let eqLowNode = null;
let eqLowMidNode = null;
let eqMidNode = null;
let eqHighMidNode = null;
let eqHighNode = null;
let compressorNode = null;
let limiterNode = null;
let eqBeforeAnalyser = null;
let eqAfterAnalyser = null;
let mediaRecorder = null;
let recordedChunks = [];

// UI state
const inputCanvas = ref(null);
const outputCanvas = ref(null);
const enableDenoise = ref(true);
const noiseReductionBlend = ref(1.0);
const gainValue = ref(1.0);
const eqLow = ref(0);
const eqLowMid = ref(0);
const eqMid = ref(0);
const eqHighMid = ref(0);
const eqHigh = ref(0);
const eqQ = ref(1.0);
const compressorThreshold = ref(-24);
const limiterThreshold = ref(-3);
const isProcessing = ref(false);
const errorMessage = ref('');
const loadingMessage = ref('');
const audioFile = ref(null);

// EQ visualization canvases
const eqBeforeCanvas = ref(null);
const eqAfterCanvas = ref(null);

// Constants for audio processing
const FRAME_SIZE = 480; // 10ms at 48kHz
const BUFFER_SIZE = 4096;

// Load RNNoise WASM module
onMounted(async () => {
  try {
    loadingMessage.value = 'RNNoise WAMSモジュールを読み込み中...';
    
    // Load WASM with correct path
    rnnoiseInstance = await Rnnoise.load({ 
      assetsPath: '' 
    });
    
    loadingMessage.value = 'RNNoise WAMSモジュールの読み込みが完了しました';
    setTimeout(() => {
      loadingMessage.value = '';
    }, 2000);
  } catch (error) {
    console.error('Failed to load RNNoise WASM module:', error);
    errorMessage.value = `RNNoise WAMSモジュールの読み込みに失敗しました: ${error.message}`;
  }
});

// Clean up resources
onBeforeUnmount(() => {
  stopAudio();
  if (denoiseState) {
    denoiseState.destroy();
    denoiseState = null;
  }
});

// Initialize audio context and analyzer nodes
const initAudioContext = () => {
  if (audioContext) return;
  
  try {
    audioContext = new (window.AudioContext || window.webkitAudioContext)({
      sampleRate: 48000 // RNNoise works best with 48kHz
    });
    
    // Create analyzer nodes for visualizing input and output
    analyserInputNode = audioContext.createAnalyser();
    analyserInputNode.fftSize = 2048;
    
    analyserOutputNode = audioContext.createAnalyser();
    analyserOutputNode.fftSize = 2048;
    
    // Create analyzer nodes for EQ visualization
    eqBeforeAnalyser = audioContext.createAnalyser();
    eqBeforeAnalyser.fftSize = 2048;
    
    eqAfterAnalyser = audioContext.createAnalyser();
    eqAfterAnalyser.fftSize = 2048;
    
    // Start visualization
    requestAnimationFrame(updateVisualization);
  } catch (error) {
    console.error('Failed to initialize audio context:', error);
    errorMessage.value = `オーディオコンテキストの初期化に失敗しました: ${error.message}`;
  }
};

// Handle microphone input
const startMicrophone = async () => {
  try {
    stopAudio();
    initAudioContext();
    
    // Get microphone stream
    const stream = await navigator.mediaDevices.getUserMedia({ 
      audio: { 
        echoCancellation: false,
        noiseSuppression: false,
        autoGainControl: false
      } 
    });
    
    // Create source node from microphone
    sourceNode = audioContext.createMediaStreamSource(stream);
    
    // Set up audio processing
    setupAudioProcessing();
    
    isProcessing.value = true;
  } catch (error) {
    console.error('Failed to access microphone:', error);
    errorMessage.value = `マイクへのアクセスに失敗しました: ${error.message}`;
  }
};

// Handle file input
const onFileChange = (event) => {
  const file = event.target.files[0];
  if (file) {
    audioFile.value = file;
  }
};

// Play audio file
const playAudioFile = async () => {
  if (!audioFile.value) return;
  
  try {
    stopAudio();
    initAudioContext();
    
    // Read file as array buffer
    const arrayBuffer = await audioFile.value.arrayBuffer();
    
    // Decode audio data
    const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
    
    // Create source node from audio buffer
    sourceNode = audioContext.createBufferSource();
    sourceNode.buffer = audioBuffer;
    
    // Set up audio processing
    setupAudioProcessing();
    
    // Start playback
    sourceNode.start();
    isProcessing.value = true;
    
    // Stop when playback ends
    sourceNode.onended = () => {
      stopAudio();
    };
  } catch (error) {
    console.error('Failed to play audio file:', error);
    errorMessage.value = `オーディオファイルの再生に失敗しました: ${error.message}`;
  }
};

// Set up audio processing pipeline
const setupAudioProcessing = () => {
  if (!sourceNode || !audioContext) return;
  
  try {
    // Create script processor node for custom processing
    scriptNode = audioContext.createScriptProcessor(BUFFER_SIZE, 1, 1);
    
    // Initialize RNNoise denoiser if not already created
    if (!denoiseState && rnnoiseInstance) {
      denoiseState = rnnoiseInstance.createDenoiseState();
    }
    
    // Create gain node
    gainNode = audioContext.createGain();
    gainNode.gain.value = gainValue.value;
    
    // Create EQ nodes
    eqLowNode = audioContext.createBiquadFilter();
    eqLowNode.type = 'lowshelf';
    eqLowNode.frequency.value = 100;
    eqLowNode.gain.value = eqLow.value;
    
    eqLowMidNode = audioContext.createBiquadFilter();
    eqLowMidNode.type = 'peaking';
    eqLowMidNode.frequency.value = 300;
    eqLowMidNode.Q.value = eqQ.value;
    eqLowMidNode.gain.value = eqLowMid.value;
    
    eqMidNode = audioContext.createBiquadFilter();
    eqMidNode.type = 'peaking';
    eqMidNode.frequency.value = 1000;
    eqMidNode.Q.value = eqQ.value;
    eqMidNode.gain.value = eqMid.value;
    
    eqHighMidNode = audioContext.createBiquadFilter();
    eqHighMidNode.type = 'peaking';
    eqHighMidNode.frequency.value = 3000;
    eqHighMidNode.Q.value = eqQ.value;
    eqHighMidNode.gain.value = eqHighMid.value;
    
    eqHighNode = audioContext.createBiquadFilter();
    eqHighNode.type = 'highshelf';
    eqHighNode.frequency.value = 10000;
    eqHighNode.gain.value = eqHigh.value;
    
    // Create compressor node
    compressorNode = audioContext.createDynamicsCompressor();
    compressorNode.threshold.value = compressorThreshold.value;
    compressorNode.ratio.value = 4;
    compressorNode.attack.value = 0.005;
    compressorNode.release.value = 0.25;
    
    // Create limiter node
    limiterNode = audioContext.createDynamicsCompressor();
    limiterNode.threshold.value = limiterThreshold.value;
    limiterNode.ratio.value = 20;
    limiterNode.attack.value = 0.001;
    limiterNode.release.value = 0.1;
    
    // Process audio
    scriptNode.onaudioprocess = (audioProcessingEvent) => {
      const inputBuffer = audioProcessingEvent.inputBuffer;
      const outputBuffer = audioProcessingEvent.outputBuffer;
      
      const inputData = inputBuffer.getChannelData(0);
      const outputData = outputBuffer.getChannelData(0);
      
      if (enableDenoise.value && denoiseState) {
        // Process audio in chunks of FRAME_SIZE
        for (let i = 0; i < inputData.length; i += FRAME_SIZE) {
          const frameSize = Math.min(FRAME_SIZE, inputData.length - i);
          
          // Create input frame
          const inputFrame = new Float32Array(frameSize);
          for (let j = 0; j < frameSize; j++) {
            inputFrame[j] = inputData[i + j];
          }
          
          // Process frame with RNNoise
          const outputFrame = denoiseState.processFrame(inputFrame);
          
          // Apply noise reduction blend
          for (let j = 0; j < frameSize; j++) {
            // Blend between original and processed audio
            outputData[i + j] = (outputFrame[j] * noiseReductionBlend.value) + 
                               (inputFrame[j] * (1 - noiseReductionBlend.value));
          }
        }
      } else {
        // Pass through without processing
        for (let i = 0; i < inputData.length; i++) {
          outputData[i] = inputData[i];
        }
      }
    };
    
    // Connect nodes
    sourceNode.connect(analyserInputNode);
    analyserInputNode.connect(scriptNode);
    scriptNode.connect(eqBeforeAnalyser);
    eqBeforeAnalyser.connect(eqLowNode);
    eqLowNode.connect(eqLowMidNode);
    eqLowMidNode.connect(eqMidNode);
    eqMidNode.connect(eqHighMidNode);
    eqHighMidNode.connect(eqHighNode);
    eqHighNode.connect(eqAfterAnalyser);
    eqAfterAnalyser.connect(compressorNode);
    compressorNode.connect(limiterNode);
    limiterNode.connect(gainNode);
    gainNode.connect(analyserOutputNode);
    analyserOutputNode.connect(audioContext.destination);
    
    // Set up watchers for parameter changes
    watchEffect(() => {
      if (gainNode) gainNode.gain.value = gainValue.value;
    });
    
    watchEffect(() => {
      if (eqLowNode) eqLowNode.gain.value = eqLow.value;
    });
    
    watchEffect(() => {
      if (eqLowMidNode) {
        eqLowMidNode.gain.value = eqLowMid.value;
        eqLowMidNode.Q.value = eqQ.value;
      }
    });
    
    watchEffect(() => {
      if (eqMidNode) {
        eqMidNode.gain.value = eqMid.value;
        eqMidNode.Q.value = eqQ.value;
      }
    });
    
    watchEffect(() => {
      if (eqHighMidNode) {
        eqHighMidNode.gain.value = eqHighMid.value;
        eqHighMidNode.Q.value = eqQ.value;
      }
    });
    
    watchEffect(() => {
      if (eqHighNode) eqHighNode.gain.value = eqHigh.value;
    });
    
    watchEffect(() => {
      if (compressorNode) compressorNode.threshold.value = compressorThreshold.value;
    });
    
    watchEffect(() => {
      if (limiterNode) limiterNode.threshold.value = limiterThreshold.value;
    });
  } catch (error) {
    console.error('Failed to set up audio processing:', error);
    errorMessage.value = `オーディオ処理の設定に失敗しました: ${error.message}`;
  }
};

// Stop audio processing
const stopAudio = () => {
  if (sourceNode) {
    sourceNode.disconnect();
    sourceNode = null;
  }
  
  if (scriptNode) {
    scriptNode.disconnect();
    scriptNode = null;
  }
  
  if (analyserInputNode) {
    analyserInputNode.disconnect();
  }
  
  if (analyserOutputNode) {
    analyserOutputNode.disconnect();
  }
  
  if (gainNode) {
    gainNode.disconnect();
    gainNode = null;
  }
  
  if (eqLowNode) {
    eqLowNode.disconnect();
    eqLowNode = null;
  }
  
  if (eqLowMidNode) {
    eqLowMidNode.disconnect();
    eqLowMidNode = null;
  }
  
  if (eqMidNode) {
    eqMidNode.disconnect();
    eqMidNode = null;
  }
  
  if (eqHighMidNode) {
    eqHighMidNode.disconnect();
    eqHighMidNode = null;
  }
  
  if (eqHighNode) {
    eqHighNode.disconnect();
    eqHighNode = null;
  }
  
  if (compressorNode) {
    compressorNode.disconnect();
    compressorNode = null;
  }
  
  if (limiterNode) {
    limiterNode.disconnect();
    limiterNode = null;
  }
  
  if (eqBeforeAnalyser) {
    eqBeforeAnalyser.disconnect();
  }
  
  if (eqAfterAnalyser) {
    eqAfterAnalyser.disconnect();
  }
  
  if (mediaRecorder && mediaRecorder.state !== 'inactive') {
    mediaRecorder.stop();
  }
  
  isProcessing.value = false;
};

// Process and download audio
const processAndDownload = () => {
  if (!isProcessing.value || !audioContext) return;
  
  try {
    // Create a MediaStream destination node
    const destination = audioContext.createMediaStreamDestination();
    
    // Connect the output to the destination
    analyserOutputNode.disconnect(audioContext.destination);
    analyserOutputNode.connect(destination);
    
    // Create a MediaRecorder to record the processed audio
    recordedChunks = [];
    mediaRecorder = new MediaRecorder(destination.stream, { mimeType: 'audio/webm' });
    
    // Handle data available event
    mediaRecorder.ondataavailable = (event) => {
      if (event.data.size > 0) {
        recordedChunks.push(event.data);
      }
    };
    
    // Handle recording stop event
    mediaRecorder.onstop = () => {
      // Create a blob from the recorded chunks
      const blob = new Blob(recordedChunks, { type: 'audio/webm' });
      
      // Create a download link
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.style.display = 'none';
      a.href = url;
      a.download = 'processed_audio.webm';
      document.body.appendChild(a);
      
      // Trigger download
      a.click();
      
      // Clean up
      setTimeout(() => {
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      }, 100);
      
      // Reconnect to audio context destination for continued playback
      analyserOutputNode.connect(audioContext.destination);
    };
    
    // Start recording
    mediaRecorder.start();
    
    // Record for the duration of the audio file or a fixed duration for microphone
    if (sourceNode && sourceNode.buffer) {
      // For audio file, record for its duration
      setTimeout(() => {
        if (mediaRecorder && mediaRecorder.state !== 'inactive') {
          mediaRecorder.stop();
        }
      }, sourceNode.buffer.duration * 1000);
    } else {
      // For microphone, record for 10 seconds
      setTimeout(() => {
        if (mediaRecorder && mediaRecorder.state !== 'inactive') {
          mediaRecorder.stop();
        }
      }, 10000);
    }
    
    loadingMessage.value = '処理中...';
    setTimeout(() => {
      loadingMessage.value = '';
    }, 2000);
  } catch (error) {
    console.error('Failed to process and download audio:', error);
    errorMessage.value = `オーディオの処理とダウンロードに失敗しました: ${error.message}`;
  }
};

// Update visualization
const updateVisualization = () => {
  if (!inputCanvas.value || !outputCanvas.value || !eqBeforeCanvas.value || !eqAfterCanvas.value) {
    requestAnimationFrame(updateVisualization);
    return;
  }
  
  const inputCtx = inputCanvas.value.getContext('2d');
  const outputCtx = outputCanvas.value.getContext('2d');
  const eqBeforeCtx = eqBeforeCanvas.value.getContext('2d');
  const eqAfterCtx = eqAfterCanvas.value.getContext('2d');
  
  // Clear canvases
  inputCtx.clearRect(0, 0, inputCanvas.value.width, inputCanvas.value.height);
  outputCtx.clearRect(0, 0, outputCanvas.value.width, outputCanvas.value.height);
  eqBeforeCtx.clearRect(0, 0, eqBeforeCanvas.value.width, eqBeforeCanvas.value.height);
  eqAfterCtx.clearRect(0, 0, eqAfterCanvas.value.width, eqAfterCanvas.value.height);
  
  if (analyserInputNode && analyserOutputNode) {
    // Draw input waveform
    const inputDataArray = new Uint8Array(analyserInputNode.frequencyBinCount);
    analyserInputNode.getByteTimeDomainData(inputDataArray);
    drawWaveform(inputCtx, inputDataArray, inputCanvas.value.width, inputCanvas.value.height);
    
    // Draw output waveform
    const outputDataArray = new Uint8Array(analyserOutputNode.frequencyBinCount);
    analyserOutputNode.getByteTimeDomainData(outputDataArray);
    drawWaveform(outputCtx, outputDataArray, outputCanvas.value.width, outputCanvas.value.height);
  }
  
  if (eqBeforeAnalyser && eqAfterAnalyser) {
    // Draw EQ before waveform
    const eqBeforeDataArray = new Uint8Array(eqBeforeAnalyser.frequencyBinCount);
    eqBeforeAnalyser.getByteTimeDomainData(eqBeforeDataArray);
    drawWaveform(eqBeforeCtx, eqBeforeDataArray, eqBeforeCanvas.value.width, eqBeforeCanvas.value.height);
    
    // Draw EQ after waveform
    const eqAfterDataArray = new Uint8Array(eqAfterAnalyser.frequencyBinCount);
    eqAfterAnalyser.getByteTimeDomainData(eqAfterDataArray);
    drawWaveform(eqAfterCtx, eqAfterDataArray, eqAfterCanvas.value.width, eqAfterCanvas.value.height);
  }
  
  requestAnimationFrame(updateVisualization);
};

// Draw waveform on canvas
const drawWaveform = (ctx, dataArray, width, height) => {
  ctx.lineWidth = 2;
  ctx.strokeStyle = 'rgb(0, 125, 255)';
  ctx.beginPath();
  
  const sliceWidth = width / dataArray.length;
  let x = 0;
  
  for (let i = 0; i < dataArray.length; i++) {
    const v = dataArray[i] / 128.0;
    const y = v * height / 2;
    
    if (i === 0) {
      ctx.moveTo(x, y);
    } else {
      ctx.lineTo(x, y);
    }
    
    x += sliceWidth;
  }
  
  ctx.lineTo(width, height / 2);
  ctx.stroke();
};
</script>

<style scoped>
.container { max-width: 800px; }
label { display: block; margin-bottom: .5rem; }
button { margin-top: 1rem; }
.btn {
  @apply px-4 py-2 rounded;
}
.btn-primary {
  @apply bg-blue-500 text-white hover:bg-blue-600;
}
.btn-secondary {
  @apply bg-gray-500 text-white hover:bg-gray-600;
}
</style>
