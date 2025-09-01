<script setup lang="ts">
import { ref } from 'vue';
import seedrandom from 'seedrandom';
import AboutModal from '../components/AboutModal.vue';

// --- 状態変数 ---
const selectedMenu = ref<string | null>(null);
const isPlaying = ref(false);
const volume = ref(0.5);
const isModalVisible = ref(false);
const currentSeed = ref<string>('');
const seedInput = ref<string>('');

// --- Web Audio API関連 ---
let audioContext: AudioContext;
let masterGainNode: GainNode;
let activeNodes: AudioNode[] = [];
let soundInterval: any;

// --- 関数定義 ---

// 音楽再生を開始するメインの関数
const playMusic = (menuName: string, seed?: string) => {
  if (isPlaying.value) stopMusic();
  if (!audioContext || audioContext.state === 'closed') {
    audioContext = new AudioContext();
  }
  
  const randomPart = seed || Date.now().toString(36) + Math.random().toString(36).substring(2);
  currentSeed.value = `${menuName}:${randomPart}`;
  const rng = seedrandom(randomPart);

  masterGainNode = audioContext.createGain();
  masterGainNode.gain.setValueAtTime(volume.value, audioContext.currentTime);
  masterGainNode.connect(audioContext.destination);

  switch (menuName) {
    case '集中ブレンド': createConcentrationSound(rng); break;
    case 'リラックス・デカフェ': createRelaxSound(rng); break;
    case 'ジャズ・スペシャル': createJazzSound(rng); break;
    case 'Lo-Fi・ビター': createLoFiSound(rng); break;
    default: createTestSound(); break;
  }
  isPlaying.value = true;
  selectedMenu.value = menuName;
};

const stopMusic = () => {
  if (!isPlaying.value) return;
  clearInterval(soundInterval);
  clearTimeout(soundInterval);
  activeNodes.forEach(node => {
    if (node instanceof OscillatorNode || node instanceof AudioBufferSourceNode) {
      try { 'stop' in node && node.stop(); } catch (e) {}
    }
    node.disconnect();
  });
  activeNodes = [];
  isPlaying.value = false;
};

const togglePlayback = () => {
  if (isPlaying.value) {
    stopMusic();
  } else {
    if (selectedMenu.value && currentSeed.value) {
      const [menuName, seed] = currentSeed.value.split(':');
      if (menuName && seed) playMusic(menuName, seed);
    }
  }
};

const handleVolumeChange = (event: Event) => {
  const newVolume = parseFloat((event.target as HTMLInputElement).value);
  volume.value = newVolume;
  if (masterGainNode) { masterGainNode.gain.linearRampToValueAtTime(newVolume, audioContext.currentTime + 0.1); }
};

const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const copySeed = () => { navigator.clipboard.writeText(currentSeed.value); };

const playFromSeed = () => {
  if (seedInput.value) {
    const [menuName, seed] = seedInput.value.split(':');
    const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター'];
    if (menuName && seed && validMenus.includes(menuName)) {
      playMusic(menuName, seed);
    } else {
      alert('レコード番号の形式が正しくないか、存在しないジャンルです。');
    }
  }
};


// --- サウンド生成関数 ---
const createConcentrationSound = (rng: () => number) => {
  const bufferSize = 2 * audioContext.sampleRate;
  const noiseBuffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
  const output = noiseBuffer.getChannelData(0);
  for (let i = 0; i < bufferSize; i++) { output[i] = rng() * 2 - 1; }
  const whiteNoise = audioContext.createBufferSource(); whiteNoise.buffer = noiseBuffer; whiteNoise.loop = true;
  const pinkFilter = audioContext.createBiquadFilter(); pinkFilter.type = 'lowpass'; pinkFilter.frequency.value = 1000 + rng() * 500;
  const lfo = audioContext.createOscillator(); lfo.type = 'sine'; lfo.frequency.value = 0.1 + rng() * 0.3;
  const lfoGain = audioContext.createGain(); lfoGain.gain.value = 0.03 + rng() * 0.05;
  whiteNoise.connect(pinkFilter); pinkFilter.connect(masterGainNode); lfo.connect(lfoGain); lfoGain.connect(masterGainNode.gain); whiteNoise.start(); lfo.start();
  activeNodes.push(whiteNoise, pinkFilter, lfo, lfoGain);
};

