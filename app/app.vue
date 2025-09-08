<script setup lang="ts">
import { ref, onUnmounted, watch } from 'vue';
import seedrandom from 'seedrandom';
import type * as ToneType from 'tone';
// import { SampleLibrary } from 'tonejs-instruments'; // REMOVED
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
let Tone: typeof ToneType |
null = null;
let samplers: { [key: string]: { sampler: ToneType.Sampler, baseNote?: string } } = {};
// baseNote is now optional
let targetSamplers: { [key: string]: { sampler: ToneType.Sampler, baseNote: string } } = {};

// 新規: ヘビメタ音源用のSampler
let heavyMetalSampler: ToneType.Sampler | null = null;
let newRockBassSampler: { sampler: ToneType.Sampler, baseNote: string } | null = null;
let newSynthPianoSampler: { sampler: ToneType.Sampler, baseNote: string } | null = null;
let newElecOrganSampler: { sampler: ToneType.Sampler, baseNote: string } | null = null;

// DEBUG: Raw MP3 Player
let rawMp3Player: ToneType.Player | null = null;

// 旧ギターエフェクト群 (機能は温存)
let guitarPitchShift: ToneType.PitchShift | null = null;
let guitarVibrato: ToneType.Vibrato | null = null;
let guitarEQ: ToneType.EQ3 | null = null;
let guitarMidEQ: ToneType.EQ3 |
null = null;
let guitarDistortion: ToneType.Distortion | null = null;
let guitarCabinetFilter: ToneType.Filter | null = null;
let guitarComp: ToneType.Compressor | null = null;
let guitarChorus: ToneType.Chorus | null = null;
let bassEQNode: ToneType.EQ3 | null = null;

let rawSamplePlayers: ToneType.Players |
null = null;
let targetGuitarPlayer: ToneType.Player | null = null;
let targetBassPlayer: ToneType.Player | null = null;
let reverb: ToneType.Reverb | null = null;
let chorus: ToneType.Chorus | null = null;
let delay: ToneType.PingPongDelay |
null = null;
let masterComp: ToneType.Compressor | null = null;
let limiter: ToneType.Limiter | null = null;
let drumBusComp: ToneType.Compressor | null = null;
let drumBusVolume: ToneType.Volume | null = null;

let rideFilter: ToneType.Filter |
null = null;

const scheduledEvents: (ToneType.Loop | ToneType.Part | ToneType.Sequence)[] = [];
let noise: ToneType.Noise | null = null;
let isAudioInitialized = ref(false);

type TuningParams = Record<string, any>;

const tuningParams = ref<TuningParams>({});
const LOCAL_STORAGE_KEY = 'otoya-tuning-params-v12-pro';

const masterTunedParams: TuningParams = {
  "piano": { "volume": 0, "attack": 0.01, "release": 1 },
  "bass": { "volume": -3, "attack": 0.01, "release": 0.5 },
  "ride": { "volume": -9, "attack": 0.01, "release": 0.5 },
  "brush": { "volume": -9, "attack": 0.01, "release": 0.2 },
  "epiano": { "volume": -3, "attack": 0.01, "release": 1 },
  "kick": { "volume": 0, "attack": 0.01, "release": 0.2 },
  "snare": { "volume": -3, "attack": 0.01, "release": 0.2 },
  "pad": { "volume": -6, "attack": 0.1, "release": 1 },
  "sax": { "volume": -3, "attack": 0.01, "release": 1 },
  "trombone": { "volume": -3, "attack": 0.01, "release": 1 },
  "rockKick": { "volume": 0, "attack": 0.01, "release": 0.2 },
  "rockSnare": { "volume": -3, "attack": 0.01, "release": 0.2 },
  "crash": { "volume": -9, "attack": 0.01, "release": 0.5 },
  "tomHigh": { "volume": -6, "attack": 0.01, "release": 0.4 },
  "tomMid": { "volume": -6, "attack": 0.01, "release": 0.4 },
  "tomFloor": { "volume": -6, "attack": 0.01, "release": 0.4 },
  "target_eguitar": { "volume": 0, "attack": 0.005, "release": 0.6, "detune": 0, "eqLow": 0, "eqMid": 0, "eqHigh": 0, "distortion": 0.00 },
  "target_ebass": { "volume": 0, "attack": 0.018, "release": 1.3, "eqLow": 0, "eqMid": 0, "eqHigh": 0 },
  "spiano": { "volume": -15, "attack": 0.01, "release": 1.5 },
  "eorgan": { "volume": -12, "attack": 0.05, "release": 1 }
};

const heavyMetalUrls = {
  'A3': 'HMRhyA_Powerchord-A.mp3',
  'B3': 'HMRhyA_Powerchord-B.mp3',
  'C4': 'HMRhyA_Powerchord-C.mp3',
  'D3': 'HMRhyA_Powerchord-D_Lo.mp3',
  'D4': 'HMRhyA_Powerchord-D_Hi.mp3',
  'E3': 'HMRhyA_Powerchord-E.mp3',
  'F3': 'HMRhyA_Powerchord-F.mp3',
  'G3': 'HMRhyA_Powerchord-G.mp3'
};

