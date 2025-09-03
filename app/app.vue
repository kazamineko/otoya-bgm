<script setup lang="ts">
import { ref, onUnmounted, watch } from 'vue';
import seedrandom from 'seedrandom';
import type * as ToneType from 'tone';
import AboutModal from '../components/AboutModal.vue';
import SoundCheckModal from '../components/SoundCheckModal.vue';

// --- UI状態変数 ---
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

// --- 音声関連 ---
let Tone: typeof ToneType | null = null;
let samplers: { [key: string]: { sampler: ToneType.Sampler, baseNote: string } } = {};
let rawSamplePlayers: ToneType.Players | null = null;
let reverb: ToneType.Reverb | null = null;
let chorus: ToneType.Chorus | null = null;
let delay: ToneType.PingPongDelay | null = null;
let masterComp: ToneType.Compressor | null = null;
let limiter: ToneType.Limiter | null = null;

// --- Virtual Amp Rig ---
let guitarPreComp: ToneType.Compressor | null = null;
let guitarDistortion: ToneType.Distortion | null = null;
let guitarPostEQ: ToneType.EQ3 | null = null;
let guitarCab: ToneType.Convolver | null = null;
let bassDistortion: ToneType.Distortion | null = null; // Corrected: Replaced Saturator with Distortion
let bassEQ: ToneType.EQ3 | null = null;
let bassCab: ToneType.Convolver | null = null;
let bassPostComp: ToneType.Compressor | null = null;

const scheduledEvents: (ToneType.Loop | ToneType.Part | ToneType.Sequence)[] = [];
let noise: ToneType.Noise | null = null;
let isAudioInitialized = ref(false);

type TuningParams = Record<string, { volume: number; attack: number; release: number; distortion?: number }>;

const tuningParams = ref<TuningParams>({});
const LOCAL_STORAGE_KEY = 'otoya-tuning-params-v6';

watch(tuningParams, (newParams) => {
  if (!isAudioInitialized.value) return;
  for (const instrumentName in newParams) {
    if (samplers[instrumentName]) {
      const sampler = samplers[instrumentName]!.sampler;
      const params = newParams[instrumentName]!;
      sampler.volume.value = params.volume;
      sampler.attack = params.attack;
      sampler.release = params.release;

      if (instrumentName === 'eguitar' && guitarDistortion && params.distortion !== undefined) {
        guitarDistortion.distortion = params.distortion;
      }
    }
  }
}, { deep: true });