const createRelaxSound = (rng: () => number) => {
  const chordPool = [ [261.63, 329.63, 392.00, 493.88], [349.23, 440.00, 523.25, 659.26], [392.00, 493.88, 587.33, 783.99], [293.66, 369.99, 440.00, 554.37] ];
  let currentOscillators: { osc: OscillatorNode, gain: GainNode }[] = [];
  const playChord = () => {
    currentOscillators.forEach(({ osc, gain }) => {
      gain.gain.linearRampToValueAtTime(0, audioContext.currentTime + 2.5);
      osc.stop(audioContext.currentTime + 2.5);
    });
    const frequencies = chordPool[Math.floor(rng() * chordPool.length)];
    if (frequencies) {
      currentOscillators = frequencies.flatMap(freq => {
        const baseOsc = audioContext.createOscillator(); baseOsc.type = 'sine'; baseOsc.frequency.setValueAtTime(freq, audioContext.currentTime);
        const baseGain = audioContext.createGain(); baseGain.gain.setValueAtTime(0, audioContext.currentTime); baseGain.gain.linearRampToValueAtTime(0.1, audioContext.currentTime + 2.0);
        baseOsc.connect(baseGain); baseGain.connect(masterGainNode); baseOsc.start();
        const overtoneOsc = audioContext.createOscillator(); overtoneOsc.type = 'sine'; overtoneOsc.frequency.setValueAtTime(freq * 2, audioContext.currentTime);
        const overtoneGain = audioContext.createGain(); overtoneGain.gain.setValueAtTime(0, audioContext.currentTime); overtoneGain.gain.linearRampToValueAtTime(0.05, audioContext.currentTime + 2.0);
        overtoneOsc.connect(overtoneGain); overtoneGain.connect(masterGainNode); overtoneOsc.start();
        activeNodes.push(baseOsc, baseGain, overtoneOsc, overtoneGain);
        return [ { osc: baseOsc, gain: baseGain }, { osc: overtoneOsc, gain: overtoneGain } ];
      });
    }
  };
  const scheduleNextChord = () => { playChord(); const nextInterval = 4000 + rng() * 3000; soundInterval = setTimeout(scheduleNextChord, nextInterval); };
  scheduleNextChord();
};

const createJazzSound = (rng: () => number) => {
  let beat = 0; const tempo = 120; const intervalTime = 60000 / tempo;
  const pianoScale = [261.6, 311.1, 349.2, 392.0, 466.1, 523.2, 587.3];
  const bassPatterns = [ [130.8, 174.6, 196.0, 130.8], [130.8, 196.0, 174.6, 130.8] ];
  const bassLine = bassPatterns[Math.floor(rng() * bassPatterns.length)];
  let nextPianoTime = 0;
  const playNote = (freq: number, startTime: number, duration: number, vol: number, type: OscillatorType = 'sine') => { const osc = audioContext.createOscillator(); const gain = audioContext.createGain(); osc.type = type; osc.frequency.setValueAtTime(freq, startTime); gain.gain.setValueAtTime(0, startTime); gain.gain.linearRampToValueAtTime(vol, startTime + 0.01); gain.gain.linearRampToValueAtTime(0, startTime + duration); osc.connect(gain); gain.connect(masterGainNode); osc.start(startTime); osc.stop(startTime + duration); };
  const playHiHat = (startTime: number) => { const noise = audioContext.createBufferSource(); const bufferSize = audioContext.sampleRate * 0.1; const buffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate); const data = buffer.getChannelData(0); for (let i = 0; i < bufferSize; i++) data[i] = rng() * 2 - 1; noise.buffer = buffer; const filter = audioContext.createBiquadFilter(); filter.type = 'highpass'; filter.frequency.value = 5000; const gain = audioContext.createGain(); gain.gain.setValueAtTime(0, startTime); gain.gain.linearRampToValueAtTime(0.1, startTime + 0.01); gain.gain.linearRampToValueAtTime(0, startTime + 0.05); noise.connect(filter); filter.connect(gain); gain.connect(masterGainNode); noise.start(startTime); };
  const sequencer = () => {
    const now = audioContext.currentTime;
    playHiHat(now); playHiHat(now + (intervalTime / 1000) * 0.66);
    if (beat % 4 === 0 && bassLine) { const bassNote = bassLine[Math.floor(beat / 4) % bassLine.length]; if (bassNote) playNote(bassNote, now, intervalTime / 1000 * 1.5, 0.3); }
    if (now >= nextPianoTime) { const pianoNote = pianoScale[Math.floor(rng() * pianoScale.length)]; if (pianoNote) { const duration = (intervalTime / 1000) * (0.5 + rng() * 1.5); playNote(pianoNote, now, duration, 0.2, 'triangle'); } nextPianoTime = now + 0.25 + rng() * 1.25; }
    beat = (beat + 1) % 16;
  };
  soundInterval = setInterval(sequencer, intervalTime);
};

