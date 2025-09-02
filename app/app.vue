<template>
  <div>
    <div class="background-container">
      <div v-if="isLoading" class="loading-overlay">
        <div class="loading-text">{{ loadingMessage }}</div>
      </div>
      <div v-else class="content-panel">
        <h1 class="title">AI-BGM 喫茶「おとや」</h1>
        <p class="subtitle">本日のBGMをお選びください</p>
        <div class="menu-container">
          <!-- 集中ブレンド -->
          <button class="menu-button" @click="playMusic('集中ブレンド')" :class="{ 'is-active': selectedMenu === '集中ブレンド' }">
            <div class="menu-content">
              <span class="menu-title">集中ブレンド</span>
              <span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span>
            </div>
            <div v-if="selectedMenu === '集中ブレンド'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><rect x="4" y="4" width="16" height="18" rx="2"/><path d="M4 10h16"/></svg>
            </div>
          </button>
          <!-- リラックス・デカフェ -->
          <button class="menu-button" @click="playMusic('リラックス・デカフェ')" :class="{ 'is-active': selectedMenu === 'リラックス・デカフェ' }">
            <div class="menu-content">
              <span class="menu-title">リラックス・デカフェ</span>
              <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
            </div>
            <div v-if="selectedMenu === 'リラックス・デカフェ'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><rect x="4" y="4" width="16" height="18" rx="2"/><path d="M4 10h16"/></svg>
            </div>
          </button>
          <!-- ジャズ・スペシャル -->
          <button class="menu-button" @click="playMusic('ジャズ・スペシャル')" :class="{ 'is-active': selectedMenu === 'ジャズ・スペシャル' }">
            <div class="menu-content">
              <span class="menu-title">ジャズ・スペシャル</span>
              <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
            </div>
            <div v-if="selectedMenu === 'ジャズ・スペシャル'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><rect x="4" y="4" width="16" height="18" rx="2"/><path d="M4 10h16"/></svg>
            </div>
          </button>
          <!-- Lo-Fi・ビター -->
          <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" :class="{ 'is-active': selectedMenu === 'Lo-Fi・ビター' }">
            <div class="menu-content">
              <span class="menu-title">Lo-Fi・ビター</span>
              <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
            </div>
            <div v-if="selectedMenu === 'Lo-Fi・ビター'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><rect x="4" y="4" width="16" height="18" rx="2"/><path d="M4 10h16"/></svg>
            </div>
          </button>
          <!-- ロック・ビート -->
          <button class="menu-button" @click="playMusic('ロック・ビート')" :class="{ 'is-active': selectedMenu === 'ロック・ビート' }">
            <div class="menu-content">
              <span class="menu-title">ロック・ビート</span>
              <span class="menu-description">魂を揺さぶる、力強いリズムと歪んだギターのブレンド。</span>
            </div>
            <div v-if="selectedMenu === 'ロック・ビート'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
            </div>
          </button>
        </div>
        <div class="controls-container">
          <button @click="togglePlayback" class="control-button" :disabled="!selectedMenu && !isPlaying" :class="{ 'is-disabled': !selectedMenu && !isPlaying }">
            {{ isPlaying ? '■' : '▶' }}
          </button>
          <button @click="stopAll" class="control-button" :disabled="!isPlaying">停止</button>
          <div class="volume-control">
            <label for="volume">音量</label>
            <input id="volume" type="range" min="0" max="1" step="0.01" :value="volume" @input="onVolumeChange" />
          </div>
          <div class="seed-display">
            <span>{{ currentSeed }}</span>
            <button @click="copySeed" title="コピー">📋</button>
          </div>
        </div>
        <div class="seed-input-container">
          <input type="text" v-model="seedInput" placeholder="レコード番号を入力" />
          <button @click="playFromSeed" :disabled="!seedInput">このレコードを聴く</button>
        </div>
      </div>
    </div>
    <div class="footer-link-container">
      <a href="#" @click.prevent="openModal" class="footer-link">このBGMについて</a>
    </div>
    <AboutModal :isVisible="isModalVisible" @close="closeModal" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as Tone from 'tone'
import AboutModal from '../components/AboutModal.vue'

type MenuKey = '集中ブレンド' | 'リラックス・デカフェ' | 'ジャズ・スペシャル' | 'Lo-Fi・ビター' | 'ロック・ビート'