watch(tuningParams, (newParams) => {
  if (!isAudioInitialized.value || !Tone) return;

  if (newRockBassSampler && newParams.target_ebass) {
      const p = newParams.target_ebass;
      if(p.volume !== undefined) newRockBassSampler.sampler.volume.value = p.volume;
      if(p.attack !== undefined) newRockBassSampler.sampler.attack = p.attack;
      if(p.release !== undefined) newRockBassSampler.sampler.release = p.release;
      if(bassEQNode) {
        if(p.eqLow !== undefined) bassEQNode.low.value = p.eqLow;
        if(p.eqMid !== undefined) bassEQNode.mid.value = p.eqMid;
        if(p.eqHigh !== undefined) bassEQNode.high.value = p.eqHigh;
      }
  }
  if (newSynthPianoSampler && newParams.spiano) {
      const p = newParams.spiano;
      if(p.volume !== undefined) newSynthPianoSampler.sampler.volume.value = p.volume;
      if(p.attack !== undefined) newSynthPianoSampler.sampler.attack = p.attack;
      if(p.release !== undefined) newSynthPianoSampler.sampler.release = p.release;
  }
  if (newElecOrganSampler && newParams.eorgan) {
      const p = newParams.eorgan;
      if(p.volume !== undefined) newElecOrganSampler.sampler.volume.value = p.volume;
      if(p.attack !== undefined) newElecOrganSampler.sampler.attack = p.attack;
      if(p.release !== undefined) newElecOrganSampler.sampler.release = p.release;
  }

  const eguitarTargetParams = newParams.target_eguitar;
  if (heavyMetalSampler && eguitarTargetParams) {
      if(eguitarTargetParams.volume !== undefined) heavyMetalSampler.volume.value = eguitarTargetParams.volume;
      if(eguitarTargetParams.attack !== undefined) heavyMetalSampler.attack = eguitarTargetParams.attack;
      if(eguitarTargetParams.release !== undefined) heavyMetalSampler.release = eguitarTargetParams.release;
  }

  for (const instrumentName in newParams) {
    const exclusionList = ['target_ebass', 'spiano', 'eorgan', 'target_eguitar'];
    if (exclusionList.includes(instrumentName)) continue;

    const activeSampler = samplers[instrumentName] || targetSamplers[instrumentName];
    if (activeSampler) {
      const sampler = activeSampler.sampler;
      const params = newParams[instrumentName]!;
      if(params.volume !== undefined) sampler.volume.value = params.volume;
      if(params.attack !== undefined) sampler.attack = params.attack;
      if(params.release !== undefined) sampler.release = params.release;
    }
  }
}, { deep: true });

