<script setup lang="ts">
import { ref, onMounted } from 'vue';
import seedrandom from 'seedrandom';
import AboutModal from '../components/AboutModal.vue';

// --- 状態変数 ---
const selectedMenu = ref<string | null>(null);
const isPlaying = ref(false);
const volume = ref(0.5);
const isModalVisible = ref(false);
const currentSeed = ref<string>('');
const seedInput = ref<string>('');
const isLoading = ref<boolean>(true);
const loadingMessage = ref<string>('コーヒー豆を挽いています...');

// --- サンプル音源バッファ ---
const pianoSample = ref<AudioBuffer | null>(null);

// --- Web Audio API関連 ---
let audioContext: AudioContext;
let masterGainNode: GainNode;
let activeNodes: AudioNode[] = [];
let soundInterval: any;

onMounted(async () => {
  try {
    audioContext = new AudioContext();
    loadingMessage.value = 'ピアノを調律しています...';
    const response = await fetch('/piano-c4.wav');
    const arrayBuffer = await response.arrayBuffer();
    pianoSample.value = await audioContext.decodeAudioData(arrayBuffer);
    loadingMessage.value = '準備ができました';
    isLoading.value = false;
  } catch (error) {
    loadingMessage.value = '楽器の準備に失敗しました。ページを再読み込みしてください。';
    console.error("Error loading audio sample:", error);
  }
});