const createLoFiSound = (rng: () => number) => {
  const bufferSize = 2 * audioContext.sampleRate;
  const noiseBuffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
  const output = noiseBuffer.getChannelData(0); for (let i = 0; i < bufferSize; i++) { output[i] = rng() * 2 - 1; }
  const vinylNoise = audioContext.createBufferSource(); vinylNoise.buffer = noiseBuffer; vinylNoise.loop = true;
  const vinylFilter = audioContext.createBiquadFilter(); vinylFilter.type = 'highpass'; vinylFilter.frequency.value = 2500 + rng() * 1000;
  const vinylGain = audioContext.createGain(); vinylGain.gain.value = 0.04 + rng() * 0.02;
  vinylNoise.connect(vinylFilter); vinylFilter.connect(vinylGain); vinylGain.connect(masterGainNode); vinylNoise.start();
  activeNodes.push(vinylNoise, vinylFilter, vinylGain);
  const tempo = 80 + rng() * 15;
  const intervalTime = 60000 / tempo / 4;
  const chordPatterns = [ [[130.81, 196, 246.94, 311.13], [174.61, 261.63, 329.63, 415.3]], [[130.81, 196, 246.94, 311.13], [155.56, 233.08, 293.66, 369.99]] ];
  const chords = chordPatterns[Math.floor(rng() * chordPatterns.length)];
  const kickPatterns = [ [1,0,0,0, 1,0,0,0, 1,0,0,0, 1,0,0,0], [1,0,0,1, 0,0,1,0, 1,0,0,0, 1,0,1,0] ];
  const snarePatterns = [ [0,0,0,0, 1,0,0,0, 0,0,0,0, 1,0,0,0], [0,0,0,0, 1,0,0,1, 0,0,0,0, 1,0,0,0] ];
  const kickPattern = kickPatterns[Math.floor(rng() * kickPatterns.length)];
  const snarePattern = snarePatterns[Math.floor(rng() * snarePatterns.length)];
  let beat = 0;
  const playNote = (freq: number, sT: number, dur: number, vol: number, type: OscillatorType) => { const o = audioContext.createOscillator(), g = audioContext.createGain(); o.type = type; o.frequency.setValueAtTime(freq, sT); g.gain.setValueAtTime(0, sT); g.gain.linearRampToValueAtTime(vol, sT + 0.02); g.gain.exponentialRampToValueAtTime(0.0001, sT + dur); o.connect(g); g.connect(masterGainNode); o.start(sT); o.stop(sT + dur); };
  const playKick = (sT: number) => { playNote(60, sT, 0.2, 0.5, 'sine'); };
  const playSnare = (sT: number) => { const n = audioContext.createBufferSource(), bS = audioContext.sampleRate*0.1, b = audioContext.createBuffer(1, bS, audioContext.sampleRate), d = b.getChannelData(0); for (let i=0; i<bS; i++) d[i] = rng()*2-1; n.buffer = b; const f = audioContext.createBiquadFilter(); f.type = 'bandpass'; f.frequency.value = 1500; const g = audioContext.createGain(); g.gain.setValueAtTime(0, sT); g.gain.linearRampToValueAtTime(0.4, sT + 0.01); g.gain.exponentialRampToValueAtTime(0.0001, sT + 0.1); n.connect(f); f.connect(g); g.connect(masterGainNode); n.start(sT); };
  const playChord = (sT: number, freqs: number[]) => { freqs.forEach(f => { playNote(f, sT, 1.5, 0.1, 'triangle'); }); };
  const sequencer = () => {
    const now = audioContext.currentTime; const c16 = beat % 16;
    if (kickPattern?.[c16]) playKick(now);
    if (snarePattern?.[c16]) playSnare(now);
    if (c16 === 0 && chords) { const chord = chords[Math.floor(beat / 16) % chords.length]; if (chord) playChord(now, chord); }
    beat = (beat + 1) % 64;
  };
  soundInterval = setInterval(sequencer, intervalTime);
};

const createTestSound = () => { const o=audioContext.createOscillator(); o.type='sine'; o.frequency.setValueAtTime(440, audioContext.currentTime); o.connect(masterGainNode); o.start(); activeNodes.push(o);};
</script>