const isPlaying = ref(false)
const isLoading = ref(true)
const loadingMessage = ref('コーヒー豆を挽いています...')
const selectedMenu = ref<MenuKey | null>(null)
const currentSeed = ref<string>('')
const seedInput = ref<string>('')
const volume = ref(0.6)

let players: Tone.Players | null = null
let reverb: Tone.Reverb | null = null
let distortion: Tone.Distortion | null = null
let scheduledEvents: (Tone.Loop | Tone.Part | Tone.Sequence)[] = []
let noise: Tone.Noise | null = null

onMounted(async () => {
  const samplePaths = {
    piano: 'piano-c4.wav', 
    bass: 'bass-c1.wav', 
    ride: 'drum-ride.wav', 
    brush: 'drum-brush.wav',
    epiano: 'epiano-c4.wav', 
    kick: 'drum-kick.wav', 
    snare: 'drum-snare.wav', 
    pad: 'pad-cmaj7.wav',
    sax: 'sax-c4.wav', 
    trombone: 'trombone-c3.wav',
    eguitar: 'eguitar-dist-c4.wav', 
    ebass: 'ebass-e1.wav', 
    rockKick: 'rock-kick.wav', 
    rockSnare: 'rock-snare.wav', 
    crash: 'drum-crash.wav',
  };

  try {
    // ユーザー操作を待ってオーディオコンテキストを開始
    await Tone.start();
    
    // エフェクトの準備
    distortion = new Tone.Distortion(0.6).toDestination();
    reverb = new Tone.Reverb({ decay: 2.5, preDelay: 0.01 }).toDestination();
    await reverb.generate();
    Tone.Destination.volume.value = Tone.gainToDb(volume.value);

    // サンプラーの読み込み
    players = await new Promise<Tone.Players>((resolve, reject) => {
        const p = new Tone.Players({
            urls: samplePaths,
            baseUrl: "/",
            onload: () => resolve(p),
            onerror: (err) => reject(err)
        }).toDestination();
    });

    // 全体にリバーブを差す（後で個別に直結し直すトラックもある）
    players.connect(reverb);

    // ギターは歪みへチェーン
    const eguitarPlayer = players.player('eguitar');
    eguitarPlayer.chain(distortion!);

    // ドラムは基本ドライで
    ['kick','snare','rockKick','rockSnare','crash','ride','brush'].forEach((drum) => {
      const player = players!.player(drum as keyof typeof samplePaths);
      player.toDestination();
    });

    isLoading.value = false;
    loadingMessage.value = '';
  } catch (e) {
    console.error(e);
    loadingMessage.value = '音源の読み込みに失敗しました…';
  }
});

onBeforeUnmount(() => {
  stopAll();
  players?.dispose();
  reverb?.dispose();
  distortion?.dispose();
  noise?.dispose();
});

function onVolumeChange(event: Event) { 
  const newVolume = parseFloat((event.target as HTMLInputElement).value); 
  volume.value = newVolume; 
  Tone.Destination.volume.value = Tone.gainToDb(newVolume); 
};

const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const isModalVisible = ref(false);

async function playMusic(menu: MenuKey) {
  if (isLoading.value) return;
  if (!players) return;

  // 既存のイベントを停止・破棄
  stopGenerators();
  selectedMenu.value = menu;
  currentSeed.value = generateSeed();

  // グローバル設定
  Tone.Transport.bpm.value = 88;
  Tone.Transport.swingSubdivision = '8n';
  Tone.Transport.swing = 0.2;

  switch (menu) {
    case '集中ブレンド':
      scheduledEvents = generateFocusBlend();
      break;
    case 'リラックス・デカフェ':
      scheduledEvents = generateRelaxDecaf();
      break;
    case 'ジャズ・スペシャル':
      scheduledEvents = generateJazzSpecial();
      break;
    case 'Lo-Fi・ビター':
      scheduledEvents = generateLofiBitter();
      break;
    case 'ロック・ビート':
      scheduledEvents = generateRockBeat();
      break;
  }

  // 再生
  if (!isPlaying.value) {
    Tone.Transport.start();
    isPlaying.value = true;
  }
}

function togglePlayback() {
  if (isPlaying.value) {
    Tone.Transport.pause();
    isPlaying.value = false;
  } else {
    Tone.Transport.start();
    isPlaying.value = true;
  }
}

function stopAll() {
  if (!isPlaying.value && scheduledEvents.length === 0) return;
  stopGenerators();
  Tone.Transport.stop();
  Tone.Transport.cancel(0);

  players?.stopAll();
  isPlaying.value = false;
  selectedMenu.value = null;
}

