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

const soundSourceSelection = ref<Record<'eguitar' | 'ebass', 'sampler' | 'di'>>({
  eguitar: 'sampler',
  ebass: 'sampler'
});

// --- 音声関連 ---
let Tone: typeof ToneType | null = null;
let samplers: { [key: string]: { sampler: ToneType.Sampler, baseNote: string } } = {};
let diSamplers: { [key: string]: { sampler: ToneType.Sampler, baseNote: string } } = {};
let targetSamplers: { [key: string]: { sampler: ToneType.Sampler, baseNote: string } } = {};
let targetSamplerMulti: { sampler: ToneType.Sampler, baseNote: string } | null = null;

// NEW: Guitar Nuance Engine
let guitarVibrato: ToneType.Vibrato | null = null;
let guitarEQ: ToneType.EQ3 | null = null;


let rawSamplePlayers: ToneType.Players | null = null;
let targetGuitarPlayer: ToneType.Player | null = null;
let targetBassPlayer: ToneType.Player | null = null;
let reverb: ToneType.Reverb | null = null;
let chorus: ToneType.Chorus | null = null;
let delay: ToneType.PingPongDelay | null = null;
let masterComp: ToneType.Compressor | null = null;
let limiter: ToneType.Limiter | null = null;

let drumBusComp: ToneType.Compressor | null = null;
let drumBusVolume: ToneType.Volume | null = null;

// --- Virtual Amp Rig ---
let guitarInputGain: ToneType.Volume | null = null; 
let guitarPreDistComp: ToneType.Compressor | null = null;
let guitarPreEQ: ToneType.Filter | null = null; 
let guitarDistortion: ToneType.Distortion | null = null;
let guitarPostEQ: ToneType.EQ3 | null = null;
let guitarChorus: ToneType.Chorus | null = null;
let guitarCab: ToneType.Convolver | null = null;
let guitarMakeUpGain: ToneType.Volume | null = null;

// --- Bass Parallel Processing Rig ---
let bassInputGain: ToneType.Volume | null = null;
let bassPreDistComp: ToneType.Compressor | null = null;
let bassEQ: ToneType.EQ3 | null = null;
let bassDistortion: ToneType.Distortion | null = null;
let bassCab: ToneType.Convolver | null = null;
let bassMakeUpGain: ToneType.Volume | null = null;
let bassPostComp: ToneType.Compressor | null = null;
let bassSubFilter: ToneType.Filter | null = null;
let bassSubGain: ToneType.Volume | null = null;

let rideFilter: ToneType.Filter | null = null;

const scheduledEvents: (ToneType.Loop | ToneType.Part | ToneType.Sequence)[] = [];
let noise: ToneType.Noise | null = null;
let isAudioInitialized = ref(false);

type AmpParams = {
  inputGain: number;
  preCompThreshold: number; preCompRatio: number; preCompAttack: number; preCompRelease: number;
  preEqFreq: number; preEqGain: number;
  distortion: number;
  postEqLow: number; postEqMid: number; postEqHigh: number;
  chorusDepth: number; chorusRate: number;
};
type BassAmpParams = {
  inputGain: number;
  preCompThreshold: number; preCompRatio: number; preCompAttack: number; preCompRelease: number;
  subBlend: number; drive: number;
  eqLow: number; eqMid: number; eqHigh: number;
};
type TuningParams = Record<string, any>;

const tuningParams = ref<TuningParams>({});
const LOCAL_STORAGE_KEY = 'otoya-tuning-params-v12-pro';

