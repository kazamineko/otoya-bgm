<script setup lang="ts">
import { ref, onUnmounted } from 'vue';
import seedrandom from 'seedrandom';
// Tone.jsの型情報のみを静的にインポートする
import type * as ToneType from 'tone';
import AboutModal from '../components/AboutModal.vue';
import SoundCheckModal from '../components/SoundCheckModal.vue';

// --- 状態変数 ---
const selectedMenu = ref<string | null>(null);
const isPlaying = ref(false);
const volume = ref(0.5);
const isModalVisible = ref(false);
const isSoundCheckModalVisible = ref(false);
const instrumentList = ref<string[]>([]);
const currentSeed = ref<string>('');
const seedInput = ref<string>('');
const isLoading = ref<boolean>(false);
const loadingMessage = ref<string>('');

// --- Tone.js関連 ---
let Tone: typeof ToneType | null = null;
let players: ToneType.Players | null = null;
let distortion: ToneType.Distortion | null = null;
let reverb: ToneType.Reverb | null = null;
let chorus: ToneType.Chorus | null = null;
let delay: ToneType.PingPongDelay | null = null;
let masterComp: ToneType.Compressor | null = null;
let panners: { [key: string]: ToneType.Panner } = {};
const scheduledEvents: (ToneType.Loop | ToneType.Part | ToneType.Sequence | ToneType.LFO)[] = [];
let noise: ToneType.Noise | null = null;
let isAudioInitialized = ref(false);

/**
 * ユーザーの初回アクション時に音声関連のすべてを初期化する関数
 */
const initializeAudio = async () => {
  isLoading.value = true;
  loadingMessage.value = '喫茶店の準備をしています...';

  const samplePaths = {
    piano: 'piano-c4.wav', bass: 'bass-c1.wav', ride: 'drum-ride.wav', brush: 'drum-brush.wav',
    epiano: 'epiano-c4.wav', kick: 'drum-kick.wav', snare: 'drum-snare.wav', pad: 'pad-cmaj7.wav',
    sax: 'sax-c4.wav', trombone: 'trombone-c3.wav',
    eguitar: 'eguitar-dist-c4.wav', ebass: 'ebass-e1.wav', rockKick: 'rock-kick.wav', rockSnare: 'rock-snare.wav', crash: 'drum-crash.wav',
  };

  try {
    Tone = await import('tone');
    await Tone.start();
    loadingMessage.value = '店内の響きを調整しています...';
    
    masterComp = new Tone.Compressor(-12, 3).toDestination();
    Tone.Destination.volume.value = Tone.gainToDb(volume.value);

    distortion = new Tone.Distortion(0.6).connect(masterComp);
    reverb = new Tone.Reverb({ decay: 2.5, preDelay: 0.01, wet: 0.3 }).connect(masterComp);
    await reverb.generate();
    chorus = new Tone.Chorus(4, 2.5, 0.7).connect(masterComp);
    delay = new Tone.PingPongDelay("8n", 0.2).connect(masterComp);

    loadingMessage.value = '楽器を準備しています...';
    players = await new Promise<ToneType.Players>((resolve, reject) => {
        const p = new Tone!.Players({
            urls: samplePaths,
            baseUrl: "/",
            onload: () => resolve(p),
            onerror: (error: Error) => reject(error),
        });
    });

    instrumentList.value = Object.keys(samplePaths); // 【修正】samplePathsのキーから楽器リストを生成

    const instrumentsToPan = ['piano', 'epiano', 'sax', 'trombone', 'eguitar', 'ebass'];
    instrumentsToPan.forEach(inst => {
      if (players!.has(inst)) {
        panners[inst] = new Tone!.Panner(0).connect(masterComp!);
        players!.player(inst).connect(panners[inst]!);
      }
    });
    const drumInstruments = ['kick', 'snare', 'rockKick', 'rockSnare', 'crash', 'ride', 'brush', 'pad', 'bass'];
    drumInstruments.forEach(drum => {
        if(players!.has(drum)) {
            players!.player(drum).connect(masterComp!);
        }
    });

    isAudioInitialized.value = true;
    loadingMessage.value = '準備ができました';

  } catch (error: any) {
    loadingMessage.value = `エラーが発生しました: ${error.message}`;
    console.error("Error setting up Tone.js:", error);
    isAudioInitialized.value = false;
  } finally {
    isLoading.value = false;
  }
};