const initializeAudio = async () => {
  console.log('[GEMINI_DEBUG_LOG] initializeAudio: 開始');
  isLoading.value = true;
  loadingMessage.value = '喫茶店の準備をしています...';
  const allSamplePaths: Record<string, string> = {
    piano: 'keys/piano-c4.mp3', bass: 'ebass/bass-c1.mp3', ride: 'drums/drum-ride.mp3', brush: 'drums/drum-brush.mp3',
    epiano: 'keys/epiano-c4.mp3', kick: 'drums/drum-kick.mp3', snare: 'drums/drum-snare.mp3', pad: 'synth_misc/pad-cmaj7.mp3',
    sax: 'winds/sax-c4.mp3', trombone: 'winds/trombone-c3.mp3',
    rockKick: 'drums/rock-kick.mp3', rockSnare: 'drums/rock-snare.mp3', crash: 'drums/drum-crash.mp3',
    tomHigh: 'drums/tom-high.mp3', tomMid: 'drums/tom-mid.mp3', tomFloor: 'drums/tom-floor.mp3',
    target_ebass: 'ebass/ebass-e1.mp3',
  };
  instrumentList.value = Object.keys(masterTunedParams).filter(k => !k.startsWith('target_'));
  
  if (!instrumentList.value.includes('spiano')) {
    instrumentList.value.push('spiano');
  }
  if (!instrumentList.value.includes('eorgan')) {
    instrumentList.value.push('eorgan');
  }
  if (!instrumentList.value.includes('eguitar')) {
    instrumentList.value.push('eguitar');
  }
  if (!instrumentList.value.includes('ebass')) {
    instrumentList.value.push('ebass');
  }

  const initialParams = JSON.parse(JSON.stringify(masterTunedParams));
  try {
    const savedParams = localStorage.getItem(LOCAL_STORAGE_KEY);
    if (savedParams) {
      const parsed = JSON.parse(savedParams);
      Object.keys(initialParams).forEach(key => {
        if (parsed[key]) { initialParams[key] = { ...initialParams[key], ...parsed[key] }; }
      });
    }
  } catch (e) { console.error("Failed to load tuning params", e); }
  tuningParams.value = initialParams;

  try {
    console.log('[GEMINI_DEBUG_LOG] Tone.jsのインポートと開始...');
    Tone = await import('tone');
    await Tone.start();
    if (!Tone) { throw new Error('Tone.js failed to load'); }
    console.log('[GEMINI_DEBUG_LOG] Tone.jsの準備完了');
    loadingMessage.value = '店内の響きを調整しています...';
    
    limiter = new Tone.Limiter(-0.1).toDestination();
    masterComp = new Tone.Compressor({ threshold: -12, ratio: 3 }).connect(limiter);
    
    drumBusVolume = new Tone.Volume(-3).connect(masterComp);
    drumBusComp = new Tone.Compressor({ threshold: -25, ratio: 5, attack: 0.01, release: 0.1 }).connect(drumBusVolume);

    Tone.Destination.volume.value = Tone.gainToDb(volume.value);

    reverb = new Tone.Reverb({ decay: 2.5, preDelay: 0.01, wet: 0.3 }).connect(masterComp);
    chorus = new Tone.Chorus(4, 2.5, 0.7).connect(masterComp);
    delay = new Tone.PingPongDelay("8n", 0.2).connect(masterComp);
    
    rawMp3Player = new Tone.Player().toDestination();
    
    const ebassTargetP = tuningParams.value.target_ebass;
    bassEQNode = new Tone.EQ3({
        low: ebassTargetP.eqLow,
        mid: ebassTargetP.eqMid,
        high: ebassTargetP.eqHigh
    });
    console.log('[GEMINI_DEBUG_LOG] 全てのエフェクトノードをインスタンス化完了');
    
    loadingMessage.value = 'AI奏者を準備しています...';
    
    rideFilter = new Tone.Filter(10000, 'lowpass');
    
    targetGuitarPlayer = new Tone.Player('/eguitar/C5_s6_01.mp3').toDestination();
    targetBassPlayer = new Tone.Player('/ebass/ebass-e1.mp3').toDestination();

    console.log('[GEMINI_DEBUG_LOG] 音声ファイルの読み込み開始 (Players & Samplers)...');
    
    const guitarParams = tuningParams.value['target_eguitar'];
    const heavyMetalSamplerPromise = new Promise<void>(resolve => {
        heavyMetalSampler = new Tone!.Sampler({
            urls: heavyMetalUrls,
            baseUrl: "/HMRhyA/",
            volume: guitarParams.volume,
            attack: guitarParams.attack,
            release: guitarParams.release,
            onload: () => resolve()
        });
    });
    
    await Promise.all([targetGuitarPlayer.load, targetBassPlayer.load, heavyMetalSamplerPromise]);
    console.log('[GEMINI_DEBUG_LOG] 音声ファイルの読み込み完了');
    
    if (heavyMetalSampler && masterComp) {
        heavyMetalSampler.connect(masterComp);
    }

    loadingMessage.value = '楽器を最終調整しています...';

    const newBassUrls = {
        'E1': 'ebass-new_E.mp3', 'F1': 'ebass-new_F.mp3', 'F#1': 'ebass-new_Fs.mp3', 'G1': 'ebass-new_G.mp3',
        'G#1': 'ebass-new_Gs.mp3', 'A1': 'ebass-new_A.mp3', 'A#1': 'ebass-new_As.mp3', 'B1': 'ebass-new_B.mp3',
        'C2': 'ebass-new_C.mp3', 'C#2': 'ebass-new_Cs.mp3', 'D2': 'ebass-new_D.mp3', 'D#2': 'ebass-new_Ds.mp3',
        'E2': 'ebass-new_E2.mp3',
    };
    const newRockBassParams = tuningParams.value['target_ebass'];
    const loadedNewRockBassSampler = new Tone.Sampler({
        urls: newBassUrls, baseUrl: "/ebass/",
        volume: newRockBassParams.volume, attack: newRockBassParams.attack, release: newRockBassParams.release
    });
    newRockBassSampler = { sampler: loadedNewRockBassSampler, baseNote: 'A1' };

    const newSynthPianoUrls = {
        'C2': 'spiano-C2v100.mp3', 'F#1': 'spiano-Fs1v100.mp3',
        'C3': 'spiano-C3v100.mp3', 'F#2': 'spiano-Fs2v100.mp3',
        'C4': 'spiano-C4v100.mp3', 'F#3': 'spiano-Fs3v100.mp3',
        'C5': 'spiano-C5v100.mp3', 'F#4': 'spiano-Fs4v100.mp3',
        'C6': 'spiano-C6v100.mp3', 'F#5': 'spiano-Fs5v100.mp3',
        'C7': 'spiano-C7v100.mp3', 'F#6': 'spiano-Fs6v100.mp3',
    };
    const newSynthPianoParams = tuningParams.value['spiano'];
    const loadedNewSynthPianoSampler = new Tone.Sampler({
        urls: newSynthPianoUrls, baseUrl: "/keys/",
        volume: newSynthPianoParams.volume, attack: newSynthPianoParams.attack, release: newSynthPianoParams.release
    });
    newSynthPianoSampler = { sampler: loadedNewSynthPianoSampler, baseNote: 'C4' };
    samplers['spiano'] = newSynthPianoSampler;

    const newElecOrganUrls = {
        'C2': 'eorgan-C2.mp3', 'E2': 'eorgan-E2.mp3', 'G#2': 'eorgan-Gs2.mp3',
        'C3': 'eorgan-C3.mp3', 'E3': 'eorgan-E3.mp3', 'G#3': 'eorgan-Gs3.mp3',
        'C4': 'eorgan-C4.mp3', 'E4': 'eorgan-E4.mp3', 'G#4': 'eorgan-Gs4.mp3',
        'C5': 'eorgan-C5.mp3', 'E5': 'eorgan-E5.mp3', 'G#5': 'eorgan-Gs5.mp3',
        'C6': 'eorgan-C6.mp3', 'E6': 'eorgan-E6.mp3', 'G#6': 'eorgan-Gs6.mp3',
        'C7': 'eorgan-C7.mp3'
    };
    const newElecOrganParams = tuningParams.value['eorgan'];
    const loadedNewElecOrganSampler = new Tone.Sampler({
        urls: newElecOrganUrls, baseUrl: "/keys/",
        volume: newElecOrganParams.volume, attack: newElecOrganParams.attack, release: newElecOrganParams.release
    });
    newElecOrganSampler = { sampler: loadedNewElecOrganSampler, baseNote: 'C4' };
    samplers['eorgan'] = newElecOrganSampler;

    for (const name of Object.keys(allSamplePaths)) {
      const params = tuningParams.value[name];
      if (!params) continue;
      const urls: Record<string, string> = { 'C4': allSamplePaths[name]! };
      const sampler = new Tone.Sampler({ urls, baseUrl: "/", volume: params.volume, attack: params.attack, release: params.release });
      if (name.startsWith('target_')) {
        targetSamplers[name] = { sampler, baseNote: 'C4' };
      } else {
        samplers[name] = { sampler, baseNote: 'C4' };
      }
    }
    
    console.log('[GEMINI_DEBUG_LOG] シグナルチェーンの接続開始...');
    if (masterComp && reverb && chorus && delay && rideFilter && drumBusComp && bassEQNode && newRockBassSampler) {
      newRockBassSampler.sampler.chain(bassEQNode, drumBusComp);
      newSynthPianoSampler.sampler.fan(masterComp, reverb, delay);
      newElecOrganSampler.sampler.fan(masterComp, reverb, delay);
      
      const rockDrumKit = ['rockKick', 'rockSnare', 'crash', 'tomHigh', 'tomMid', 'tomFloor', 'ride'];

      for (const [name, data] of Object.entries(samplers)) {
        if (rockDrumKit.includes(name)) {
          if (name === 'ride') {
            data.sampler.connect(rideFilter);
            rideFilter.connect(drumBusComp);
          } else {
            data.sampler.connect(drumBusComp);
          }
        } else {
           data.sampler.fan(masterComp, reverb, chorus, delay);
        }
      }
      
      for (const [name, { sampler }] of Object.entries(targetSamplers)) {
        sampler.fan(masterComp, reverb);
      }
      console.log('[GEMINI_DEBUG_LOG] 全てのシグナルチェーン接続完了');
    } else {
        console.error('[GEMINI_DEBUG_LOG] シグナルチェーン接続エラー: 必須となるオーディオノードがnullです。');
    }

    rawSamplePlayers = await new Promise<ToneType.Players>((resolve) => {
      const p = new Tone!.Players({ urls: allSamplePaths, baseUrl: "/", onload: () => resolve(p) }).toDestination();
    });
    
    await Tone.loaded();
    isAudioInitialized.value = true;
    loadingMessage.value = '準備ができました';
    console.log('[GEMINI_DEBUG_LOG] initializeAudio: 正常終了');
  } catch (error: any) {
    loadingMessage.value = `エラーが発生しました: ${error.message}`;
    console.error("[GEMINI_DEBUG_LOG] initializeAudioで致命的なエラーが発生:", error);
  } finally {
    isLoading.value = false;
  }
};