const masterTunedParams: TuningParams = {
  "eguitar": { "inputGain": 12, "preCompThreshold": -24, "preCompRatio": 4, "preCompAttack": 0.01, "preCompRelease": 0.1, "preEqFreq": 800, "preEqGain": 12, "distortion": 0.9, "postEqLow": 3, "postEqMid": -12, "postEqHigh": 6, "chorusDepth": 0.1, "chorusRate": 1.5 },
  "ebass": { "inputGain": 12, "preCompThreshold": -20, "preCompRatio": 4, "preCompAttack": 0.02, "preCompRelease": 0.2, "subBlend": 0.5, "drive": 0.3, "eqLow": 4, "eqMid": -2, "eqHigh": 2 },
  "piano": { "volume": 0, "attack": 0.01, "release": 1.0 }, "bass": { "volume": -3, "attack": 0.01, "release": 0.5 },
  "ride": { "volume": -9, "attack": 0.01, "release": 0.5 }, "brush": { "volume": -9, "attack": 0.01, "release": 0.2 },
  "epiano": { "volume": -3, "attack": 0.01, "release": 1 }, "kick": { "volume": 0, "attack": 0.01, "release": 0.2 },
  "snare": { "volume": -3, "attack": 0.01, "release": 0.2 }, "pad": { "volume": -6, "attack": 0.1, "release": 1 },
  "sax": { "volume": -3, "attack": 0.01, "release": 1 }, "trombone": { "volume": -3, "attack": 0.01, "release": 1 },
  "rockKick": { "volume": 0, "attack": 0.01, "release": 0.2 }, "rockSnare": { "volume": -3, "attack": 0.01, "release": 0.2 },
  "crash": { "volume": -9, "attack": 0.01, "release": 0.5 },
  "tomHigh": { "volume": -6, "attack": 0.01, "release": 0.4 }, "tomMid": { "volume": -6, "attack": 0.01, "release": 0.4 },
  "tomFloor": { "volume": -6, "attack": 0.01, "release": 0.4 },
  "target_eguitar": { "volume": 0, "attack": 0.001, "release": 0.5 },
  "target_ebass": { "volume": 0, "attack": 0.01, "release": 1.0 },
};