function stopGenerators() {
  for (const ev of scheduledEvents) {
    ev.stop(0);
    ev.dispose();
  }
  scheduledEvents = [];
  if (noise) {
    noise.stop();
    noise.dispose();
    noise = null;
  }
}

/* =========================
   生成系
   ========================= */

// 集中ブレンド：レインライド + パッド
function generateFocusBlend() {
  const evs: (Tone.Loop | Tone.Part | Tone.Sequence)[] = [];
  if (!players) return evs;

  const ride = players.player('ride');
  ride.volume.value = -12;

  const pad = players.player('pad');
  pad.volume.value = -10;

  const rideLoop = new Tone.Loop((time) => {
    ride.start(time, 0, '4n');
  }, '4n').start(0);

  const padLoop = new Tone.Loop((time) => {
    pad.start(time, 0, '1m');
  }, '1m').start(0);

  evs.push(rideLoop, padLoop);
  return evs;
}

// リラックス・デカフェ：エレピ + ブラシ + ノイズ
function generateRelaxDecaf() {
  const evs: (Tone.Loop | Tone.Part | Tone.Sequence)[] = [];
  if (!players) return evs;

  const ep = players.player('epiano');
  ep.volume.value = -8;

  const br = players.player('brush');
  br.volume.value = -14;

  // ノイズ
  noise = new Tone.Noise('pink');
  const noiseFilter = new Tone.Filter(1000, 'lowpass').toDestination();
  noise.connect(noiseFilter);
  (noise as any).volume.value = -24;
  noise.start();

  const epLoop = new Tone.Loop((time) => {
    ep.start(time, 0, '2n');
  }, '2n').start(0);

  const brLoop = new Tone.Loop((time) => {
    br.start(time, 0, '2n');
  }, '2n').start('4n');

  evs.push(epLoop, brLoop);
  return evs;
}

// ジャズ・スペシャル：ピアノ分散和音 + スウィング
function generateJazzSpecial() {
  const evs: (Tone.Loop | Tone.Part | Tone.Sequence)[] = [];
  if (!players) return evs;

  Tone.Transport.bpm.value = 120;
  Tone.Transport.swing = 0.35;
  Tone.Transport.swingSubdivision = '8n';

  const p = players.player('piano');
  p.volume.value = -6;

  // シンプルなウォーキング風
  const bass = players.player('bass');
  bass.volume.value = -10;

  const chordPart = new Tone.Part((time, step: number) => {
    p.start(time, 0, '8n');
  }, Array.from({ length: 32 }, (_, i) => [i * Tone.Time('8n').toSeconds(), i])).start(0);

  const bassSeq = new Tone.Sequence((time) => {
    bass.start(time, 0, '8n');
  }, ['C2', 'D2', 'E2', 'G1'].map(() => 0), '4n').start(0);

  evs.push(chordPart, bassSeq);
  return evs;
}

// Lo-Fi・ビター：ビート + パッド
function generateLofiBitter() {
  const evs: (Tone.Loop | Tone.Part | Tone.Sequence)[] = [];
  if (!players) return evs;

  const k = players.player('kick');
  const s = players.player('snare');
  const p = players.player('pad');

  k.volume.value = -8;
  s.volume.value = -10;
  p.volume.value = -12;

  const beat = new Tone.Sequence((time, step) => {
    if (step % 2 === 0) k.start(time, 0, '8n');
    if (step % 4 === 2) s.start(time, 0, '8n');
  }, Array.from({ length: 16 }, (_, i) => i), '16n').start(0);

  const padL = new Tone.Loop((time) => {
    p.start(time, 0, '1m');
  }, '2m').start(0);

  evs.push(beat, padL);
  return evs;
}