onUnmounted(() => { stopMusic(); });

const playMusic = async (menuName: string, seed?: string) => {
  if (!isAudioInitialized.value) { await initializeAudio(); if (!isAudioInitialized.value) return; }
  console.log(`[GEMINI_DIAG_LOG] playMusic: '${menuName}' がリクエストされました。既存の音楽を停止します...`);
  if (isPlaying.value) stopMusic();
  const randomPart = seed || Date.now().toString(36) + Math.random().toString(36).substring(2);
  const newSeed = `${menuName}:${randomPart}`;
  const rng = seedrandom(randomPart);
  let musicGenerated = false;
  switch (menuName) {
    case '集中ブレンド': musicGenerated = createConcentrationSound(rng); break;
    case 'リラックス・デカフェ': musicGenerated = createRelaxSound(rng); break;
    case 'ジャズ・スペシャル': musicGenerated = createJazzSound(rng); break;
    case 'Lo-Fi・ビター': musicGenerated = createLoFiSound(rng); break;
    case 'ロック・ビート': musicGenerated = createRockSound(rng); break;
    case 'LITE-Style・Post Rock': musicGenerated = createLiteStyleRock(rng); break;
  }
  if (musicGenerated && Tone) {
    currentSeed.value = newSeed; 
    Tone.Transport.start(); 
    isPlaying.value = true; 
    selectedMenu.value = menuName;
  } else if (!seed) { 
    alert('このメニューは現在準備中です。'); 
  }
};

const stopMusic = () => {
  if (!isPlaying.value || !Tone) return;
  console.log(`[GEMINI_DIAG_LOG] stopMusic: 停止処理開始。現在 ${scheduledEvents.length} 個のイベントを処理します。`);
  Tone.Transport.stop(); 
  Tone.Transport.cancel(0);
  
  // First, stop all running events.
  scheduledEvents.forEach((event, index) => {
    // @ts-ignore
    if (event.state === "started") {
      console.log(`[GEMINI_DIAG_LOG] -> 停止中 イベント ${index + 1}/${scheduledEvents.length}: ${event.constructor.name}`);
      event.stop(0);
    }
  });

  // Then, dispose of all events.
  scheduledEvents.forEach((event, index) => {
    // @ts-ignore
    console.log(`[GEMINI_DIAG_LOG] -> 破棄中 イベント ${index + 1}/${scheduledEvents.length}: ${event.constructor.name}`);
    event.dispose();
  });

  scheduledEvents.length = 0;
  console.log(`[GEMINI_DIAG_LOG] stopMusic: イベント配列をクリアしました。残り: ${scheduledEvents.length}個`);
  if (noise) { noise.stop(0); noise.dispose(); noise = null; }
  
  heavyMetalSampler?.releaseAll();
  Object.values(samplers).forEach(s => s.sampler.releaseAll());
  Object.values(targetSamplers).forEach(s => s.sampler.releaseAll());
  
  isPlaying.value = false;
  console.log('[GEMINI_DIAG_LOG] stopMusic: 完了。');
};

const triggerGuitarSound = (note: string | string[], duration: ToneType.Unit.Time, time: ToneType.Unit.Time, velocity: number) => {
    heavyMetalSampler?.triggerAttackRelease(note, duration, time, velocity);
}