watch(tuningParams, (newParams) => {
  if (!isAudioInitialized.value || !Tone) return;
  
  const eguitarParams = newParams.eguitar as AmpParams;
  if (eguitarParams && guitarInputGain && guitarPreDistComp && guitarPreEQ && guitarDistortion && guitarPostEQ && guitarChorus) {
    guitarInputGain.volume.value = eguitarParams.inputGain;
    guitarPreDistComp.threshold.value = eguitarParams.preCompThreshold;
    guitarPreDistComp.ratio.value = eguitarParams.preCompRatio;
    guitarPreDistComp.attack.value = eguitarParams.preCompAttack;
    guitarPreDistComp.release.value = eguitarParams.preCompRelease;
    guitarPreEQ.frequency.value = eguitarParams.preEqFreq;
    guitarPreEQ.gain.value = eguitarParams.preEqGain;
    guitarDistortion.distortion = eguitarParams.distortion;
    guitarPostEQ.low.value = eguitarParams.postEqLow;
    guitarPostEQ.mid.value = eguitarParams.postEqMid;
    guitarPostEQ.high.value = eguitarParams.postEqHigh;
    guitarChorus.depth = eguitarParams.chorusDepth;
    guitarChorus.frequency.value = eguitarParams.chorusRate;
  }

  const ebassParams = newParams.ebass as BassAmpParams;
  if (ebassParams && bassInputGain && bassPreDistComp && bassSubGain && bassDistortion && bassEQ && bassMakeUpGain) {
      bassInputGain.volume.value = ebassParams.inputGain;
      bassPreDistComp.threshold.value = ebassParams.preCompThreshold;
      bassPreDistComp.ratio.value = ebassParams.preCompRatio;
      bassPreDistComp.attack.value = ebassParams.preCompAttack;
      bassPreDistComp.release.value = ebassParams.preCompRelease;
      const diGain = 1.0 - ebassParams.subBlend;
      bassMakeUpGain.volume.value = Tone.gainToDb(diGain * 16) + 8;
      bassSubGain.volume.value = Tone.gainToDb(ebassParams.subBlend * 2);
      bassDistortion.distortion = ebassParams.drive;
      bassEQ.low.value = ebassParams.eqLow;
      bassEQ.mid.value = ebassParams.eqMid;
      bassEQ.high.value = ebassParams.eqHigh;
  }
  
  for (const instrumentName in newParams) {
    const activeSampler = samplers[instrumentName] || targetSamplers[instrumentName] || diSamplers[instrumentName];
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
  isLoading.value = true;
  loadingMessage.value = '喫茶店の準備をしています...';

  const allSamplePaths: Record<string, string> = {
    piano: 'piano-c4.wav', bass: 'bass-c1.wav', ride: 'drum-ride.wav', brush: 'drum-brush.wav',
    epiano: 'epiano-c4.wav', kick: 'drum-kick.wav', snare: 'drum-snare.wav', pad: 'pad-cmaj7.wav',
    sax: 'sax-c4.wav', trombone: 'trombone-c3.wav',
    eguitar: 'eguitar-clean-c3.wav', ebass: 'ebass-di-e1.wav',
    rockKick: 'rock-kick.wav', rockSnare: 'rock-snare.wav', crash: 'drum-crash.wav',
    tomHigh: 'tom-high.wav', tomMid: 'tom-mid.wav', tomFloor: 'tom-floor.wav',
    target_ebass: 'ebass-e1.wav',
    target_eguitar: 'eguitar-dist-c4.wav',
  };
  instrumentList.value = Object.keys(masterTunedParams).filter(k => !k.startsWith('target_'));

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
    Tone = await import('tone');
    await Tone.start();
    loadingMessage.value = '店内の響きを調整しています...';
    
    limiter = new Tone.Limiter(-0.1).toDestination();
    masterComp = new Tone.Compressor({ threshold: -12, ratio: 3 }).connect(limiter);
    
    drumBusVolume = new Tone.Volume(-3).connect(masterComp);
    drumBusComp = new Tone.Compressor({ threshold: -20, ratio: 4, attack: 0.01, release: 0.1 }).connect(drumBusVolume);

    Tone.Destination.volume.value = Tone.gainToDb(volume.value);

    reverb = new Tone.Reverb({ decay: 2.5, preDelay: 0.01, wet: 0.3 }).connect(masterComp);
    chorus = new Tone.Chorus(4, 2.5, 0.7).connect(masterComp);
    delay = new Tone.PingPongDelay("8n", 0.2).connect(masterComp);
    
    guitarVibrato = new Tone.Vibrato(5, 0.02);
    guitarEQ = new Tone.EQ3({ low: -6, mid: 0, high: 3 });
    
    loadingMessage.value = '仮想アンプとAI奏者を準備しています...';
    const eguitarP = tuningParams.value.eguitar;
    guitarInputGain = new Tone.Volume(eguitarP.inputGain);
    guitarPreDistComp = new Tone.Compressor({ threshold: eguitarP.preCompThreshold, ratio: eguitarP.preCompRatio, attack: eguitarP.preCompAttack, release: eguitarP.preCompRelease });
    guitarPreEQ = new Tone.Filter({ type: 'peaking', frequency: eguitarP.preEqFreq, gain: eguitarP.preEqGain });
    guitarDistortion = new Tone.Distortion({ distortion: eguitarP.distortion, oversample: '4x' });
    guitarPostEQ = new Tone.EQ3({ low: eguitarP.postEqLow, mid: eguitarP.postEqMid, high: eguitarP.postEqHigh });
    guitarChorus = new Tone.Chorus(eguitarP.chorusRate, 0.3, eguitarP.chorusDepth).start();
    guitarCab = new Tone.Convolver('/ir-guitar-cab.wav');
    guitarMakeUpGain = new Tone.Volume(12);

    const ebassP = tuningParams.value.ebass;
    bassInputGain = new Tone.Volume(ebassP.inputGain);
    bassPreDistComp = new Tone.Compressor({ threshold: ebassP.preCompThreshold, ratio: ebassP.preCompRatio, attack: ebassP.preCompAttack, release: ebassP.preCompRelease });
    bassEQ = new Tone.EQ3({ low: ebassP.eqLow, mid: ebassP.eqMid, high: ebassP.eqHigh });
    bassDistortion = new Tone.Distortion(ebassP.drive); 
    bassCab = new Tone.Convolver('/ir-bass-cab.wav');
    bassMakeUpGain = new Tone.Volume(8);
    bassPostComp = new Tone.Compressor({threshold: -18, ratio: 4, attack: 0.02, release: 0.2});
    bassSubFilter = new Tone.Filter(120, 'lowpass');
    bassSubGain = new Tone.Volume(0);
    
    rideFilter = new Tone.Filter(10000, 'lowpass');
    
    targetGuitarPlayer = new Tone.Player('/C5_s6_01.wav').toDestination();
    targetBassPlayer = new Tone.Player('/ebass-e1.wav').toDestination();

    await Promise.all([guitarCab.load, bassCab.load, targetGuitarPlayer.load, targetBassPlayer.load]);
    loadingMessage.value = '楽器を最終調整しています...';

    const multiSampleUrls = {
      'E2':  'E2_s1_02.wav',  'F2':  'F2_s1_03.wav',
      'A2':  'A2_s2_02.wav',
      'D3':  'D3_s3_02.wav',
      'G3':  'G3_s4_02.wav',
      'B3':  'B3_s5_02.wav',
      'E4':  'E4_s6_02.wav',  'G4': 'G4_s6_02.wav',
      'B4':  'B4_s6_02.wav',
      'C5':  'C5_s6_01.wav',  'F5': 'F5_s6_02.wav',
      'G#5': 'G#5_s6_02.wav',
    };
    const multiSampleParams = tuningParams.value['target_eguitar'];
    const loadedMultiSampler = new Tone.Sampler({
      urls: multiSampleUrls, baseUrl: "/",
      volume: multiSampleParams.volume, attack: multiSampleParams.attack, release: multiSampleParams.release
    });
    targetSamplerMulti = { sampler: loadedMultiSampler, baseNote: 'G#5' }; 
    console.log("LOG: New multi-sampled guitar loaded with URLs:", multiSampleUrls);

    
    for (const name of Object.keys(allSamplePaths)) {
      const params = tuningParams.value[name];
      if (!params) continue;

      const isDi = name === 'eguitar' || name === 'ebass';
      const isTarget = name.startsWith('target_');
      
      const urls: Record<string, string> = {};
      const filePath = allSamplePaths[name]!;
      let baseNote = '';
      
      if (name === 'target_eguitar') { baseNote = 'G#3'; }
      else if (filePath.includes('-c3')) { baseNote = 'C3'; }
      else if (filePath.includes('-e1')) { baseNote = 'E1'; }
      else { baseNote = 'C4'; }
      urls[baseNote] = filePath;

      const sampler = new Tone.Sampler({
        urls, baseUrl: "/", 
        volume: params.volume, attack: params.attack, release: params.release,
      });

      const samplerData = { sampler, baseNote };
      
      if (isDi) { diSamplers[name] = samplerData; } 
      else if (isTarget) { targetSamplers[name] = samplerData; }
      else { samplers[name] = samplerData; }
    }
    
    if (masterComp && reverb && chorus && delay && rideFilter && guitarInputGain && bassInputGain && bassSubFilter && drumBusComp && guitarVibrato && guitarEQ) {
      console.log("LOG: All core audio nodes initialized successfully.");
      const eguitarDI = diSamplers['eguitar'];
      if (eguitarDI) {
        eguitarDI.sampler.connect(guitarInputGain);
        guitarInputGain.chain(guitarPreDistComp, guitarPreEQ, guitarDistortion, guitarPostEQ, guitarChorus, guitarCab, guitarMakeUpGain);
        guitarMakeUpGain.fan(masterComp, reverb);
      }
      
      const ebassDI = diSamplers['ebass'];
      if (ebassDI) {
        ebassDI.sampler.connect(bassInputGain);
        ebassDI.sampler.connect(bassSubFilter);
        bassInputGain.chain(bassPreDistComp, bassEQ, bassDistortion, bassCab, bassMakeUpGain, bassPostComp);
        bassPostComp.fan(masterComp, reverb);
        bassSubFilter.chain(bassSubGain, masterComp);
      }
      
      const rockDrumKit = ['rockKick', 'rockSnare', 'crash', 'tomHigh', 'tomMid', 'tomFloor', 'ride'];

      for (const [name, data] of Object.entries(samplers)) {
        if (rockDrumKit.includes(name)) {
          if (name === 'ride') {
            data.sampler.connect(rideFilter);
            rideFilter.connect(drumBusComp);
          } else {
            data.sampler.connect(drumBusComp);
          }
          console.log(`LOG: Routing '${name}' to Drum Bus.`);
        } else {
           data.sampler.fan(masterComp, reverb, chorus, delay);
           console.log(`LOG: Routing '${name}' to Master Bus.`);
        }
      }
      
      for (const [name, { sampler }] of Object.entries(targetSamplers)) {
        sampler.fan(masterComp, reverb);
        console.log(`LOG: Routing '${name}' to Master Bus.`);
      }

      if(targetSamplerMulti) {
        targetSamplerMulti.sampler.chain(guitarVibrato, guitarEQ);
        guitarEQ.fan(masterComp, reverb);
        console.log("LOG: Routing multi-sampled guitar through Nuance Engine to Master Bus.");
      }

      const diEguitar = diSamplers['eguitar'];
      if (targetSamplerMulti && diEguitar) {
        samplers['eguitar'] = soundSourceSelection.value.eguitar === 'sampler' ? targetSamplerMulti : diEguitar;
      }

      const targetEbass = targetSamplers['target_ebass'];
      const diEbass = diSamplers['ebass'];
      if (targetEbass && diEbass) {
        samplers['ebass'] = soundSourceSelection.value.ebass === 'sampler' ? targetEbass : diEbass;
      }
    }

    rawSamplePlayers = await new Promise<ToneType.Players>((resolve) => {
      const p = new Tone!.Players({ urls: allSamplePaths, baseUrl: "/", onload: () => resolve(p) }).toDestination();
    });

    await Tone.loaded();
    isAudioInitialized.value = true;
    loadingMessage.value = '準備ができました';

  } catch (error: any) {
    loadingMessage.value = `エラーが発生しました: ${error.message}`;
    console.error("Error setting up Tone.js:", error);
  } finally {
    isLoading.value = false;
  }
};

onUnmounted(() => { stopMusic(); });

const playMusic = async (menuName: string, seed?: string) => {
  if (!isAudioInitialized.value) { await initializeAudio(); if (!isAudioInitialized.value) return; }
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
  }
  if (musicGenerated) {
    currentSeed.value = newSeed; 
    Tone!.Transport.start(); 
    isPlaying.value = true; 
    selectedMenu.value = menuName;
  } else if (!seed) { 
    alert('このメニューは現在準備中です。'); 
  }
};

