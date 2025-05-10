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
import { ref, onMounted, onBeforeUnmount } from 'vue';

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

// UI state
const inputCanvas = ref(null);
const outputCanvas = ref(null);
const enableDenoise = ref(true);
const isProcessing = ref(false);
const errorMessage = ref('');
const loadingMessage = ref('');
const audioFile = ref(null);

// Constants for audio processing
const FRAME_SIZE = 480; // 10ms at 48kHz
const BUFFER_SIZE = 4096;

// Load RNNoise WASM module
onMounted(async () => {
  try {
    loadingMessage.value = 'RNNoise WAMSモジュールを読み込み中...';
    
    // Load WASM with correct path
    rnnoiseInstance = await Rnnoise.load({ 
      locateFile: file => `${window.location.origin}/${file}` 
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
          
          // Copy processed frame to output
          for (let j = 0; j < frameSize; j++) {
            outputData[i + j] = outputFrame[j];
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
    scriptNode.connect(analyserOutputNode);
    analyserOutputNode.connect(audioContext.destination);
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
  
  isProcessing.value = false;
};

// Update visualization
const updateVisualization = () => {
  if (!inputCanvas.value || !outputCanvas.value) {
    requestAnimationFrame(updateVisualization);
    return;
  }
  
  const inputCtx = inputCanvas.value.getContext('2d');
  const outputCtx = outputCanvas.value.getContext('2d');
  
  // Clear canvases
  inputCtx.clearRect(0, 0, inputCanvas.value.width, inputCanvas.value.height);
  outputCtx.clearRect(0, 0, outputCanvas.value.width, outputCanvas.value.height);
  
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