const togglePlayback = async () => { if (isPlaying.value) { stopMusic(); } else if (selectedMenu.value) { const [menuName, seed] = currentSeed.value.split(':'); if (menuName && seed) await playMusic(menuName, seed); } };
const handleVolumeChange = (event: Event) => { const newVolume = parseFloat((event.target as HTMLInputElement).value); volume.value = newVolume; if (isAudioInitialized.value && Tone) { Tone.Destination.volume.value = Tone.gainToDb(newVolume); } };
const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const copySeed = () => { if(currentSeed.value) navigator.clipboard.writeText(currentSeed.value); };
const playFromSeed = async () => { 
  const [menuName, seed] = seedInput.value.split(':');
  const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター', 'ロック・ビート', 'LITE-Style・Post Rock']; 
  if (menuName && seed && validMenus.includes(menuName)) { 
    await playMusic(menuName, seed); 
  } else { 
    alert('レコード番号の形式が正しくないか、存在しないジャンルです。'); 
  } 
};

const openSoundCheckModal = () => { isSoundCheckModalVisible.value = true; };
const closeSoundCheckModal = () => { isSoundCheckModalVisible.value = false; };

const handlePlaySound = async (instrumentName: string, type: 'sampler' | 'raw' | 'target' | 'target_sampler') => {
  if (!isAudioInitialized.value) { await initializeAudio(); if (!isAudioInitialized.value) { alert('音源の初期化に失敗しました。'); return; } }
  
  const duration = '2n';
  const now = Tone?.now();
  if (type === 'target_sampler') {
      if (instrumentName === 'target_ebass' && newRockBassSampler) {
        newRockBassSampler.sampler.triggerAttackRelease('E1', duration);
        return; 
      }

      if (instrumentName === 'target_eguitar' && now) {
        triggerGuitarSound('D4', '1n', now, 0.9);
      }
  } else {
      const samplerData = samplers[instrumentName];
      if (type === 'sampler' && samplerData && samplerData.baseNote) { samplerData.sampler.triggerAttackRelease(samplerData.baseNote, duration); }
      else if (type === 'raw' && rawSamplePlayers && rawSamplePlayers.has(instrumentName)) { rawSamplePlayers.player(instrumentName).start(); }
      else if (type === 'target' && instrumentName === 'eguitar' && targetGuitarPlayer) { targetGuitarPlayer.start(); }
      else if (type === 'target' && instrumentName === 'ebass' && targetBassPlayer) { targetBassPlayer.start(); }
  }
};

const handlePlayRawSample = async (payload: { url: string, folder: string }) => {
  if (!rawMp3Player) return;
  const fullUrl = `/${payload.folder}/${payload.url}`;
  console.log('[GEMINI_DEBUG_LOG] Raw MP3 Playback Requested:', fullUrl);
  try {
    if (rawMp3Player.state === 'started') {
      rawMp3Player.stop();
    }
    await rawMp3Player.load(fullUrl);
    rawMp3Player.start();
  } catch (e) {
    console.error(`[GEMINI_DEBUG_LOG] Failed to load or play raw sample ${fullUrl}`, e);
  }
};


const handleUpdateParam = (payload: { instrument: string, param: string, value: any }) => {
  if (tuningParams.value[payload.instrument]) {
    const updatedInstrumentParams = { ...tuningParams.value[payload.instrument] };
    updatedInstrumentParams[payload.param] = payload.value;
    tuningParams.value[payload.instrument] = updatedInstrumentParams;
  }
};

const handleSaveParams = () => { try { localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(tuningParams.value)); alert('現在の設定をブラウザに保存しました。'); } catch (e) { alert('設定の保存に失敗しました。'); } };
const handleExportParams = () => { console.clear(); console.log(JSON.stringify(tuningParams.value, null, 2)); alert('現在の設定を開発者コンソールに出力しました。'); };
const handleResetParams = () => { 
  if (confirm('現在の調整を破棄し、全ての設定を初期値に戻します。よろしいですか？')) {
    tuningParams.value = JSON.parse(JSON.stringify(masterTunedParams)); 
  }
};
const handleSetExtremeEq = (type: 'cut' | 'boost') => {
  const value = type === 'cut' ? -12 : 12;
  console.log(`[GEMINI_LOG_BUTTON] Diagnostic button clicked: ${type}. Setting all EQ bands to ${value}.`);
  const currentParams = tuningParams.value.target_eguitar;
  tuningParams.value.target_eguitar = {
    ...currentParams,
    eqLow: value,
    eqMid: value,
    eqHigh: value,
  };
};

// ---
// SECTION: Music Generation Logic
// ---

const ROLES = { VERSE: 'verse', CHORUS: 'chorus' } as const;
type Role = typeof ROLES[keyof typeof ROLES];