const stopMusic = () => {
  if (!isPlaying.value || !Tone) return;
  Tone.Transport.stop(); 
  Tone.Transport.cancel(0);
  scheduledEvents.forEach(event => { event.stop(0); event.dispose(); });
  scheduledEvents.length = 0;
  if (noise) { noise.stop(0); noise.dispose(); noise = null; }
  Object.values(samplers).forEach(s => s.sampler.releaseAll());
  Object.values(diSamplers).forEach(s => s.sampler.releaseAll());
  Object.values(targetSamplers).forEach(s => s.sampler.releaseAll());
  if(targetSamplerMulti) targetSamplerMulti.sampler.releaseAll();
  isPlaying.value = false;
};

const togglePlayback = async () => { if (isPlaying.value) { stopMusic(); } else if (selectedMenu.value) { const [menuName, seed] = currentSeed.value.split(':'); if (menuName && seed) await playMusic(menuName, seed); } };
const handleVolumeChange = (event: Event) => { const newVolume = parseFloat((event.target as HTMLInputElement).value); volume.value = newVolume; if (isAudioInitialized.value && Tone) { Tone.Destination.volume.value = Tone.gainToDb(newVolume); } };
const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const copySeed = () => { if(currentSeed.value) navigator.clipboard.writeText(currentSeed.value); };
const playFromSeed = async () => { const [menuName, seed] = seedInput.value.split(':'); const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター', 'ロック・ビート']; if (menuName && seed && validMenus.includes(menuName)) { await playMusic(menuName, seed); } else { alert('レコード番号の形式が正しくないか、存在しないジャンルです。'); } };

const openSoundCheckModal = () => { isSoundCheckModalVisible.value = true; };
const closeSoundCheckModal = () => { isSoundCheckModalVisible.value = false; };
const handlePlaySound = async (instrumentName: string, type: 'sampler' | 'raw' | 'target' | 'target_sampler') => {
  if (!isAudioInitialized.value) { await initializeAudio(); if (!isAudioInitialized.value) { alert('音源の初期化に失敗しました。'); return; } }
  
  if (type === 'target_sampler') {
      if (instrumentName === 'target_eguitar' && targetSamplerMulti && Tone && guitarVibrato && guitarEQ) {
        const sampler = targetSamplerMulti.sampler;
        const now = Tone.now();
        const rng = Math.random;

        guitarVibrato.frequency.value = 4 + rng() * 2;
        guitarEQ.high.value = 2 + rng() * 2;
        
        sampler.triggerAttackRelease('E5', '1n', now);

      } else {
        const targetSampler = targetSamplers[instrumentName];
        if (targetSampler) targetSampler.sampler.triggerAttackRelease(targetSampler.baseNote, '1n');
      }
  } else {
      const samplerData = (instrumentName === 'eguitar' || instrumentName === 'ebass') ? diSamplers[instrumentName] : samplers[instrumentName];
      if (type === 'sampler' && samplerData) { samplerData.sampler.triggerAttackRelease(samplerData.baseNote, '1n'); }
      else if (type === 'raw' && rawSamplePlayers && rawSamplePlayers.has(instrumentName)) { rawSamplePlayers.player(instrumentName).start(); }
      else if (type === 'target' && instrumentName === 'eguitar' && targetGuitarPlayer) { targetGuitarPlayer.start(); }
      else if (type === 'target' && instrumentName === 'ebass' && targetBassPlayer) { targetBassPlayer.start(); }
  }
};
const handleUpdateParam = (payload: { instrument: string, param: string, value: any }) => {
  if (tuningParams.value[payload.instrument]) {
    const updatedInstrumentParams = { ...tuningParams.value[payload.instrument] };
    updatedInstrumentParams[payload.param] = payload.value;
    tuningParams.value[payload.instrument] = updatedInstrumentParams;
  }
  
  if (payload.instrument === 'target_eguitar' && targetSamplerMulti) {
    const sampler = targetSamplerMulti.sampler;
    switch (payload.param) {
      case 'volume': sampler.volume.value = payload.value; break;
      case 'attack': sampler.attack = payload.value; break;
      case 'release': sampler.release = payload.value; break;
    }
  }
};
const handleSaveParams = () => { try { localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(tuningParams.value)); alert('現在の設定をブラウザに保存しました。'); } catch (e) { alert('設定の保存に失敗しました。'); } };
const handleExportParams = () => { console.clear(); console.log(JSON.stringify(tuningParams.value, null, 2)); alert('現在の設定を開発者コンソールに出力しました。'); };
const handleResetParams = () => { 
  if (confirm('現在の調整を破棄し、全ての設定を初期値に戻します。よろしいですか？')) {
    tuningParams.value = JSON.parse(JSON.stringify(masterTunedParams)); 
    if (targetSamplerMulti) {
      const defaults = masterTunedParams['target_eguitar'];
      targetSamplerMulti.sampler.volume.value = defaults.volume;
      targetSamplerMulti.sampler.attack = defaults.attack;
      targetSamplerMulti.sampler.release = defaults.release;
    }
    alert('設定を初期化しました。');
  }
};
const handleSoundSourceChange = (payload: { instrument: 'eguitar' | 'ebass', source: 'sampler' | 'di' }) => {
  soundSourceSelection.value[payload.instrument] = payload.source;
  const targetSampler = payload.instrument === 'eguitar' ? targetSamplerMulti : targetSamplers[`target_${payload.instrument}`];
  const diSampler = diSamplers[payload.instrument];

  if (payload.source === 'sampler' && targetSampler) {
    samplers[payload.instrument] = targetSampler;
  } else if (payload.source === 'di' && diSampler) {
    samplers[payload.instrument] = diSampler;
  }
};