// ロック・ビート：歪みギター + ロックドラム
function generateRockBeat() {
  const evs: (Tone.Loop | Tone.Part | Tone.Sequence)[] = [];
  if (!players) return evs;

  Tone.Transport.bpm.value = 140;
  Tone.Transport.swing = 0;
  Tone.Transport.swingSubdivision = '8n';

  const rk = players.player('rockKick');
  const rs = players.player('rockSnare');
  const cr = players.player('crash');
  const eg = players.player('eguitar');
  const eb = players.player('ebass');

  rk.volume.value = -8;
  rs.volume.value = -8;
  cr.volume.value = -10;
  eg.volume.value = -6;
  eb.volume.value = -10;

  const drumSeq = new Tone.Sequence((time, step) => {
    if (step % 4 === 0) rk.start(time, 0, '8n');
    if (step % 4 === 2) rs.start(time, 0, '8n');
    if (step === 0) cr.start(time, 0, '4n');
  }, Array.from({ length: 16 }, (_, i) => i), '16n').start(0);

  const riff = new Tone.Sequence((time, step) => {
    if (step % 2 === 0) eg.start(time, 0, '8n');
  }, Array.from({ length: 16 }, (_, i) => i), '8n').start(0);

  const bassLoop = new Tone.Loop((time) => {
    eb.start(time, 0, '8n');
  }, '8n').start(0);

  evs.push(drumSeq, riff, bassLoop);
  return evs;
}

/* =========================
   シード関連
   ========================= */

function generateSeed() {
  // YYYYMMDD-HHMM-XXXX のような簡易シード
  const now = new Date();
  const base = `${now.getFullYear()}${String(now.getMonth()+1).padStart(2,'0')}${String(now.getDate()).padStart(2,'0')}-${String(now.getHours()).padStart(2,'0')}${String(now.getMinutes()).padStart(2,'0')}`;
  const rand = Math.random().toString(36).slice(2, 6).toUpperCase();
  return `${base}-${rand}`;
}

function playFromSeed() {
  if (!seedInput.value) return;
  currentSeed.value = seedInput.value;
  if (selectedMenu.value) {
    playMusic(selectedMenu.value);
  }
}

async function copySeed() {
  try {
    await navigator.clipboard.writeText(currentSeed.value);
  } catch (e) {
    console.error('Clipboard copy failed', e);
  }
}
</script>

<style scoped>
/* 背景・配色 */
.background-container {
  min-height: 100vh;
  background: radial-gradient(ellipse at top, #3b2f2f 0%, #2b1f1a 70%), url('/watercolor-cafe.jpg') center/cover no-repeat;
  color: #f3e6d0;
}

/* ローディング */
.loading-overlay {
  position: fixed;
  inset: 0;
  display: grid;
  place-items: center;
  backdrop-filter: blur(4px);
}
.loading-text { font-size: 1.1rem; color: #f3e6d0; }

/* 内容パネル */
.content-panel {
  max-width: 960px; margin: 0 auto; padding: 32px 20px;
}

/* タイトル */
.title { font-family: "YuMincho", "Hiragino Mincho ProN", serif; font-size: 2rem; margin-bottom: 8px; }
.subtitle { opacity: 0.85; margin-bottom: 20px; }

/* メニュー */
.menu-container { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 12px; }
.menu-button {
  background: rgba(50, 35, 30, 0.7);
  border: 1px solid rgba(255,255,255,0.06);
  padding: 16px; border-radius: 14px; text-align: left; cursor: pointer;
  transition: transform .12s ease, background .12s ease, border-color .12s ease;
}
.menu-button:hover { transform: translateY(-2px); background: rgba(60, 40, 30, 0.75); border-color: rgba(255,255,255,0.1); }
.menu-button.is-active { outline: 2px solid rgba(255, 214, 153, 0.6); }

.menu-content { display: grid; gap: 4px; }
.menu-title { font-weight: 600; }
.menu-description { font-size: 0.9rem; opacity: 0.8; }
.active-indicator { margin-top: 6px; opacity: 0.8; }

/* コントロール郡 */
.controls-container { display: flex; gap: 10px; align-items: center; margin-top: 20px; flex-wrap: wrap; }
.control-button {
  background: #5b4236; color: #f3e6d0; border: none; padding: 10px 14px; border-radius: 12px;
}
.control-button.is-disabled { opacity: 0.5; cursor: not-allowed; }

.volume-control { display: inline-flex; gap: 8px; align-items: center; }
.volume-control input[type="range"] { width: 160px; }

/* シード表示 */
.seed-display {
  margin-left: auto; display: inline-flex; gap: 8px; align-items: center;
  background: rgba(0,0,0,0.25); padding: 6px 10px; border-radius: 10px;
}
.seed-input-container { margin-top: 12px; display: flex; gap: 8px; }

/* フッターリンク */
.footer-link-container { text-align: center; margin: 28px 0; }
.footer-link { color: #f3e6d0; opacity: 0.85; text-decoration: underline; }
</style>