const createConcentrationSound = (rng: () => number): boolean => { 
    if (!Tone || !masterComp) return false;
    scheduledEvents.forEach(e => e.dispose()); scheduledEvents.length = 0;
    if (noise) { noise.stop(0); noise.dispose(); noise = null; }
    
    noise = new Tone.Noise("pink").start(0);
    const filter = new Tone.Filter(800, "lowpass").connect(masterComp);
    noise.connect(filter);
    
    const loop = new Tone.Loop((time) => { filter.frequency.rampTo(600 + rng() * 400, 4, time); }, "4m").start(0);
    scheduledEvents.push(loop);
    return true;
};
const createRelaxSound = (rng: () => number): boolean => {
    if (!samplers.pad) return false;
    scheduledEvents.forEach(e => e.dispose());
    scheduledEvents.length = 0;
    const { sampler: padSampler, baseNote } = samplers.pad;
    if (!baseNote) return false;
    Tone?.Transport.scheduleOnce(time => {
        padSampler.triggerAttack(baseNote, time);
    }, 0);
    return true; 
};
const createLoFiSound = (rng: () => number): boolean => {
    if (!Tone || !samplers.epiano || !samplers.kick || !samplers.snare) return false;
    scheduledEvents.forEach(e => e.dispose()); scheduledEvents.length = 0;
    const kickSeq = new Tone.Sequence((time, note) => { if(note && samplers.kick?.baseNote) samplers.kick.sampler.triggerAttackRelease(samplers.kick.baseNote, '8n', time, 0.9 + rng() * 0.1); }, [1,0,1,0,1,0,1,0], "8n").start(0);
    const snareSeq = new Tone.Sequence((time, note) => { if(note && samplers.snare?.baseNote) samplers.snare.sampler.triggerAttackRelease(samplers.snare.baseNote, '8n', time, 0.8 + rng() * 0.2); }, [0,1,0,1], "4n").start(0);
    const chordLoop = new Tone.Loop((time) => {
      const chords = [['C2', 'Eb2', 'G2'], ['A1', 'C2', 'E2']];
      let currentChord = chords[Math.floor(rng() * 2)]!;
      if(samplers.epiano) samplers.epiano.sampler.triggerAttackRelease(currentChord, '2n', time, 0.7 + rng() * 0.2); 
    }, "1n").start(0);
    scheduledEvents.push(kickSeq, snareSeq, chordLoop);
    return true; 
};
const createJazzSound = (rng: () => number): boolean => {
    if (!Tone || !samplers.piano || !samplers.bass || !samplers.ride || !samplers.sax) return false;
    scheduledEvents.forEach(e => e.dispose()); scheduledEvents.length = 0;
    const { sampler: piano } = samplers.piano;
    const { sampler: bass } = samplers.bass;
    const { sampler: ride, baseNote: rideNote } = samplers.ride;
    const { sampler: sax } = samplers.sax;
    if(!rideNote) return false;
    
    Tone.Transport.bpm.value = 100 + rng() * 20;
    Tone.Transport.swing = 0.05;
    let isSolo = false;

    const chords = [ { time: '0:0', chord: ['D3', 'F3', 'A3', 'C4'] }, { time: '2:0', chord: ['G2', 'F3', 'B3', 'D4'] }, ];
    const pianoPart = new Tone.Part<({ time: string, chord: string[] })>((time, value) => { if(!isSolo) piano.triggerAttackRelease(value.chord, '2n', time, 0.7); }, chords).start(0);
    pianoPart.loop = true; pianoPart.loopEnd = '4m';

    const bassPart = new Tone.Sequence((time, note) => { bass.triggerAttackRelease(note, '4n', time, 0.9); }, ['D1', 'G1', 'C2', 'F1'], '2n').start(0);
    const ridePart = new Tone.Loop(time => { ride.triggerAttack(rideNote, time, 0.7); }, '4n').start(0);
    const saxPart = new Tone.Sequence((time, note) => { if(isSolo && note) sax.triggerAttackRelease(note, '8n', time, 0.8); }, ['G3', 'A3', null, 'C4', 'A3', 'G3', null, 'E3'], '8n').start(0);
    const soloToggle = new Tone.Loop(() => { isSolo = !isSolo }, '8m').start('4m');
    
    scheduledEvents.push(pianoPart, bassPart, ridePart, saxPart, soloToggle);
    return true; 
};

const createRockSound = (rng: () => number): boolean => {
    console.log('[GEMINI_DEBUG_LOG] createRockSound: Checking samplers...');
    if (!Tone || !heavyMetalSampler || !samplers.rockKick || !samplers.rockSnare || !newRockBassSampler || !newSynthPianoSampler || !newElecOrganSampler) {
        return false;
    }
    scheduledEvents.forEach(e => e.dispose()); scheduledEvents.length = 0;

    const bpm = 130 + rng() * 20;
    Tone.Transport.bpm.value = bpm;
    Tone.Transport.swing = 0.05;
    const progression = ['E', 'G', 'A', 'A'];
    let leadInstrument: 'guitar' | 'synth' | 'organ' = 'guitar';

    const kickSeq = new Tone.Sequence((time, note) => {
        if(note && samplers.rockKick?.baseNote) samplers.rockKick?.sampler.triggerAttack(samplers.rockKick.baseNote, time);
    }, [1, 0, 0, 0, 1, 0, 0, 0], "8n").start(0);
    const snareSeq = new Tone.Sequence((time, note) => {
        if(note && samplers.rockSnare?.baseNote) samplers.rockSnare?.sampler.triggerAttack(samplers.rockSnare.baseNote, time);
    }, [0, 1, 0, 1], "4n").start(0);
    const bassSeq = new Tone.Sequence((time, noteOn) => {
        if(noteOn && Tone){
            const measure = Math.floor(Tone.Transport.getTicksAtTime(time) / (Tone.Transport.PPQ * 4));
            const rootNote = progression[measure % 4]!;
            const note = rng() < 0.1 ? `${rootNote}2` : `${rootNote}1`;
            newRockBassSampler?.sampler.triggerAttackRelease(note, '8n', time);
        }
    }, [1, 1, 1, 1, 1, 1, 1, 1], "8n").start(0);

    const leadSwitchLoop = new Tone.Loop(time => {
        const choice = rng();
        if (choice < 0.5) {
            leadInstrument = 'guitar';
        } else if (choice < 0.75) {
            leadInstrument = 'synth';
        } else {
            leadInstrument = 'organ';
        }
    }, '4m').start(0);

    const mainLoop = new Tone.Loop((time) => {
      if (!Tone) return;
      const totalBeats = Math.floor(Tone.Transport.getTicksAtTime(time) / Tone.Transport.PPQ);
      const measure = Math.floor(totalBeats / 4);
      const rootNote = progression[measure % 4]!;
      const vel = 0.9 + rng() * 0.1;
      
      const isGhostNote = rng() < 0.15;
      const leadVelocity = isGhostNote ? vel * 0.3 : vel;

      const leadAction = {
        'guitar': () => triggerGuitarSound(`${rootNote}3`, '4n', time, leadVelocity),
        'synth': () => newSynthPianoSampler?.sampler.triggerAttackRelease([`${rootNote}4`, `${rootNote}5`], '8n', time, leadVelocity),
        'organ': () => newElecOrganSampler?.sampler.triggerAttackRelease([`${rootNote}3`, `${rootNote}4`], '4n', time, leadVelocity * 0.8),
      };
      
      const padAction = {
          'guitar': () => newSynthPianoSampler?.sampler.triggerAttackRelease([`${rootNote}4`], '2n', time, vel * 0.6),
          'synth': () => {
              triggerGuitarSound(`${rootNote}3`, '8n', time, vel * 0.8);
          },
          'organ': () => newSynthPianoSampler?.sampler.triggerAttackRelease([`${rootNote}5`], '2n', time, vel * 0.6)
      };

      if (totalBeats % 4 === 0) {
        leadAction[leadInstrument]();
      } else {
        padAction[leadInstrument]();
      }

    }, "4n").start(0);

    scheduledEvents.push(kickSeq, snareSeq, bassSeq, mainLoop, leadSwitchLoop);
    return true; 
};