const ROLES = { BACKING: 'backing', GUITAR_SOLO: 'guitar_solo', DRUM_BREAK: 'drum_break' } as const;
type Role = typeof ROLES[keyof typeof ROLES];
type PartEvent = { time: string, note: string, dur: ToneType.Unit.Time };
type PartPattern = PartEvent[];
type BassPattern = (string | null)[];
type Section = { role: Role; duration: number; };

const createConcentrationSound = (rng: () => number): boolean => { 
    if (!Tone || !masterComp) return false;
    if (noise) { noise.stop(0); noise.dispose(); noise = null; }
    noise = new Tone.Noise("pink").start();
    const filter = new Tone.Filter(800, "lowpass").connect(masterComp);
    noise.connect(filter);
    const loop = new Tone.Loop((time) => { filter.frequency.rampTo(600 + rng() * 400, 4, time); }, "4m").start(0);
    scheduledEvents.push(loop);
    return true;
};
const createRelaxSound = (rng: () => number): boolean => {
    if (!samplers.pad) return false;
    const { sampler: padSampler, baseNote } = samplers.pad;
    padSampler.triggerAttack(baseNote);
    return true; 
};
const createLoFiSound = (rng: () => number): boolean => {
    if (!Tone || !samplers.epiano || !samplers.kick || !samplers.snare) return false;
    const kickSeq = new Tone.Sequence((time, note) => { if(note && samplers.kick) samplers.kick.sampler.triggerAttackRelease(samplers.kick.baseNote, '8n', time, 0.9 + rng() * 0.1); }, [1,0,1,0,1,0,1,0], "8n").start(0);
    const snareSeq = new Tone.Sequence((time, note) => { if(note && samplers.snare) samplers.snare.sampler.triggerAttackRelease(samplers.snare.baseNote, '8n', time, 0.8 + rng() * 0.2); }, [0,1,0,1], "4n").start(0);
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
    const { sampler: piano } = samplers.piano;
    const { sampler: bass } = samplers.bass;
    const { sampler: ride, baseNote: rideNote } = samplers.ride;
    const { sampler: sax } = samplers.sax;
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
    if (!Tone || !samplers.eguitar || !samplers.ebass || !guitarVibrato || !guitarEQ) return false;
    const { eguitar, ebass } = samplers;
    Tone.Transport.bpm.value = 110 + rng() * 60;
    Tone.Transport.swing = 0;

    const guitarBackingPatterns: PartPattern[] = [
        [{ time: '0:0', note: 'E3', dur: '4n' }, { time: '0:2', note: 'G3', dur: '8n' }, { time: '0:3', note: 'A3', dur: '8n' }],
        [{ time: '0:0', note: 'B3', dur: '2n' }, { time: '0:2', note: 'A3', dur: '4n' }],
        [{ time: '0:0', note: 'E3', dur: '8n' }, { time: '0:1', note: 'E3', dur: '8n' }, { time: '0:2', note: 'G3', dur: '4n' }],
    ];
    const guitarSoloPatterns: PartPattern[] = [
        [{ time: '0:0', note: 'E4', dur: '8n' }, { time: '0:1', note: 'G4', dur: '8n' }, { time: '0:2', note: 'A4', dur: '8n' }, { time: '0:3', note: 'B4', dur: '8n' }],
        [{ time: '0:0', note: 'B4', dur: '4n' }, { time: '0:2', note: 'A4', dur: '4n' }, { time: '0:3', note: 'G4', dur: '8n' }],
        [{ time: '0:0', note: 'G4', dur: '2n.' }, { time: '0:3', note: 'E4', dur: '4n' }],
    ];
    const bassBackingPatterns: BassPattern[] = [
        ['E1', 'G1', 'A1', 'B1'],
        ['E1', null, 'G1', null, 'A1', 'B1', null, 'A1'],
        ['E1', 'E1', 'G1', null, 'A1', null, 'A1', null],
    ];

    const songBlueprint: Section[] = [];
    let totalMeasures = 0; 
    let soloCooldown = 0;
    while (totalMeasures < 64) {
        let nextRole: Role = ROLES.BACKING;
        let nextDuration = 8;
        if (soloCooldown > 0) {
            nextRole = ROLES.BACKING;
            soloCooldown -= nextDuration;
        } else {
            const roll = rng();
            if (roll < 0.7) { nextRole = ROLES.BACKING; nextDuration = rng() < 0.5 ? 8 : 16; } 
            else if (roll < 0.9) { nextRole = ROLES.GUITAR_SOLO; nextDuration = 8; soloCooldown = 8; } 
            else { nextRole = ROLES.DRUM_BREAK; nextDuration = 4; soloCooldown = 8; }
        }
        songBlueprint.push({ role: nextRole, duration: nextDuration }); 
        totalMeasures += nextDuration; 
    }
    
    let measureCounter = 0;
    let currentSectionIndex = 0;
    let measuresIntoSection = 0;

    const masterLoop = new Tone.Loop(time => {
        if (!Tone || !guitarVibrato || !guitarEQ) return;

        const currentSection = songBlueprint[currentSectionIndex]!;
        const role = currentSection.role;

        if (measuresIntoSection === 0) {
            const sectionStartTime = time;
            if (role === ROLES.GUITAR_SOLO && ebass && samplers.ride) {
                ebass.sampler.volume.rampTo(-6, 0.5, sectionStartTime);
                samplers.ride.sampler.volume.rampTo(tuningParams.value.ride.volume - 6, 0.5, sectionStartTime);
            } else if (ebass && samplers.ride) {
                ebass.sampler.volume.rampTo(0, 0.5, sectionStartTime);
                samplers.ride.sampler.volume.rampTo(tuningParams.value.ride.volume, 0.5, sectionStartTime);
            }
        }
        
        if (role !== ROLES.DRUM_BREAK) {
            const guitarPattern = role === ROLES.GUITAR_SOLO ? guitarSoloPatterns[Math.floor(rng() * guitarSoloPatterns.length)]! : guitarBackingPatterns[Math.floor(rng() * guitarSoloPatterns.length)]!;
            guitarPattern.forEach(noteEvent => {
                const noteTime = time + Tone!.Time(noteEvent.time).toSeconds();
                if (soundSourceSelection.value.eguitar === 'sampler' && eguitar.sampler) {
                    guitarVibrato!.frequency.value = 4 + rng() * 2;
                    guitarEQ!.high.value = 2 + rng() * 2;
                }
                eguitar.sampler.triggerAttackRelease(noteEvent.note, noteEvent.dur, noteTime);
            });

            const bassPattern = bassBackingPatterns[Math.floor(rng() * bassBackingPatterns.length)]!;
            bassPattern.forEach((note, index) => {
                if (note) {
                    const noteTime = time + index * Tone!.Time('8n').toSeconds();
                    ebass.sampler.triggerAttackRelease(note, '8n', noteTime, 0.9);
                }
            });
        }
        
        createRockDrums(rng, samplers, time, role, measuresIntoSection);

        measureCounter++;
        measuresIntoSection++;
        if (measuresIntoSection >= currentSection.duration) {
            currentSectionIndex++;
            measuresIntoSection = 0;
            if (currentSectionIndex >= songBlueprint.length) {
                masterLoop.stop(0);
            }
        }
    }, '1m').start(0);
    
    scheduledEvents.push(masterLoop);
    return true;
};

