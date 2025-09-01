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
let soundInterval: number; // タイマーのIDを管理する変数

// --- 関数定義 ---

// 音楽再生を開始するメインの関数
const playMusic = (menuName: string) => {
  if (isPlaying.value) stopMusic();
  if (!audioContext || audioContext.state === 'closed') {
    audioContext = new AudioContext();
  }
  
  masterGainNode = audioContext.createGain();
  masterGainNode.gain.setValueAtTime(volume.value, audioContext.currentTime);
  masterGainNode.connect(audioContext.destination);

  switch (menuName) {
    case '集中ブレンド':
      createConcentrationSound();
      break;
    case 'リラックス・デカフェ':
      createRelaxSound();
      break;
    case 'ジャズ・スペシャル':
      createJazzSound();
      break;
    default:
      createTestSound();
      break;
  }
  
  isPlaying.value = true;
  selectedMenu.value = menuName;
};

// 音楽を停止する関数
const stopMusic = () => {
  if (!isPlaying.value) return;
  clearInterval(soundInterval); // ★全てのタイマーをこれで停止

  activeNodes.forEach(node => {
    if (node instanceof OscillatorNode || node instanceof AudioBufferSourceNode) {
      try { 'stop' in node && node.stop(); } catch (e) {}
    }
    node.disconnect();
  });
  activeNodes = [];
  
  isPlaying.value = false;
};