const initializeAudio = async () => {
  isLoading.value = true;
  loadingMessage.value = '喫茶店の準備をしています...';

  const allSamplePaths: Record<string, string> = {
    piano: 'piano-c4.wav', bass: 'bass-c1.wav', ride: 'drum-ride.wav', brush: 'drum-brush.wav',
    epiano: 'epiano-c4.wav', kick: 'drum-kick.wav', snare: 'drum-snare.wav', pad: 'pad-cmaj7.wav',
    sax: 'sax-c4.wav', trombone: 'trombone-c3.wav',
    eguitar: 'eguitar-clean-c3.wav', ebass: 'ebass-di-e1.wav',
    rockKick: 'rock-kick.wav', rockSnare: 'rock-snare.wav', crash: 'drum-crash.wav',
    tomHigh: 'tom-high.wav', tomMid: 'tom-mid.wav', tomFloor: 'tom-floor.wav',
  };
  instrumentList.value = Object.keys(allSamplePaths);

  const masterTunedParams: TuningParams = {
    "piano": { "volume": 0, "attack": 0.01, "release": 1.0 }, "bass": { "volume": -3, "attack": 0.01, "release": 0.5 },
    "ride": { "volume": -9, "attack": 0.01, "release": 0.5 }, "brush": { "volume": -9, "attack": 0.01, "release": 0.2 },
    "epiano": { "volume": -3, "attack": 0.01, "release": 1 }, "kick": { "volume": 0, "attack": 0.01, "release": 0.2 },
    "snare": { "volume": -3, "attack": 0.01, "release": 0.2 }, "pad": { "volume": -6, "attack": 0.1, "release": 1 },
    "sax": { "volume": -3, "attack": 0.01, "release": 1 }, "trombone": { "volume": -3, "attack": 0.01, "release": 1 },
    "eguitar": { "volume": -6, "attack": 0.01, "release": 1.5, "distortion": 0.9 }, // Final tuning for IR
    "ebass": { "volume": 0, "attack": 0.01, "release": 1.5 }, // Final tuning for IR
    "rockKick": { "volume": 0, "attack": 0.01, "release": 0.2 }, "rockSnare": { "volume": -3, "attack": 0.01, "release": 0.2 },
    "crash": { "volume": -9, "attack": 0.01, "release": 0.5 },
    "tomHigh": { "volume": -6, "attack": 0.01, "release": 0.4 }, "tomMid": { "volume": -6, "attack": 0.01, "release": 0.4 },
    "tomFloor": { "volume": -6, "attack": 0.01, "release": 0.4 },
  };
  
  try {
    const savedParams = localStorage.getItem(LOCAL_STORAGE_KEY);
    if (savedParams) {
      const parsed = JSON.parse(savedParams);
      Object.keys(masterTunedParams).forEach(key => {
        if (parsed[key]) { masterTunedParams[key] = { ...masterTunedParams[key], ...parsed[key] }; }
      });
    }
  } catch (e) { console.error("Failed to load tuning params", e); }
  tuningParams.value = masterTunedParams;

  try {
    Tone = await import('tone');
    await Tone.start();
    loadingMessage.value = '店内の響きを調整しています...';
    
    // --- Master Output Chain ---
    limiter = new Tone.Limiter(-0.1).toDestination();
    masterComp = new Tone.Compressor({ threshold: -12, ratio: 3 }).connect(limiter);
    Tone.Destination.volume.value = Tone.gainToDb(volume.value);

    // --- Global Effects (Sends) ---
    reverb = new Tone.Reverb({ decay: 2.5, preDelay: 0.01, wet: 0.3 }).connect(masterComp);
    chorus = new Tone.Chorus(4, 2.5, 0.7).connect(masterComp);
    delay = new Tone.PingPongDelay("8n", 0.2).connect(masterComp);
    
    // --- Virtual Amp Rig Construction ---
    loadingMessage.value = '仮想アンプを構築しています...';
    // Guitar Rig
    guitarPreComp = new Tone.Compressor(-24, 8);
    guitarDistortion = new Tone.Distortion(tuningParams.value.eguitar?.distortion ?? 0.9);
    guitarPostEQ = new Tone.EQ3({ low: 2, mid: -8, high: 4 }); // Metal "scoop"
    guitarCab = new Tone.Convolver('/ir-guitar-cab.wav');

    // Bass Rig
    bassEQ = new Tone.EQ3({ low: 4, mid: 0, high: -2 });
    bassDistortion = new Tone.Distortion(0.2); // CORRECTED IMPLEMENTATION FOR SATURATION
    bassCab = new Tone.Convolver('/ir-bass-cab.wav');
    bassPostComp = new Tone.Compressor({threshold: -20, ratio: 4});

    await Promise.all([guitarCab.load, bassCab.load]);
    loadingMessage.value = '楽器を準備しています...';
    
    for (const name of instrumentList.value) {
      const params = tuningParams.value[name]!;
      const urls: Record<string, string> = {};
      const filePath = allSamplePaths[name]!;
      let baseNote = '';
      
      if (filePath.includes('-c3')) { baseNote = 'C3'; }
      else if (filePath.includes('-e1')) { baseNote = 'E1'; }
      else { baseNote = 'C4'; }
      urls[baseNote] = filePath;

      const sampler = new Tone.Sampler({
        urls, baseUrl: "/", volume: params.volume,
        attack: params.attack, release: params.release,
      });

      // --- Final Audio Routing ---
      switch(name) {
        case 'eguitar':
          sampler.chain(guitarPreComp, guitarDistortion, guitarPostEQ, guitarCab);
          guitarCab.fan(masterComp, reverb);
          break;
        case 'ebass':
          sampler.chain(bassEQ, bassDistortion, bassCab, bassPostComp, masterComp);
          break;
        case 'kick':
        case 'rockKick':
        case 'tomHigh':
        case 'tomMid':
        case 'tomFloor':
          sampler.connect(masterComp);
          break;
        case 'snare':
        case 'rockSnare':
        case 'crash':
        case 'ride':
        case 'brush':
           sampler.fan(masterComp, reverb);
          break;
        case 'pad':
          sampler.chain(reverb, masterComp);
          break;
        default:
          sampler.fan(masterComp, reverb, chorus, delay);
          break;
      }
      samplers[name] = { sampler, baseNote };
    }
    
    rawSamplePlayers = await new Promise<ToneType.Players>((resolve, reject) => {
        const p = new Tone!.Players({ urls: allSamplePaths, baseUrl: "/", onload: () => resolve(p) }).toDestination();
    });

    await Tone.loaded();
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
  if (samplers) {
    Object.values(samplers).forEach(s => s.sampler.releaseAll());
  }
  isPlaying.value = false;
};

// --- UI Event Handlers ---
const togglePlayback = async () => { if (isPlaying.value) { stopMusic(); } else { if (selectedMenu.value) { const [menuName, seed] = currentSeed.value.split(':'); if (menuName && seed) await playMusic(menuName, seed); } } };
const handleVolumeChange = (event: Event) => { const newVolume = parseFloat((event.target as HTMLInputElement).value); volume.value = newVolume; if (isAudioInitialized.value) { Tone!.Destination.volume.value = Tone!.gainToDb(newVolume); } };
const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const copySeed = () => { if(currentSeed.value) navigator.clipboard.writeText(currentSeed.value); };
const playFromSeed = async () => { if (seedInput.value) { const [menuName, seed] = seedInput.value.split(':'); const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター', 'ロック・ビート']; if (menuName && seed && validMenus.includes(menuName)) { await playMusic(menuName, seed); } else { alert('レコード番号の形式が正しくないか、存在しないジャンルです。'); } } };

// --- Sound Tuning Modal Handlers ---
const openSoundCheckModal = () => { isSoundCheckModalVisible.value = true; };
const closeSoundCheckModal = () => { isSoundCheckModalVisible.value = false; };

const handlePlaySound = async (instrumentName: string, type: 'sampler' | 'raw') => {
  if (!isAudioInitialized.value) {
    await initializeAudio();
    if (!isAudioInitialized.value) { alert('音源の初期化に失敗しました。'); return; }
  }

  const samplerData = samplers[instrumentName];
  if (type === 'sampler' && samplerData) {
    samplerData.sampler.triggerAttackRelease(samplerData.baseNote, '1n');
  } else if (type === 'raw' && rawSamplePlayers && rawSamplePlayers.has(instrumentName)) {
    const player = rawSamplePlayers.player(instrumentName);
    player.loop = false;
    player.start();
  }
};

const handleUpdateParam = (payload: { instrument: string, param: keyof TuningParams[string], value: number }) => {
  const { instrument, param, value } = payload;
  if (tuningParams.value[instrument]) {
    (tuningParams.value[instrument] as any)[param] = value;
  }
};

const handleSaveParams = () => {
  try {
    localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(tuningParams.value));
    alert('現在の設定をブラウザに保存しました。');
  } catch (e) {
    alert('設定の保存に失敗しました。');
    console.error(e);
  }
};

const handleExportParams = () => {
  console.clear();
  console.log("--- 喫茶「おとや」マスター認定サウンドパラメータ ---");
  console.log("以下のオブジェクトをコピーして、Geminiに提出してください。");
  console.table(tuningParams.value);
  console.log(JSON.stringify(tuningParams.value, null, 2));
  alert('現在の設定を開発者コンソールに出力しました。(F12で確認)');
};


// --- 音楽生成関数 ---
const createConcentrationSound = (rng: () => number) => {
    if (!Tone || !masterComp) return;
    if (!noise) noise = new Tone.Noise("pink").start();
    const filter = new Tone.Filter(800, "lowpass").connect(masterComp);
    noise.connect(filter);
    const loop = new Tone.Loop((time) => { filter.frequency.rampTo(600 + rng() * 400, 4, time); }, "4m").start(0);
    scheduledEvents.push(loop);
};

const createRelaxSound = (rng: () => number) => {
    if (!samplers.pad) return;
    const padSampler = samplers.pad.sampler;
    padSampler.triggerAttack('C4');
};

const createLoFiSound = (rng: () => number) => {
    if (!Tone || !samplers.epiano || !samplers.kick || !samplers.snare) return;
    const epiano = samplers.epiano.sampler;
    const kick = samplers.kick.sampler;
    const snare = samplers.snare.sampler;

    Tone.Transport.bpm.value = 80 + rng() * 15;
    const chords = [['C2', 'Eb2', 'G2'], ['A1', 'C2', 'E2']];
    let currentChord = chords[Math.floor(rng() * 2)]!;
    
    const kickSeq = new Tone.Sequence((time, note) => {
        if(note) kick.triggerAttackRelease('C4', '8n', time, 0.9 + rng() * 0.1);
    }, [1,0,1,0,1,0,1,0], "8n").start(0);

    const snareSeq = new Tone.Sequence((time, note) => {
        if(note) snare.triggerAttackRelease('C4', '8n', time, 0.8 + rng() * 0.2);
    }, [0,1,0,1], "4n").start(0);

    const chordLoop = new Tone.Loop((time) => {
        epiano.triggerAttackRelease(currentChord, '2n', time, 0.7 + rng() * 0.2);
    }, "1n").start(0);
    scheduledEvents.push(kickSeq, snareSeq, chordLoop);
};

const createRockSound = (rng: () => number) => {
    if (!Tone || !samplers.eguitar || !samplers.ebass || !samplers.rockKick || !samplers.rockSnare || !samplers.crash || !samplers.tomHigh || !samplers.tomMid || !samplers.tomFloor || !samplers.ride) return;
    const guitar = samplers.eguitar.sampler;
    const bass = samplers.ebass.sampler;
    const kick = samplers.rockKick.sampler;
    const snare = samplers.rockSnare.sampler;
    const crash = samplers.crash.sampler;
    const tomHigh = samplers.tomHigh.sampler;
    const tomMid = samplers.tomMid.sampler;
    const tomFloor = samplers.tomFloor.sampler;
    const ride = samplers.ride.sampler;

    Tone.Transport.bpm.value = 125 + rng() * 20;

    const drumLoop = new Tone.Loop(time => {
        const sixteenth = Tone!.Time('16n').toSeconds();
        const position = Tone!.Transport.position.toString();
        const measureStr = position.split(':')[0];
        const measure = measureStr ? parseInt(measureStr, 10) : 0;
        
        if (measure % 4 === 3 && rng() < 0.7) {
            const fillPatterns = [
                [{ offset: sixteenth * 12, note: tomHigh }, { offset: sixteenth * 13, note: tomMid }, { offset: sixteenth * 14, note: tomFloor }, { offset: sixteenth * 15, note: crash }],
                [{ offset: sixteenth * 12, note: snare }, { offset: sixteenth * 13, note: snare }, { offset: sixteenth * 14, note: tomHigh }, { offset: sixteenth * 15, note: tomMid }]
            ];
            const pattern = fillPatterns[Math.floor(rng() * fillPatterns.length)]!;
            pattern.forEach(fill => {
                fill.note.triggerAttack('C4', time + fill.offset, 0.8 + rng() * 0.2);
            });
        } else {
            kick.triggerAttack('C4', time, 1.0);
            snare.triggerAttack('C4', time + Tone!.Time('4n').toSeconds(), 0.9);
            kick.triggerAttack('C4', time + Tone!.Time('2n').toSeconds(), 1.0);
            snare.triggerAttack('C4', time + Tone!.Time('2n + 4n').toSeconds(), 0.9);
        }
        for (let i = 0; i < 8; i++) {
            ride.triggerAttack('C4', time + (sixteenth * 2 * i), 0.4 + (i % 2 === 0 ? 0.2 : 0) + rng() * 0.1);
        }
    }, '1m').start(0);

    const bassPart = new Tone.Sequence((time, note) => {
        if (note) bass.triggerAttackRelease(note, '8n', time, 0.9);
    }, ['E1', 'G1', 'A1', 'B1'], "4n").start(0);

    const guitarRiffPatterns = [
      [ { time: '0:0', note: 'E3', dur: '4n' }, { time: '0:2', note: 'G3', dur: '8n' }, { time: '0:3', note: 'A3', dur: '8n' } ],
      [ { time: '0:0', note: 'B3', dur: '2n' }, { time: '0:2', note: 'A3', dur: '4n' } ],
    ];
    const guitarRiff = guitarRiffPatterns[Math.floor(rng() * guitarRiffPatterns.length)]!;

    const guitarPart = new Tone.Part<{ time: string, note: string, dur: ToneType.Unit.Time }>((time, value) => {
        const vel = 0.9 + rng() * 0.1;
        guitar.triggerAttackRelease(value.note, value.dur, time, vel);
    }, guitarRiff).start(0);
    guitarPart.loop = true; guitarPart.loopEnd = '1m';
    
    scheduledEvents.push(bassPart, guitarPart, drumLoop);
};

const createJazzSound = (rng: () => number) => {
    if (!Tone || !samplers.piano || !samplers.bass || !samplers.ride || !samplers.sax) return;
    const piano = samplers.piano.sampler;
    const bass = samplers.bass.sampler;
    const ride = samplers.ride.sampler;
    const sax = samplers.sax.sampler;

    Tone.Transport.bpm.value = 100 + rng() * 20;
    Tone.Transport.swing = 0.05;
    let isSolo = false;

    const chords = [
        { time: '0:0', chord: ['D3', 'F3', 'A3', 'C4'] }, { time: '2:0', chord: ['G2', 'F3', 'B3', 'D4'] },
    ];
    const pianoPart = new Tone.Part((time, value) => {
        if(isSolo) return;
        piano.triggerAttackRelease(value.chord, '2n', time, 0.7);
    }, chords).start(0);
    pianoPart.loop = true; pianoPart.loopEnd = '4m';
    
    const bassPart = new Tone.Sequence((time, note) => {
        if (note) bass.triggerAttackRelease(note, '4n', time, 0.9);
    }, ['D1', 'G1', 'C2', 'F1'], '2n').start(0);

    const ridePart = new Tone.Loop(time => {
        ride.triggerAttack('C4', time, 0.7);
    }, '4n').start(0);

    const saxPart = new Tone.Sequence((time, note) => {
        if(isSolo && note) sax.triggerAttackRelease(note, '8n', time, 0.8);
    }, ['G3', 'A3', null, 'C4', 'A3', 'G3', null, 'E3'], '8n').start(0);
    
    const soloToggle = new Tone.Loop(() => { isSolo = !isSolo }, '8m').start('4m');
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
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('リラックス・デカフェ')" :class="{ 'is-active': selectedMenu === 'リラックス・デカフェ' }">
            <div class="menu-content">
              <span class="menu-title">リラックス・デカフェ</span>
              <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
            </div>
            <div v-if="selectedMenu === 'リラックス・デカフェ' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ジャズ・スペシャル')" :class="{ 'is-active': selectedMenu === 'ジャズ・スペシャル' }">
            <div class="menu-content">
              <span class="menu-title">ジャズ・スペシャル</span>
              <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
            </div>
            <div v-if="selectedMenu === 'ジャズ・スペシャル' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" :class="{ 'is-active': selectedMenu === 'Lo-Fi・ビター' }">
            <div class="menu-content">
              <span class="menu-title">Lo-Fi・ビター</span>
              <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
            </div>
            <div v-if="selectedMenu === 'Lo-Fi・ビター' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ロック・ビート')" :class="{ 'is-active': selectedMenu === 'ロック・ビート' }">
            <div class="menu-content">
              <span class="menu-title">ロック・ビート</span>
              <span class="menu-description">魂を揺さぶる、力強いリズムと歪んだギターのブレンド。</span>
            </div>
            <div v-if="selectedMenu === 'ロック・ビート' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
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
      <a href="#" @click.prevent="openSoundCheckModal" class="footer-link">サウンドチューニング</a>
    </div>
    <AboutModal :isVisible="isModalVisible" @close="closeModal" />
    <SoundCheckModal 
      :isVisible="isSoundCheckModalVisible" 
      :instruments="instrumentList"
      :tuningParams="tuningParams"
      @close="closeSoundCheckModal"
      @playSound="handlePlaySound"
      @update-param="handleUpdateParam"
      @save-params="handleSaveParams"
      @export-params="handleExportParams"
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