const playMusic = (menuName: string, seed?: string) => {
  if (isLoading.value || (audioContext && audioContext.state === 'suspended')) {
    audioContext.resume();
  }
  if (isPlaying.value) stopMusic();
  const randomPart = seed || Date.now().toString(36) + Math.random().toString(36).substring(2);
  currentSeed.value = `${menuName}:${randomPart}`;
  const rng = seedrandom(randomPart);
  masterGainNode = audioContext.createGain();
  masterGainNode.gain.setValueAtTime(volume.value, audioContext.currentTime);
  masterGainNode.connect(audioContext.destination);

  switch (menuName) {
    case 'ジャズ・スペシャル': createJazzSound(rng); break;
    default: 
      alert('現在、ジャズ・スペシャルの調律のみ完了しております。');
      return;
  }
  isPlaying.value = true;
  selectedMenu.value = menuName;
};
const stopMusic = () => { if (!isPlaying.value) return; clearInterval(soundInterval); clearTimeout(soundInterval); activeNodes.forEach(node => { if (node instanceof OscillatorNode || node instanceof AudioBufferSourceNode) { try { 'stop' in node && node.stop(); } catch (e) {} } node.disconnect(); }); activeNodes = []; isPlaying.value = false; };
const togglePlayback = () => { if (isPlaying.value) { stopMusic(); } else { if (selectedMenu.value && currentSeed.value) { const [menuName, seed] = currentSeed.value.split(':'); if (menuName && seed) playMusic(menuName, seed); } } };
const handleVolumeChange = (event: Event) => { const newVolume = parseFloat((event.target as HTMLInputElement).value); volume.value = newVolume; if (masterGainNode) { masterGainNode.gain.linearRampToValueAtTime(newVolume, audioContext.currentTime + 0.1); } };
const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const copySeed = () => { navigator.clipboard.writeText(currentSeed.value); };
const playFromSeed = () => { if (seedInput.value) { const [menuName, seed] = seedInput.value.split(':'); const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター']; if (menuName && seed && validMenus.includes(menuName)) { playMusic(menuName, seed); } else { alert('レコード番号の形式が正しくないか、存在しないジャンルです。'); } } };

const createJazzSound = (rng: () => number) => {
  if (!pianoSample.value) return;

  let beat = 0; const tempo = 100 + rng() * 20; const intervalTime = 60000 / tempo;
  const pianoScale = [-1200, -1000, -800, -700, -500, -300, 0, 200, 300, 500, 700, 800, 1000, 1200];
  const bassPatterns = [ [130.8, 174.6, 196.0, 130.8], [130.8, 196.0, 174.6, 130.8] ];
  const bassLine = bassPatterns[Math.floor(rng() * bassPatterns.length)];
  let nextPianoTime = 0;

  const playPianoNote = (startTime: number, detune: number, duration: number, vol: number) => {
    const source = audioContext.createBufferSource();
    source.buffer = pianoSample.value;
    source.detune.value = detune;
    source.loop = true;
    source.loopStart = 0.1;
    source.loopEnd = 2.5;
    const gain = audioContext.createGain();
    gain.gain.setValueAtTime(0, startTime);
    gain.gain.linearRampToValueAtTime(vol, startTime + 0.01);
    gain.gain.setValueAtTime(vol, startTime + duration - 0.1);
    gain.gain.linearRampToValueAtTime(0, startTime + duration);
    source.connect(gain);
    gain.connect(masterGainNode);
    source.start(startTime);
    source.stop(startTime + duration);
    activeNodes.push(source, gain);
  };
  
  const playBassNote = (freq: number, startTime: number, duration: number, vol: number) => { const osc = audioContext.createOscillator(); osc.type = 'sine'; const gain = audioContext.createGain(); osc.frequency.setValueAtTime(freq, startTime); gain.gain.setValueAtTime(0, startTime); gain.gain.linearRampToValueAtTime(vol, startTime + 0.01); gain.gain.linearRampToValueAtTime(0, startTime + duration); osc.connect(gain); gain.connect(masterGainNode); osc.start(startTime); osc.stop(startTime + duration); activeNodes.push(osc, gain); };
  
  // ★★★ ここが修正箇所です ★★★
  const playHiHat = (startTime: number, vol: number) => {
    const noise = audioContext.createBufferSource();
    const bufferSize = audioContext.sampleRate * 0.1;
    const buffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) data[i] = rng() * 2 - 1;
    noise.buffer = buffer;
    const filter = audioContext.createBiquadFilter();
    filter.type = 'highpass';
    filter.frequency.value = 5000;
    const gain = audioContext.createGain();
    gain.gain.setValueAtTime(0, startTime);
    gain.gain.linearRampToValueAtTime(vol, startTime + 0.01);
    gain.gain.linearRampToValueAtTime(0, startTime + 0.05); // a を追加
    noise.connect(filter);
    filter.connect(gain);
    gain.connect(masterGainNode);
    noise.start(startTime);
    activeNodes.push(noise, filter, gain);
  };
  
  const sequencer = () => {
    const now = audioContext.currentTime;
    playHiHat(now, 0.1); playHiHat(now + (intervalTime / 1000) * 0.66, 0.05);
    if (beat % 4 === 0 && bassLine) { const bassNote = bassLine[Math.floor(beat / 4) % bassLine.length]; if (bassNote) playBassNote(bassNote, now, intervalTime / 1000 * 1.5, 0.3); }
    
    if (now >= nextPianoTime) {
      const detuneValue = pianoScale[Math.floor(rng() * pianoScale.length)];
      if (detuneValue !== undefined) {
        const duration = (intervalTime / 1000) * (0.5 + rng() * 3.5); 
        playPianoNote(now, detuneValue, duration, 0.25);
      }
      nextPianoTime = now + (intervalTime / 1000) * (0.5 + rng() * 1.5);
    }
    beat = (beat + 1) % 16;
  };
  soundInterval = setInterval(sequencer, intervalTime);
};
</script>

<template>
  <div class="background-container">
    <div v-if="isLoading" class="loading-overlay">
      <div class="loading-text">{{ loadingMessage }}</div>
    </div>
    <div v-else class="content-panel">
      <h1 class="title">AI-BGM 喫茶「おとや」</h1>
      <p class="subtitle">本日のBGMをお選びください</p>
      <div class="menu-container">
        <button class="menu-button" @click="playMusic('集中ブレンド')" disabled><div class="menu-content"><span class="menu-title">集中ブレンド</span><span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span></div><div v-if="selectedMenu === '集中ブレンド'" class="active-indicator"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg></div></button>
        <button class="menu-button" @click="playMusic('リラックス・デカフェ')" disabled><div class="menu-content"><span class="menu-title">リラックス・デカフェ</span><span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span></div><div v-if="selectedMenu === 'リラックス・デカフェ'" class="active-indicator"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg></div></button>
        <button class="menu-button" @click="playMusic('ジャズ・スペシャル')"><div class="menu-content"><span class="menu-title">ジャズ・スペシャル</span><span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span></div><div v-if="selectedMenu === 'ジャズ・スペシャル'" class="active-indicator"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg></div></button>
        <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" disabled><div class="menu-content"><span class="menu-title">Lo-Fi・ビター</span><span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span></div><div v-if="selectedMenu === 'Lo-Fi・ビター'" class="active-indicator"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg></div></button>
      </div>
      <div class="controls-container"><button @click="togglePlayback" class="control-button" :disabled="!selectedMenu && !isPlaying" :class="{ 'is-disabled': !selectedMenu && !isPlaying }">{{ isPlaying ? '■' : '▶' }}</button><input type="range" min="0" max="1" step="0.01" :value="volume" @input="handleVolumeChange" class="volume-slider"/></div>
      <div v-if="isPlaying" class="seed-container"><p>レコード番号 (シード値):</p><div class="seed-display"><span>{{ currentSeed }}</span><button @click="copySeed" title="コピー">📄</button></div></div>
      <div class="seed-input-container"><input type="text" v-model="seedInput" placeholder="レコード番号を入力" /><button @click="playFromSeed" :disabled="!seedInput">このレコードを聴く</button></div>
    </div>
  </div>
  <AboutModal :isVisible="isModalVisible" @close="closeModal" />
</template>

<style>
/* (Styleは変更なし) */
body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif; }
.background-container { background-image: url('/bg-main.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
.content-panel { background-color: rgba(255, 255, 255, 0.88); padding: 20px 40px 30px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); backdrop-filter: blur(5px); width: 90%; max-width: 600px; text-align: center; }
.title { color: #363636; font-weight: bold; margin-bottom: 8px; }
.subtitle { color: #555; margin-top: 0; margin-bottom: 30px; }
.menu-container { display: flex; flex-direction: column; gap: 15px; }
.menu-button { background-color: #f5f5f5; border: 1px solid #dbdbdb; border-radius: 4px; padding: 15px 20px; cursor: pointer; transition: all 0.2s ease; text-align: left; display: flex; justify-content: space-between; align-items: center; font-family: inherit; }
.menu-button:hover { background-color: #e8e8e8; border-color: #b5b5b5; transform: translateY(-2px); box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.menu-button:disabled { cursor: not-allowed; opacity: 0.5; }
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
.loading-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background-color: rgba(0,0,0,0.5); display: flex; justify-content: center; align-items: center; z-index: 2000; }
.loading-text { color: white; font-size: 1.2em; }
</style>