onUnmounted(() => {
    stopMusic();
});

const playMusic = async (menuName: string, seed?: string) => {
  if (!isAudioInitialized.value) {
    await initializeAudio();
    if (!isAudioInitialized.value) return;
  }
  if (isPlaying.value) stopMusic();
  const randomPart = seed || Date.now().toString(36) + Math.random().toString(36).substring(2);
  currentSeed.value = `${menuName}:${randomPart}`;
  const rng = seedrandom(randomPart);
  switch (menuName) {
    case '集中ブレンド': createConcentrationSound(rng); break;
    case 'リラックス・デカフェ': createRelaxSound(rng); break;
    case 'ジャズ・スペシャル': createJazzSound(rng); break;
    case 'Lo-Fi・ビター': createLoFiSound(rng); break;
    case 'ロック・ビート': createRockSound(rng); break;
  }
  Tone!.Transport.start();
  isPlaying.value = true;
  selectedMenu.value = menuName;
};

const stopMusic = () => {
  if (!isPlaying.value || !Tone) return;
  Tone.Transport.stop();
  Tone.Transport.cancel(0);
  scheduledEvents.forEach(event => { event.stop(0); event.dispose(); });
  scheduledEvents.length = 0;
  if (noise) { noise.stop(0); noise.dispose(); noise = null; }
  if (players) { players.stopAll(); }
  isPlaying.value = false;
};

const togglePlayback = async () => { if (isPlaying.value) { stopMusic(); } else { if (selectedMenu.value) { const [menuName, seed] = currentSeed.value.split(':'); if (menuName && seed) await playMusic(menuName, seed); } } };
const handleVolumeChange = (event: Event) => { const newVolume = parseFloat((event.target as HTMLInputElement).value); volume.value = newVolume; if (isAudioInitialized.value) { Tone!.Destination.volume.value = Tone!.gainToDb(newVolume); } };
const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };

const openSoundCheckModal = () => { isSoundCheckModalVisible.value = true; };
const closeSoundCheckModal = () => { isSoundCheckModalVisible.value = false; };
const handlePlaySoundCheck = async (instrumentName: string) => {
  if (!isAudioInitialized.value) {
    await initializeAudio();
    if (!isAudioInitialized.value) {
        alert('音源の初期化に失敗しました。ページをリロードしてください。');
        return;
    }
  }

  if (players && players.has(instrumentName)) {
    players.player(instrumentName).start();
  }
};