const createLiteStyleRock = (rng: () => number): boolean => {
    console.log('[GEMINI_DIAG_LOG] createLiteStyleRock (Multi-Event Ver.): 初期化中...');
    if (!Tone || !heavyMetalSampler || !samplers.rockKick || !samplers.rockSnare || !samplers.ride || !samplers.crash) {
        console.error('[GEMINI_DIAG_LOG] createLiteStyleRock: 必須サンプラーがありません。');
        return false;
    }
    scheduledEvents.forEach(e => e.dispose());
    scheduledEvents.length = 0;

    Tone.Transport.bpm.value = 135;
    Tone.Transport.swing = 0;

    const { sampler: kick, baseNote: kickNote } = samplers.rockKick;
    const { sampler: snare, baseNote: snareNote } = samplers.rockSnare;
    const { sampler: ride, baseNote: rideNote } = samplers.ride;
    const { sampler: crash, baseNote: crashNote } = samplers.crash;
    if (!kickNote || !snareNote || !rideNote || !crashNote) return false;

    // --- Verse Patterns ---
    const verseKickSeq = new Tone.Sequence((time, note) => { if (note) kick.triggerAttack(kickNote, time); }, [1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0], "16n");
    const verseSnareSeq = new Tone.Sequence((time, note) => { if (note) snare.triggerAttack(snareNote, time); }, [0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0], "16n");
    const verseRideSeq = new Tone.Sequence((time, note) => { if (note) ride.triggerAttack(rideNote, time, 0.7); }, [1, 1, 1, 1, 1, 1, 1, 1], "8n");
    const verseGuitarRiff = new Tone.Sequence((time, note) => { if (note) triggerGuitarSound(note, "8n", time, 1.0); }, ['E3', null, 'G3', 'A3', null, 'G3', null, 'D4'], "8n");

    // --- Chorus Patterns ---
    const chorusKickSeq = new Tone.Sequence((time, note) => { if (note) kick.triggerAttack(kickNote, time, 1.0); }, [1, 1, 0, 1, 1, 0, 1, 0, 1, 1, 0, 1, 1, 1, 1, 0], "16n");
    const chorusSnareSeq = new Tone.Sequence((time, note) => { if (note) snare.triggerAttack(snareNote, time, 1.0); }, [0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 1, 0, 0, 1], "16n");
    const chorusCrashSeq = new Tone.Sequence((time, note) => { if (note) crash.triggerAttack(crashNote, time, 0.9); }, [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0], "2m");
    
    const chorusGuitarPart = new Tone.Part<{ time: string, note: string, duration: string }>((time, value) => {
        triggerGuitarSound(value.note, value.duration, time, 1.0);
    }, [
        { time: '0:0:0', note: 'E3', duration: '8n' }, { time: '0:0:2', note: 'G3', duration: '8n' }, { time: '0:1:0', note: 'A3', duration: '8n' }, { time: '0:1:2', note: 'B3', duration: '8n' },
        { time: '0:2:0', note: 'C4', duration: '8n' }, { time: '0:2:2', note: 'B3', duration: '8n' }, { time: '0:3:0', note: 'A3', duration: '8n' }, { time: '0:3:2', note: 'G3', duration: '8n' },
        { time: '1:0:0', note: 'E3', duration: '8n' }, { time: '1:0:2', note: 'G3', duration: '8n' }, { time: '1:1:0', note: 'A3', duration: '8n' }, { time: '1:1:2', note: 'G3', duration: '8n' }, { time: '1:2:0', note: 'E3', duration: '2n.' },
        { time: '2:0:0', note: 'E3', duration: '8n' }, { time: '2:0:2', note: 'G3', duration: '8n' }, { time: '2:1:0', note: 'A3', duration: '8n' }, { time: '2:1:2', note: 'B3', duration: '8n' },
        { time: '2:2:0', note: 'C4', duration: '8n' }, { time: '2:2:2', note: 'B3', duration: '8n' }, { time: '2:3:0', note: 'A3', duration: '8n' }, { time: '2:3:2', note: 'G3', duration: '8n' },
        { time: '3:0:0', note: 'E3', duration: '8n' }, { time: '3:0:2', note: 'G3', duration: '8n' }, { time: '3:1:0', note: 'A3', duration: '8n' }, { time: '3:1:2', note: 'G3', duration: '8n' }, { time: '3:2:0', note: 'B3', duration: '2n.' },
    ]);
    chorusGuitarPart.loop = true;
    chorusGuitarPart.loopEnd = '4m';

    let currentSection = '';
    const sectionScheduler = new Tone.Loop(time => {
        const currentTone = Tone;
        if (!currentTone) return; // THIS IS THE CORRECT FIX
        const measures = Math.floor(currentTone.Transport.getTicksAtTime(time) / (currentTone.Transport.PPQ * 4));
        const measureNum = measures % 32;

        let newSection = '';
        if (measureNum >= 24 || (measureNum >= 8 && measureNum < 16)) {
            newSection = ROLES.CHORUS;
        } else {
            newSection = ROLES.VERSE;
        }

        if (newSection !== currentSection) {
            console.log(`[GEMINI_DIAG_LOG] ★★★ セクション変更！'${currentSection || 'None'}' -> '${newSection}' at ${time} ★★★`);
            currentSection = newSection;
            const allParts = [verseKickSeq, verseSnareSeq, verseRideSeq, verseGuitarRiff, chorusKickSeq, chorusSnareSeq, chorusCrashSeq, chorusGuitarPart];
            allParts.forEach(part => { 
                if (part.state === 'started') {
                    console.log(`[GEMINI_DIAG_LOG]    - 即時停止: ${part.constructor.name}`);
                    part.stop(currentTone.immediate());
                }
            });

            if (newSection === ROLES.VERSE) {
                console.log(`[GEMINI_DIAG_LOG]    - VERSE パートを '${time}' に開始予約`);
                verseKickSeq.start(time);
                verseSnareSeq.start(time);
                verseRideSeq.start(time);
                verseGuitarRiff.start(time);
            } else { // CHORUS
                console.log(`[GEMINI_DIAG_LOG]    - CHORUS パートを '${time}' に開始予約`);
                chorusKickSeq.start(time);
                chorusSnareSeq.start(time);
                chorusCrashSeq.start(time);
                chorusGuitarPart.start(time);
            }
        }
    }, "1m").start(0);
    
    scheduledEvents.push(sectionScheduler, verseKickSeq, verseSnareSeq, verseRideSeq, verseGuitarRiff, chorusKickSeq, chorusSnareSeq, chorusCrashSeq, chorusGuitarPart);
    console.log(`[GEMINI_DIAG_LOG] createLiteStyleRock: ${scheduledEvents.length} 個の独立イベントをスケジュールしました。`);
    return true;
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
              <svg :xmlns="'http://www.w3.org/2000/svg'" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('リラックス・デカフェ')" :class="{ 'is-active': selectedMenu === 'リラックス・デカフェ' }">
            <div class="menu-content">
              <span class="menu-title">リラックス・デカフェ</span>
              <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
            </div>
            <div v-if="selectedMenu === 'リラックス・デカフェ' && isPlaying" class="active-indicator">
              <svg :xmlns="'http://www.w3.org/2000/svg'" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ジャズ・スペシャル')" :class="{ 'is-active': selectedMenu === 'ジャズ・スペシャル' }">
            <div class="menu-content">
              <span class="menu-title">ジャズ・スペシャル</span>
              <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
            </div>
            <div v-if="selectedMenu === 'ジャズ・スペシャル' && isPlaying" class="active-indicator">
              <svg :xmlns="'http://www.w3.org/2000/svg'" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" :class="{ 'is-active': selectedMenu === 'Lo-Fi・ビター' }">
            <div class="menu-content">
              <span class="menu-title">Lo-Fi・ビター</span>
              <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
            </div>
            <div v-if="selectedMenu === 'Lo-Fi・ビター' && isPlaying" class="active-indicator">
              <svg :xmlns="'http://www.w3.org/2000/svg'" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ロック・ビート')" :class="{ 'is-active': selectedMenu === 'ロック・ビート' }">
            <div class="menu-content">
              <span class="menu-title">ロック・ビート</span>
              <span class="menu-description">魂を揺さぶる、力強いリズムと歪んだギターのブレンド。</span>
            </div>
            <div v-if="selectedMenu === 'ロック・ビート' && isPlaying" class="active-indicator">
              <svg :xmlns="'http://www.w3.org/2000/svg'" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('LITE-Style・Post Rock')" :class="{ 'is-active': selectedMenu === 'LITE-Style・Post Rock' }">
            <div class="menu-content">
              <span class="menu-title">LITE-Style・Post Rock</span>
              <span class="menu-description">反復と構築。ミニマルな骨格が生むグルーヴ。</span>
            </div>
            <div v-if="selectedMenu === 'LITE-Style・Post Rock' && isPlaying" class="active-indicator">
              <svg :xmlns="'http://www.w3.org/2000/svg'" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
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
      :heavyMetalUrls="heavyMetalUrls"
      @close="closeSoundCheckModal"
      @playSound="handlePlaySound"
      @playRawSample="handlePlayRawSample"
      @update-param="handleUpdateParam"
      @save-params="handleSaveParams"
      @export-params="handleExportParams"
      @reset-params="handleResetParams"
      @set-extreme-eq="handleSetExtremeEq"
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