const createRockDrums = (rng: () => number, instruments: typeof samplers, time: number, role: Role, measureInSec: number) => {
    if (!Tone) return;

    if (measureInSec === 0 && instruments.crash) { 
        instruments.crash.sampler.triggerAttack('C4', time);
    }
    
    if (role === ROLES.DRUM_BREAK) {
        if (measureInSec % 2 === 0 && instruments.tomFloor) instruments.tomFloor.sampler.triggerAttack('C4', time);
        else if (instruments.rockSnare) instruments.rockSnare.sampler.triggerAttack('C4', time);
        return;
    }
    
    const isFillMeasure = measureInSec % 4 === 3;
    const basePattern = { kick: [1,0,0,0,1,0,0,0], snare: [0,0,1,0,0,0,1,0], ride: [1,1,1,1,1,1,1,1] };

    for (let i = 0; i < 8; i++) {
        const stepTime = time + i * Tone.Time('8n').toSeconds();
        if (isFillMeasure && i >= 6) {
            if (i === 6 && instruments.tomMid) instruments.tomMid.sampler.triggerAttack('C4', stepTime);
            if (i === 7 && instruments.tomFloor) instruments.tomFloor.sampler.triggerAttack('C4', stepTime);
            continue;
        }
        if (basePattern.kick[i] && instruments.rockKick) instruments.rockKick.sampler.triggerAttack('C4', stepTime, 1.0);
        if (basePattern.snare[i] && instruments.rockSnare) instruments.rockSnare.sampler.triggerAttack('C4', stepTime, 0.9);
        if (basePattern.ride[i] && instruments.ride) {
            const vel = 0.5 + rng() * 0.5;
            if (rideFilter) rideFilter.frequency.value = 2000 + (vel * 8000);
            instruments.ride.sampler.triggerAttack('C4', stepTime + (rng() - 0.5) * 0.02, vel);
        }
    }
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
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('リラックス・デカフェ')" :class="{ 'is-active': selectedMenu === 'リラックス・デカフェ' }">
            <div class="menu-content">
              <span class="menu-title">リラックス・デカフェ</span>
              <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
            </div>
            <div v-if="selectedMenu === 'リラックス・デカフェ' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ジャズ・スペシャル')" :class="{ 'is-active': selectedMenu === 'ジャズ・スペシャル' }">
            <div class="menu-content">
              <span class="menu-title">ジャズ・スペシャル</span>
              <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
            </div>
            <div v-if="selectedMenu === 'ジャズ・スペシャル' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" :class="{ 'is-active': selectedMenu === 'Lo-Fi・ビター' }">
            <div class="menu-content">
              <span class="menu-title">Lo-Fi・ビター</span>
              <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
            </div>
            <div v-if="selectedMenu === 'Lo-Fi・ビター' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <button class="menu-button" @click="playMusic('ロック・ビート')" :class="{ 'is-active': selectedMenu === 'ロック・ビート' }">
            <div class="menu-content">
              <span class="menu-title">ロック・ビート</span>
              <span class="menu-description">魂を揺さぶる、力強いリズムと歪んだギターのブレンド。</span>
            </div>
            <div v-if="selectedMenu === 'ロック・ビート' && isPlaying" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
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
      :soundSourceSelection="soundSourceSelection"
      @close="closeSoundCheckModal"
      @playSound="handlePlaySound"
      @update-param="handleUpdateParam"
      @save-params="handleSaveParams"
      @export-params="handleExportParams"
      @reset-params="handleResetParams"
      @update-sound-source="handleSoundSourceChange"
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