const copySeed = () => { if(currentSeed.value) navigator.clipboard.writeText(currentSeed.value); };
const playFromSeed = async () => { if (seedInput.value) { const [menuName, seed] = seedInput.value.split(':'); const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター', 'ロック・ビート']; if (menuName && seed && validMenus.includes(menuName)) { await playMusic(menuName, seed); } else { alert('レコード番号の形式が正しくないか、存在しないジャンルです。'); } } };

// --- 音楽生成関数 --- (変更なし)

const createConcentrationSound = (rng: () => number) => {
    if (!Tone || !masterComp) return;
    if (!noise) noise = new Tone.Noise("pink").start();
    const filter = new Tone.Filter(800, "lowpass").connect(masterComp);
    noise.connect(filter);
    const loop = new Tone.Loop((time) => { filter.frequency.rampTo(600 + rng() * 400, 4, time); }, "4m").start(0);
    scheduledEvents.push(loop);
};

const createRelaxSound = (rng: () => number) => {
    if (!players || !reverb) return;
    const padPlayer = players.player('pad');
    padPlayer.loop = true;
    padPlayer.connect(reverb);
    padPlayer.start();
};

const createLoFiSound = (rng: () => number) => {
    if (!players || !Tone || !chorus) return;
    Tone.Transport.bpm.value = 80 + rng() * 15;
    
    const epiano = players.player('epiano');
    epiano.chain(panners['epiano']!, chorus);
    panners['epiano']!.pan.value = -0.2;

    const chords = [[-1200, -700, -400], [-1700, -1200, -900]];
    let currentChord = chords[Math.floor(rng()*2)]!;

    const kickPlayer = players.player('kick');
    const snarePlayer = players.player('snare');

    const kickSeq = new Tone.Sequence((time, note) => { 
        if (note) {
            kickPlayer.volume.value = Tone!.gainToDb(0.7 + rng() * 0.3);
            kickPlayer.start(time, 0, "8n");
        }
    }, [1,0,1,0,1,0,1,0], "8n").start(0);

    const snareSeq = new Tone.Sequence((time, note) => { 
        if (note) {
            snarePlayer.volume.value = Tone!.gainToDb(0.6 + rng() * 0.2);
            snarePlayer.start(time, 0, "8n");
        }
    }, [0,1,0,1], "4n").start(0);
    
    const chordLoop = new Tone.Loop((time) => {
        currentChord.forEach((detune, i) => {
            epiano.volume.value = Tone!.gainToDb(0.5 + rng() * 0.2);
            epiano.playbackRate = Math.pow(2, detune / 1200);
            epiano.start(time + i * 0.05, 0, "2n");
        });
    }, "1n").start(0);

    scheduledEvents.push(kickSeq, snareSeq, chordLoop);
};

const createRockSound = (rng: () => number) => {
    if (!players || !Tone || !distortion) return;
    Tone.Transport.bpm.value = 125 + rng() * 20;

    const guitar = players.player('eguitar');
    guitar.chain(panners['eguitar']!, distortion);
    const bass = players.player('ebass');
    panners['eguitar']!.pan.value = 0.3;
    panners['ebass']!.pan.value = -0.3;
    let isSolo = false;

    const rockKickPlayer = players.player('rockKick');
    const rockSnarePlayer = players.player('rockSnare');
    const crashPlayer = players.player('crash');
    
    const drumPart = new Tone.Part((time, value) => {
        if (value.kick) {
            rockKickPlayer.volume.value = Tone!.gainToDb(0.9 + rng() * 0.1);
            rockKickPlayer.start(time, 0, "8n");
        }
        if (value.snare) {
            rockSnarePlayer.volume.value = Tone!.gainToDb(value.accent ? 0.9 : 0.7);
            rockSnarePlayer.start(time, 0, "8n");
        }
        if (value.crash) {
            crashPlayer.volume.value = Tone!.gainToDb(0.6);
            crashPlayer.start(time, 0, "2n");
        }
    }, [ { time: '0:0', kick: true, crash: true }, { time: '0:1', kick: true }, { time: '0:2', snare: true, accent: true }, { time: '0:3', kick: true }, { time: '1:0', kick: true }, { time: '1:1', snare: true }, { time: '1:2', kick: true }, { time: '1:3', snare: true, accent: true } ]).start(0);
    drumPart.loop = true; drumPart.loopEnd = '2m';

    const bassPart = new Tone.Sequence((time, note) => { 
        if (note !== null) {
            bass.volume.value = Tone!.gainToDb(0.8 + rng() * 0.2);
            bass.playbackRate = Math.pow(2, note/1200);
            bass.start(time, 0, "8n"); 
        }
    }, [-12, null, -5, -5, 0, null, -2, -2], "8n").start(0);

    const guitarPart = new Tone.Part((time, value) => {
        if (isSolo) {
            const note = [-12, -9, -7, -5, 0][Math.floor(rng()*5)]!;
            guitar.volume.value = Tone!.gainToDb(0.8);
            guitar.playbackRate = Math.pow(2, note / 1200);
            guitar.start(time, 0, "16n");
        } else {
            guitar.volume.value = Tone!.gainToDb(0.7);
            guitar.playbackRate = Math.pow(2, value.detune / 1200);
            guitar.start(time, 0, value.dur);
        }
    }, [{time: '0', dur: '2n', detune: -12}, {time: '1', dur: '2n', detune: -5}]).start(0);
    guitarPart.loop = true; guitarPart.loopEnd = '2m';
    
    const soloToggle = new Tone.Loop(() => { isSolo = !isSolo; }, '8m').start('4m');
    scheduledEvents.push(drumPart, bassPart, guitarPart, soloToggle);
};

const createJazzSound = (rng: () => number) => {
    if (!players || !Tone || !reverb || !delay) return;
    Tone.Transport.bpm.value = 100 + rng() * 20;
    Tone.Transport.swing = 0.05;

    const piano = players.player('piano');
    const bass = players.player('bass');
    const ride = players.player('ride');
    const sax = players.player('sax');
    
    panners['piano']!.pan.rampTo(-0.4, 0.1);
    panners['sax']!.pan.rampTo(0.4, 0.1);
    piano.connect(reverb);
    sax.connect(delay);
    let isSolo = false;

    const chords = [
        { time: '0:0', chord: ['D3', 'F4', 'A4', 'C5'] }, { time: '2:0', chord: ['G3', 'F4', 'B4', 'D5'] },
        { time: '4:0', chord: ['C3', 'E4', 'G4', 'B4'] }, { time: '6:0', chord: ['A3', 'G4', 'C5', 'E5'] }, 
    ];
    const pianoPart = new Tone.Part((time, value) => {
        if (isSolo && rng() < 0.6) return;
        value.chord.forEach((note) => {
            const vel = isSolo ? 0.3 + rng() * 0.2 : 0.4 + rng() * 0.2;
            piano.volume.value = Tone!.gainToDb(vel);
            piano.playbackRate = Tone!.Frequency(note).toFrequency() / Tone!.Frequency('C4').toFrequency();
            piano.start(time + rng() * 0.05, 0, "2n");
        });
    }, chords).start(0);
    pianoPart.loop = true; pianoPart.loopEnd = '8m';

    const bassPart = new Tone.Sequence((time, note) => {
        bass.volume.value = Tone!.gainToDb(0.7 + rng() * 0.2);
        bass.playbackRate = Tone!.Frequency(note).toFrequency() / Tone!.Frequency('C1').toFrequency();
        bass.start(time, 0, "4n");
    }, ['D1', 'A1', 'G1', 'D2', 'C1', 'G1', 'A1', 'E2'], "2n").start(0);
    bassPart.loop = true;

    const ridePart = new Tone.Loop((time) => { 
        ride.volume.value = Tone!.gainToDb(0.4 + rng() * 0.2);
        ride.start(time, 0, "4n");
    }, "4n").start(0);

    const saxPart = new Tone.Sequence((time, note) => {
        if (!isSolo || !note) return;
        sax.volume.value = Tone!.gainToDb(0.6 + rng() * 0.2);
        sax.playbackRate = Tone!.Frequency(note).toFrequency() / Tone!.Frequency('C4').toFrequency();
        sax.start(time, 0, "8n");
    }, ['C5', 'A4', 'G4', null, 'E4', 'D4', null, 'G4'], "8n").start(0);
    saxPart.loop = true;
    
    const soloToggle = new Tone.Loop(() => { isSolo = !isSolo; }, '8m').start('8m');
    scheduledEvents.push(pianoPart, bassPart, ridePart, saxPart, soloToggle);
};
</script>

<template>
  <div>
    <div class="background-container">
      <div v-if="isLoading" class="loading-overlay">
        <div class="loading-text">{{ loadingMessage }}</div>
      </div>
      <div class="content-panel">
        <h1 class="title">AI-BGM 喫茶「おとや」</h1>
        <p class="subtitle">本日のBGMをお選びください</p>
        <div class="menu-container">
          <button class="menu-button" @click="playMusic('集中ブレンド')" :class="{ 'is-active': selectedMenu === '集中ブレンド' }">
            <div class="menu-content">
              <span class="menu-title">集中ブレンド</span>
              <span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span>
            </div>
            <div v-if="selectedMenu === '集中ブレンド' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('リラックス・デカフェ')" :class="{ 'is-active': selectedMenu === 'リラックス・デカフェ' }">
            <div class="menu-content">
              <span class="menu-title">リラックス・デカフェ</span>
              <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
            </div>
            <div v-if="selectedMenu === 'リラックス・デカフェ' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ジャズ・スペシャル')" :class="{ 'is-active': selectedMenu === 'ジャズ・スペシャル' }">
            <div class="menu-content">
              <span class="menu-title">ジャズ・スペシャル</span>
              <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
            </div>
            <div v-if="selectedMenu === 'ジャズ・スペシャル' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" :class="{ 'is-active': selectedMenu === 'Lo-Fi・ビター' }">
            <div class="menu-content">
              <span class="menu-title">Lo-Fi・ビター</span>
              <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
            </div>
            <div v-if="selectedMenu === 'Lo-Fi・ビター' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ロック・ビート')" :class="{ 'is-active': selectedMenu === 'ロック・ビート' }">
            <div class="menu-content">
              <span class="menu-title">ロック・ビート</span>
              <span class="menu-description">魂を揺さぶる、力強いリズムと歪んだギターのブレンド。</span>
            </div>
            <div v-if="selectedMenu === 'ロック・ビート' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
            </div>
          </button>
        </div>
        <div class="controls-container">
          <button @click="togglePlayback" class="control-button" :disabled="!selectedMenu" :class="{ 'is-disabled': !selectedMenu }">
            {{ isPlaying ? '■' : '▶' }}
          </button>
          <input type="range" min="0" max="1" step="0.01" :value="volume" @input="handleVolumeChange" class="volume-slider"/>
        </div>
        <div v-if="currentSeed" class="seed-container">
          <p>レコード番号 (シード値):</p>
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
      <a href="#" @click.prevent="openSoundCheckModal" class="footer-link">サウンドチェック</a>
    </div>
    <AboutModal :isVisible="isModalVisible" @close="closeModal" />
    <SoundCheckModal 
      :isVisible="isSoundCheckModalVisible" 
      :instruments="instrumentList"
      @close="closeSoundCheckModal"
      @playSound="handlePlaySoundCheck" 
    />
  </div>
</template>

<style>
body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif; }
.background-container { background-image: url('/bg-main.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
.content-panel { background-color: rgba(255, 255, 255, 0.88); padding: 20px 40px 30px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); backdrop-filter: blur(5px); width: 90%; max-width: 600px; text-align: center; }
.title { color: #363636; font-weight: bold; margin-bottom: 8px; }
.subtitle { color: #555; margin-top: 0; margin-bottom: 30px; }
.menu-container { display: flex; flex-direction: column; gap: 15px; }
.menu-button { background-color: #f5f5f5; border: 1px solid #dbdbdb; border-radius: 4px; padding: 15px 20px; cursor: pointer; transition: all 0.2s ease; text-align: left; display: flex; justify-content: space-between; align-items: center; font-family: inherit; width: 100%; }
.menu-button:hover { background-color: #e8e8e8; border-color: #b5b5b5; transform: translateY(-2px); box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.menu-button:disabled { cursor: not-allowed; opacity: 0.5; }
.menu-button.is-active { border-color: #485fc7; background-color: #eff2fb; }
.menu-content { display: flex; flex-direction: column; }
.menu-title { font-size: 1.1em; font-weight: bold; color: #363636; }
.menu-description { font-size: 0.9em; color: #7a7a7a; margin-top: 4px; }
.active-indicator svg { color: #485fc7; animation: fadeIn 0.5s ease; }
@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
.controls-container { margin-top: 30px; display: flex; justify-content: center; align-items: center; gap: 20px; }
.control-button { background-color: #363636; color: white; border: none; border-radius: 50%; width: 50px; height: 50px; font-size: 20px; cursor: pointer; display: flex; justify-content: center; align-items: center; transition: all 0.2s ease; }
.control-button:hover { background-color: #555; }
.control-button.is-disabled { background-color: #c5c5c5; cursor: not-allowed; }
.volume-slider { width: 150px; cursor: pointer; }
.footer-link-container { position: fixed; bottom: 15px; right: 20px; z-index: 10; display: flex; gap: 20px; }
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