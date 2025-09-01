<script setup lang="ts">
import { ref } from 'vue';

// --- 状態変数 ---
const selectedMenu = ref<string | null>(null);
const isPlaying = ref(false);
const volume = ref(0.5);

// --- Web Audio API関連 ---
let audioContext: AudioContext;
let masterGainNode: GainNode;
let activeNodes: AudioNode[] = [];

// --- 関数定義 ---

// 音楽再生を開始するメインの関数（改良版）
const playMusic = (menuName: string) => {
  // もし既に何かが再生中なら、まずそれを停止する
  if (isPlaying.value) {
    stopMusic();
  }

  // AudioContextがなければ初期化
  if (!audioContext || audioContext.state === 'closed') {
    audioContext = new AudioContext();
  }
  
  // マスターゲインを初期化
  masterGainNode = audioContext.createGain();
  masterGainNode.gain.setValueAtTime(volume.value, audioContext.currentTime);
  masterGainNode.connect(audioContext.destination);

  // メニュー名に応じて再生する音を切り替える
  switch (menuName) {
    case '集中ブレンド':
      createConcentrationSound();
      break;
    default:
      createTestSound();
      break;
  }
  
  isPlaying.value = true;
  selectedMenu.value = menuName;
};

// 音楽を停止する関数（改良版）
const stopMusic = () => {
  if (!isPlaying.value) return;

  // アクティブなノードをすべて停止・切断
  activeNodes.forEach(node => {
    if (node instanceof OscillatorNode || node instanceof AudioBufferSourceNode) {
      try {
        'stop' in node && node.stop();
      } catch (e) {
        // すでに停止している場合のエラーは無視
      }
    }
    node.disconnect();
  });
  activeNodes = [];
  
  isPlaying.value = false;
  // selectedMenu.value は次の再生のために残しておく
};

// 再生/一時停止の切り替え（改良版）
const togglePlayback = () => {
  if (isPlaying.value) {
    stopMusic();
  } else {
    // 最後に選択されていたメニューがあれば、それを再生する
    if (selectedMenu.value) {
      playMusic(selectedMenu.value);
    }
  }
};

// 音量変更のハンドリング
const handleVolumeChange = (event: Event) => {
  const newVolume = parseFloat((event.target as HTMLInputElement).value);
  volume.value = newVolume;
  if (masterGainNode) {
    masterGainNode.gain.linearRampToValueAtTime(newVolume, audioContext.currentTime + 0.1);
  }
};

// --- サウンド生成関数 (変更なし) ---
const createConcentrationSound = () => {
  const bufferSize = 2 * audioContext.sampleRate;
  const noiseBuffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
  const output = noiseBuffer.getChannelData(0);
  for (let i = 0; i < bufferSize; i++) { output[i] = Math.random() * 2 - 1; }
  const whiteNoise = audioContext.createBufferSource();
  whiteNoise.buffer = noiseBuffer;
  whiteNoise.loop = true;
  const pinkFilter = audioContext.createBiquadFilter();
  pinkFilter.type = 'lowpass';
  pinkFilter.frequency.value = 1200;
  const lfo = audioContext.createOscillator();
  lfo.type = 'sine';
  lfo.frequency.value = 0.2;
  const lfoGain = audioContext.createGain();
  lfoGain.gain.value = 0.05;
  whiteNoise.connect(pinkFilter);
  pinkFilter.connect(masterGainNode);
  lfo.connect(lfoGain);
  lfoGain.connect(masterGainNode.gain);
  whiteNoise.start();
  lfo.start();
  activeNodes.push(whiteNoise, pinkFilter, lfo, lfoGain);
};
const createTestSound = () => {
  const oscillator = audioContext.createOscillator();
  oscillator.type = 'sine';
  oscillator.frequency.setValueAtTime(440, audioContext.currentTime);
  oscillator.connect(masterGainNode);
  oscillator.start();
  activeNodes.push(oscillator);
};
</script>

<template>
  <!-- (HTML部分は変更ありません) -->
  <div class="background-container">
    <div class="content-panel">
      <!-- ... -->
      <h1 class="title">AI-BGM 喫茶「おとや」</h1>
      <p class="subtitle">本日のBGMをお選びください</p>
      <div class="menu-container">
        <button class="menu-button" @click="playMusic('集中ブレンド')">
          <span class="menu-title">集中ブレンド</span>
          <span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span>
        </button>
        <button class="menu-button" @click="playMusic('リラックス・デカフェ')">
          <span class="menu-title">リラックス・デカフェ</span>
          <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
        </button>
        <button class="menu-button" @click="playMusic('ジャズ・スペシャル')">
          <span class="menu-title">ジャズ・スペシャル</span>
          <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
        </button>
        <button class="menu-button" @click="playMusic('Lo-Fi・ビター')">
          <span class="menu-title">Lo-Fi・ビター</span>
          <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
        </button>
      </div>
      <div class="controls-container">
        <button @click="togglePlayback" class="control-button">
          {{ isPlaying ? '■' : '▶' }}
        </button>
        <input 
          type="range" 
          min="0" 
          max="1" 
          step="0.01" 
          :value="volume" 
          @input="handleVolumeChange" 
          class="volume-slider"
        />
      </div>
    </div>
  </div>
</template>

<style>
/* (CSS部分は変更ありません) */
body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif; }
/* ... (以下、変更なし) ... */
.background-container { background-image: url('/bg-main.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
.content-panel { background-color: rgba(255, 255, 255, 0.85); padding: 20px 40px 30px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); backdrop-filter: blur(5px); width: 90%; max-width: 600px; text-align: center; }
.title { color: #363636; font-weight: bold; margin-bottom: 8px; }
.subtitle { color: #555; margin-top: 0; margin-bottom: 30px; }
.menu-container { display: flex; flex-direction: column; gap: 15px; }
.menu-button { background-color: #f5f5f5; border: 1px solid #dbdbdb; border-radius: 4px; padding: 15px 20px; cursor: pointer; transition: all 0.2s ease; text-align: left; display: flex; flex-direction: column; font-family: inherit; }
.menu-button:hover { background-color: #e8e8e8; border-color: #b5b5b5; transform: translateY(-2px); box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.menu-title { font-size: 1.1em; font-weight: bold; color: #363636; }
.menu-description { font-size: 0.9em; color: #7a7a7a; margin-top: 4px; }
.controls-container { margin-top: 30px; display: flex; justify-content: center; align-items: center; gap: 20px; }
.control-button { background-color: #363636; color: white; border: none; border-radius: 50%; width: 50px; height: 50px; font-size: 20px; cursor: pointer; display: flex; justify-content: center; align-items: center; transition: background-color 0.2s ease; }
.control-button:hover { background-color: #555; }
.volume-slider { width: 150px; cursor: pointer; }
</style>