<template>
  <div class="background-container">
    <div class="content-panel">
      <h1 class="title">AI-BGM 喫茶「おとや」</h1>
      <p class="subtitle">本日のBGMをお選びください</p>
      <div class="menu-container">
        <button class="menu-button" @click="playMusic('集中ブレンド')">
          <div class="menu-content">
            <span class="menu-title">集中ブレンド</span>
            <span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span>
          </div>
          <div v-if="selectedMenu === '集中ブレンド'" class="active-indicator">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
          </div>
        </button>
        <button class="menu-button" @click="playMusic('リラックス・デカフェ')">
          <div class="menu-content">
            <span class="menu-title">リラックス・デカフェ</span>
            <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
          </div>
          <div v-if="selectedMenu === 'リラックス・デカフェ'" class="active-indicator">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
          </div>
        </button>
        <button class="menu-button" @click="playMusic('ジャズ・スペシャル')">
          <div class="menu-content">
            <span class="menu-title">ジャズ・スペシャル</span>
            <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
          </div>
           <div v-if="selectedMenu === 'ジャズ・スペシャル'" class="active-indicator">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
          </div>
        </button>
        <button class="menu-button" @click="playMusic('Lo-Fi・ビター')">
          <div class="menu-content">
            <span class="menu-title">Lo-Fi・ビター</span>
            <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
          </div>
           <div v-if="selectedMenu === 'Lo-Fi・ビター'" class="active-indicator">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
          </div>
        </button>
      </div>
      <div class="controls-container">
        <button @click="togglePlayback" class="control-button" :disabled="!selectedMenu && !isPlaying" :class="{ 'is-disabled': !selectedMenu && !isPlaying }">
          {{ isPlaying ? '■' : '▶' }}
        </button>
        <input type="range" min="0" max="1" step="0.01" :value="volume" @input="handleVolumeChange" class="volume-slider"/>
      </div>

      <div v-if="isPlaying" class="seed-container">
        <p>レコード番号 (シード値):</p>
        <div class="seed-display">
          <span>{{ currentSeed }}</span>
          <button @click="copySeed" title="コピー">📄</button>
        </div>
      </div>
      <div class="seed-input-container">
        <input type="text" v-model="seedInput" placeholder="レコード番号を入力" />
        <button @click="playFromSeed" :disabled="!seedInput">このレコードを聴く</button>
      </div>

    </div>
  </div>
  <AboutModal :isVisible="isModalVisible" @close="closeModal" />
</template>

<style>
body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif; }
.background-container { background-image: url('/bg-main.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
.content-panel { background-color: rgba(255, 255, 255, 0.88); padding: 20px 40px 30px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); backdrop-filter: blur(5px); width: 90%; max-width: 600px; text-align: center; }
.title { color: #363636; font-weight: bold; margin-bottom: 8px; }
.subtitle { color: #555; margin-top: 0; margin-bottom: 30px; }
.menu-container { display: flex; flex-direction: column; gap: 15px; }
.menu-button { background-color: #f5f5f5; border: 1px solid #dbdbdb; border-radius: 4px; padding: 15px 20px; cursor: pointer; transition: all 0.2s ease; text-align: left; display: flex; justify-content: space-between; align-items: center; font-family: inherit; }
.menu-button:hover { background-color: #e8e8e8; border-color: #b5b5b5; transform: translateY(-2px); box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.menu-content { display: flex; flex-direction: column; }
.menu-title { font-size: 1.1em; font-weight: bold; color: #363636; }
.menu-description { font-size: 0.9em; color: #7a7a7a; margin-top: 4px; }
.active-indicator svg { color: #a5a5a5; animation: fadeIn 0.5s ease; }
@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
.controls-container { margin-top: 30px; display: flex; justify-content: center; align-items: center; gap: 20px; }
.control-button { background-color: #363636; color: white; border: none; border-radius: 50%; width: 50px; height: 50px; font-size: 20px; cursor: pointer; display: flex; justify-content: center; align-items: center; transition: all 0.2s ease; }
.control-button:hover { background-color: #555; }
.control-button.is-disabled { background-color: #c5c5c5; cursor: not-allowed; }
.volume-slider { width: 150px; cursor: pointer; }
.footer-link-container { position: fixed; bottom: 15px; right: 20px; z-index: 10; }
.footer-link { font-size: 14px; color: rgba(255, 255, 255, 0.7); text-decoration: none; transition: color 0.2s ease; }
.footer-link:hover { color: rgba(255, 255, 255, 1); }
.seed-container, .seed-input-container { margin-top: 20px; font-size: 14px; color: #555; }
.seed-container p { margin: 0 0 5px 0; }
.seed-display { display: flex; justify-content: center; align-items: center; background: #eee; padding: 5px 10px; border-radius: 4px; }
.seed-display span { font-family: monospace; letter-spacing: 1px; word-break: break-all; }
.seed-display button { background: none; border: none; cursor: pointer; margin-left: 10px; font-size: 16px; }
.seed-input-container { display: flex; gap: 10px; margin-top: 10px; }
.seed-input-container input { flex-grow: 1; border: 1px solid #dbdbdb; border-radius: 4px; padding: 8px; font-family: monospace; }
.seed-input-container button { background-color: #363636; color: white; border: none; border-radius: 4px; padding: 8px 12px; cursor: pointer; transition: background-color 0.2s ease; }
.seed-input-container button:hover { background-color: #555; }
.seed-input-container button:disabled { background-color: #c5c5c5; cursor: not-allowed; }
</style>