// 再生/一時停止の切り替え
const togglePlayback = () => {
  if (isPlaying.value) {
    stopMusic();
  } else {
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

// --- サウンド生成関数 ---

// 「集中ブレンド」のサウンドを生成
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

// 「リラックス・デカフェ」のサウンドを生成
const createRelaxSound = () => {
  const chords = [
    [261.63, 329.63, 392.00, 493.88], // Cmaj7
    [349.23, 440.00, 523.25, 659.26], // Fmaj7
    [392.00, 493.88, 587.33, 783.99], // G7
    [261.63, 329.63, 392.00, 493.88]  // Cmaj7
  ];
  let currentChordIndex = 0;
  let currentOscillators: { osc: OscillatorNode, gain: GainNode }[] = [];

  const playChord = () => {
    currentOscillators.forEach(({ osc, gain }) => {
      gain.gain.linearRampToValueAtTime(0, audioContext.currentTime + 2.5);
      osc.stop(audioContext.currentTime + 2.5);
    });
    
    const frequencies = chords[currentChordIndex];
    currentOscillators = frequencies.flatMap(freq => {
      const baseOsc = audioContext.createOscillator();
      baseOsc.type = 'sine';
      baseOsc.frequency.setValueAtTime(freq, audioContext.currentTime);
      const baseGain = audioContext.createGain();
      baseGain.gain.setValueAtTime(0, audioContext.currentTime);
      baseGain.gain.linearRampToValueAtTime(0.1, audioContext.currentTime + 2.0);
      baseOsc.connect(baseGain);
      baseGain.connect(masterGainNode);
      baseOsc.start();

      const overtoneOsc = audioContext.createOscillator();
      overtoneOsc.type = 'sine';
      overtoneOsc.frequency.setValueAtTime(freq * 2, audioContext.currentTime);
      const overtoneGain = audioContext.createGain();
      overtoneGain.gain.setValueAtTime(0, audioContext.currentTime);
      overtoneGain.gain.linearRampToValueAtTime(0.05, audioContext.currentTime + 2.0);
      overtoneOsc.connect(overtoneGain);
      overtoneGain.connect(masterGainNode);
      overtoneOsc.start();
      
      activeNodes.push(baseOsc, baseGain, overtoneOsc, overtoneGain);
      return [
        { osc: baseOsc, gain: baseGain },
        { osc: overtoneOsc, gain: overtoneGain }
      ];
    });
    currentChordIndex = (currentChordIndex + 1) % chords.length;
  };
  playChord();
  soundInterval = window.setInterval(playChord, 5000);
};

// 「ジャズ・スペシャル」のサウンドを生成
const createJazzSound = () => {
  let beat = 0;
  const tempo = 120; // BPM
  const intervalTime = 60000 / tempo / 2; // 8分音符の間隔

  // ピアノのメロディ用の音階 (Cマイナーペンタトニックスケール)
  const pianoScale = [261.6, 311.1, 349.2, 392.0, 466.1, 523.2];
  // ベースのコード進行 (Cm -> Fm -> G -> Cm)
  const bassLine = [130.8, 174.6, 196.0, 130.8];

  // 音符を再生するヘルパー関数
  const playNote = (freq: number, startTime: number, duration: number, vol: number, type: OscillatorType = 'sine') => {
    const osc = audioContext.createOscillator();
    const gain = audioContext.createGain();
    osc.type = type;
    osc.frequency.setValueAtTime(freq, startTime);
    gain.gain.setValueAtTime(0, startTime);
    gain.gain.linearRampToValueAtTime(vol, startTime + 0.01);
    gain.gain.linearRampToValueAtTime(0, startTime + duration);
    osc.connect(gain);
    gain.connect(masterGainNode);
    osc.start(startTime);
    osc.stop(startTime + duration);
  };
  
  // ハイハットを再生するヘルパー関数
  const playHiHat = (startTime: number) => {
    const noise = audioContext.createBufferSource();
    const bufferSize = audioContext.sampleRate * 0.1;
    const buffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) data[i] = Math.random() * 2 - 1;
    noise.buffer = buffer;

    const filter = audioContext.createBiquadFilter();
    filter.type = 'highpass';
    filter.frequency.value = 5000;

    const gain = audioContext.createGain();
    gain.gain.setValueAtTime(0, startTime);
    gain.gain.linearRampToValueAtTime(0.1, startTime + 0.01);
    gain.gain.linearRampToValueAtTime(0, startTime + 0.05);

    noise.connect(filter);
    filter.connect(gain);
    gain.connect(masterGainNode);
    noise.start(startTime);
  };

  // シーケンサー
  const sequencer = () => {
    const now = audioContext.currentTime;
    
    // 2拍目と4拍目にハイハット
    if (beat % 2 === 1) {
      playHiHat(now);
    }

    // 4拍ごとにベース音
    if (beat % 4 === 0) {
      const bassNote = bassLine[Math.floor(beat / 4) % bassLine.length];
      playNote(bassNote, now, intervalTime / 1000 * 2, 0.3);
    }
    
    // 15%の確率でピアノの音を鳴らす
    if (Math.random() < 0.15) {
      const pianoNote = pianoScale[Math.floor(Math.random() * pianoScale.length)];
      playNote(pianoNote, now, intervalTime / 1000, 0.2, 'triangle');
    }

    beat = (beat + 1) % 16; // 4小節でループ
  };

  soundInterval = window.setInterval(sequencer, intervalTime);
};

// テスト用のサウンドを生成
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
  <div class="background-container">
    <div class="content-panel">
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
body, html {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif;
}
.background-container {
  background-image: url('/bg-main.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
.content-panel {
  background-color: rgba(255, 255, 255, 0.85);
  padding: 20px 40px 30px;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  backdrop-filter: blur(5px);
  width: 90%;
  max-width: 600px;
  text-align: center;
}
.title {
  color: #363636;
  font-weight: bold;
  margin-bottom: 8px;
}
.subtitle {
  color: #555;
  margin-top: 0;
  margin-bottom: 30px;
}
.menu-container {
  display: flex;
  flex-direction: column;
  gap: 15px;
}
.menu-button {
  background-color: #f5f5f5;
  border: 1px solid #dbdbdb;
  border-radius: 4px;
  padding: 15px 20px;
  cursor: pointer;
  transition: all 0.2s ease;
  text-align: left;
  display: flex;
  flex-direction: column;
  font-family: inherit;
}
.menu-button:hover {
  background-color: #e8e8e8;
  border-color: #b5b5b5;
  transform: translateY(-2px);
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}
.menu-title {
  font-size: 1.1em;
  font-weight: bold;
  color: #363636;
}
.menu-description {
  font-size: 0.9em;
  color: #7a7a7a;
  margin-top: 4px;
}
.controls-container {
  margin-top: 30px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
}
.control-button {
  background-color: #363636;
  color: white;
  border: none;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: background-color 0.2s ease;
}
.control-button:hover {
  background-color: #555;
}
.volume-slider {
  width: 150px;
  cursor: pointer;
}
</style>