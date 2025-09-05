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
let targetGuitarPlayer: ToneType.Player | null = null;
let targetBassPlayer: ToneType.Player | null = null;
let reverb: ToneType.Reverb | null = null;
let chorus: ToneType.Chorus | null = null;
let delay: ToneType.PingPongDelay | null = null;
let masterComp: ToneType.Compressor | null = null;
let limiter: ToneType.Limiter | null = null;

// --- Virtual Amp Rig ---
let guitarPreComp: ToneType.Compressor | null = null;
let guitarPreEQ: ToneType.Filter | null = null; 
let guitarDistortion: ToneType.Distortion | null = null;
let guitarPostEQ: ToneType.EQ3 | null = null;
let guitarChorus: ToneType.Chorus | null = null;
let guitarCab: ToneType.Convolver | null = null;
let guitarMakeUpGain: ToneType.Volume | null = null;

// --- Bass Parallel Processing Rig ---
let bassEQ: ToneType.EQ3 | null = null;
let bassDistortion: ToneType.Distortion | null = null;
let bassCab: ToneType.Convolver | null = null;
let bassMakeUpGain: ToneType.Volume | null = null;
let bassPostComp: ToneType.Compressor | null = null;
let bassSubFilter: ToneType.Filter | null = null;
let bassSubGain: ToneType.Volume | null = null;

// --- AI Drummer Nuance Engine ---
let rideFilter: ToneType.Filter | null = null;

const scheduledEvents: (ToneType.Loop | ToneType.Part | ToneType.Sequence)[] = [];
let noise: ToneType.Noise | null = null;
let isAudioInitialized = ref(false);

type AmpParams = {
  preEqFreq: number; preEqGain: number;
  distortion: number;
  postEqLow: number; postEqMid: number; postEqHigh: number;
  chorusDepth: number; chorusRate: number;
};
type BassAmpParams = {
  subBlend: number; drive: number;
  eqLow: number; eqMid: number; eqHigh: number;
};
type TuningParams = Record<string, any>;

const tuningParams = ref<TuningParams>({});
const LOCAL_STORAGE_KEY = 'otoya-tuning-params-v12-pro'; // Pro Build

const masterTunedParams: TuningParams = {
  "eguitar": { "preEqFreq": 800, "preEqGain": 12, "distortion": 0.9, "postEqLow": 3, "postEqMid": -12, "postEqHigh": 6, "chorusDepth": 0.1, "chorusRate": 1.5 },
  "ebass": { "subBlend": 0.5, "drive": 0.3, "eqLow": 4, "eqMid": -2, "eqHigh": 2 },
  "piano": { "volume": 0, "attack": 0.01, "release": 1.0 }, "bass": { "volume": -3, "attack": 0.01, "release": 0.5 },
  "ride": { "volume": -9, "attack": 0.01, "release": 0.5 }, "brush": { "volume": -9, "attack": 0.01, "release": 0.2 },
  "epiano": { "volume": -3, "attack": 0.01, "release": 1 }, "kick": { "volume": 0, "attack": 0.01, "release": 0.2 },
  "snare": { "volume": -3, "attack": 0.01, "release": 0.2 }, "pad": { "volume": -6, "attack": 0.1, "release": 1 },
  "sax": { "volume": -3, "attack": 0.01, "release": 1 }, "trombone": { "volume": -3, "attack": 0.01, "release": 1 },
  "rockKick": { "volume": 0, "attack": 0.01, "release": 0.2 }, "rockSnare": { "volume": -3, "attack": 0.01, "release": 0.2 },
  "crash": { "volume": -9, "attack": 0.01, "release": 0.5 },
  "tomHigh": { "volume": -6, "attack": 0.01, "release": 0.4 }, "tomMid": { "volume": -6, "attack": 0.01, "release": 0.4 },
  "tomFloor": { "volume": -6, "attack": 0.01, "release": 0.4 },
};

// Watcher to apply tuning changes in real-time
watch(tuningParams, (newParams) => {
  if (!isAudioInitialized.value || !Tone) return;
  
  // Apply eGuitar params
  const eguitarParams = newParams.eguitar as AmpParams;
  if (eguitarParams && guitarPreEQ && guitarDistortion && guitarPostEQ && guitarChorus) {
    guitarPreEQ.frequency.value = eguitarParams.preEqFreq;
    guitarPreEQ.gain.value = eguitarParams.preEqGain;
    guitarDistortion.distortion = eguitarParams.distortion;
    guitarPostEQ.low.value = eguitarParams.postEqLow;
    guitarPostEQ.mid.value = eguitarParams.postEqMid;
    guitarPostEQ.high.value = eguitarParams.postEqHigh;
    guitarChorus.depth = eguitarParams.chorusDepth; // .value is not used for depth
    guitarChorus.frequency.value = eguitarParams.chorusRate;
  }

  // Apply eBass params
  const ebassParams = newParams.ebass as BassAmpParams;
  if (ebassParams && bassSubGain && bassDistortion && bassEQ && bassMakeUpGain) {
      const diGain = 1.0 - ebassParams.subBlend;
      bassMakeUpGain.volume.value = Tone.gainToDb(diGain * 16) + 8; // DI path makeup gain
      bassSubGain.volume.value = Tone.gainToDb(ebassParams.subBlend * 2); // Sub path makeup gain
      bassDistortion.distortion = ebassParams.drive;
      bassEQ.low.value = ebassParams.eqLow;
      bassEQ.mid.value = ebassParams.eqMid;
      bassEQ.high.value = ebassParams.eqHigh;
  }
  
  // Apply general sampler params
  for (const instrumentName in newParams) {
    if (samplers[instrumentName]) {
      const sampler = samplers[instrumentName]!.sampler;
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
  };
  instrumentList.value = Object.keys(allSamplePaths);

  const initialParams = JSON.parse(JSON.stringify(masterTunedParams)); // Deep copy
  
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
    Tone.Destination.volume.value = Tone.gainToDb(volume.value);

    reverb = new Tone.Reverb({ decay: 2.5, preDelay: 0.01, wet: 0.3 }).connect(masterComp);
    chorus = new Tone.Chorus(4, 2.5, 0.7).connect(masterComp);
    delay = new Tone.PingPongDelay("8n", 0.2).connect(masterComp);
    
    loadingMessage.value = '仮想アンプとAI奏者を準備しています...';
    const eguitarP = tuningParams.value.eguitar;
    guitarPreComp = new Tone.Compressor({threshold: -20, ratio: 4, attack: 0.01, release: 0.1});
    guitarPreEQ = new Tone.Filter({ type: 'peaking', frequency: eguitarP.preEqFreq, gain: eguitarP.preEqGain });
    guitarDistortion = new Tone.Distortion({ distortion: eguitarP.distortion, oversample: '4x' });
    guitarPostEQ = new Tone.EQ3({ low: eguitarP.postEqLow, mid: eguitarP.postEqMid, high: eguitarP.postEqHigh });
    guitarChorus = new Tone.Chorus(eguitarP.chorusRate, 0.3, eguitarP.chorusDepth).start();
    guitarCab = new Tone.Convolver('/ir-guitar-cab.wav');
    guitarMakeUpGain = new Tone.Volume(12);

    const ebassP = tuningParams.value.ebass;
    bassEQ = new Tone.EQ3({ low: ebassP.eqLow, mid: ebassP.eqMid, high: ebassP.eqHigh });
    bassDistortion = new Tone.Distortion(ebassP.drive); 
    bassCab = new Tone.Convolver('/ir-bass-cab.wav');
    bassMakeUpGain = new Tone.Volume(8);
    bassPostComp = new Tone.Compressor({threshold: -18, ratio: 4, attack: 0.02, release: 0.2});
    bassSubFilter = new Tone.Filter(120, 'lowpass');
    bassSubGain = new Tone.Volume(0);
    
    rideFilter = new Tone.Filter(10000, 'lowpass');
    
    targetGuitarPlayer = new Tone.Player('/eguitar-dist-c4.wav').toDestination();
    targetBassPlayer = new Tone.Player('/ebass-e1.wav').toDestination();

    await Promise.all([guitarCab.load, bassCab.load, targetGuitarPlayer.load, targetBassPlayer.load]);
    loadingMessage.value = '楽器を最終調整しています...';
    
    for (const name of instrumentList.value) {
      const isAmped = name === 'eguitar' || name === 'ebass';
      const params = tuningParams.value[name]!;
      const urls: Record<string, string> = {};
      const filePath = allSamplePaths[name]!;
      let baseNote = '';
      
      if (filePath.includes('-c3')) { baseNote = 'C3'; }
      else if (filePath.includes('-e1')) { baseNote = 'E1'; }
      else { baseNote = 'C4'; }
      urls[baseNote] = filePath;

      const sampler = new Tone.Sampler({
        urls, baseUrl: "/", 
        volume: isAmped ? 0 : params.volume,
        attack: isAmped ? 0.01 : params.attack,
        release: isAmped ? 1.5 : params.release,
      });

      switch(name) {
        case 'eguitar':
          sampler.chain(guitarPreComp, guitarPreEQ, guitarDistortion, guitarPostEQ, guitarChorus, guitarCab, guitarMakeUpGain);
          guitarMakeUpGain.fan(masterComp, reverb);
          break;
        case 'ebass':
          sampler.connect(bassEQ);
          sampler.connect(bassSubFilter);
          bassEQ.chain(bassDistortion, bassCab, bassMakeUpGain, bassPostComp, masterComp);
          bassSubFilter.chain(bassSubGain, masterComp);
          break;
        case 'ride': 
          sampler.connect(rideFilter); 
          rideFilter.fan(masterComp, reverb); 
          break;
        case 'pad': 
          sampler.chain(reverb, masterComp); 
          break;
        default:
          const isDryInstrument = /kick|snare|tom|sax|crash/i.test(name);
          if (isDryInstrument) {
            // ディレイとコーラスをバイパスする、クリーンな経路
            sampler.fan(masterComp, reverb);
          } else {
            // 空間系・色彩楽器: 従来通り全てのエフェクトへ
            sampler.fan(masterComp, reverb, chorus, delay);
          }
          break;
      }
      samplers[name] = { sampler, baseNote };
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
  if (!isAudioInitialized.value) {
    await initializeAudio();
    if (!isAudioInitialized.value) return;
  }
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
  if (samplers) { Object.values(samplers).forEach(s => s.sampler.releaseAll()); }
  isPlaying.value = false;
};

// --- UI Event Handlers ---
const togglePlayback = async () => { if (isPlaying.value) { stopMusic(); } else if (selectedMenu.value) { const [menuName, seed] = currentSeed.value.split(':'); if (menuName && seed) await playMusic(menuName, seed); } };
const handleVolumeChange = (event: Event) => { const newVolume = parseFloat((event.target as HTMLInputElement).value); volume.value = newVolume; if (isAudioInitialized.value && Tone) { Tone.Destination.volume.value = Tone.gainToDb(newVolume); } };
const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };
const copySeed = () => { if(currentSeed.value) navigator.clipboard.writeText(currentSeed.value); };
const playFromSeed = async () => { const [menuName, seed] = seedInput.value.split(':'); const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター', 'ロック・ビート']; if (menuName && seed && validMenus.includes(menuName)) { await playMusic(menuName, seed); } else { alert('レコード番号の形式が正しくないか、存在しないジャンルです。'); } };

// --- Sound Tuning Modal Handlers ---
const openSoundCheckModal = () => { isSoundCheckModalVisible.value = true; };
const closeSoundCheckModal = () => { isSoundCheckModalVisible.value = false; };
const handlePlaySound = async (instrumentName: string, type: 'sampler' | 'raw' | 'target') => {
  if (!isAudioInitialized.value) { await initializeAudio(); if (!isAudioInitialized.value) { alert('音源の初期化に失敗しました。'); return; } }
  const samplerData = samplers[instrumentName];
  if (type === 'sampler' && samplerData) { samplerData.sampler.triggerAttackRelease(samplerData.baseNote, '1n'); }
  else if (type === 'raw' && rawSamplePlayers && rawSamplePlayers.has(instrumentName)) { rawSamplePlayers.player(instrumentName).start(); }
  else if (type === 'target' && instrumentName === 'eguitar' && targetGuitarPlayer) { targetGuitarPlayer.start(); }
  else if (type === 'target' && instrumentName === 'ebass' && targetBassPlayer) { targetBassPlayer.start(); }
};
const handleUpdateParam = (payload: { instrument: string, param: string, value: number }) => { if (tuningParams.value[payload.instrument]) { tuningParams.value[payload.instrument][payload.param] = payload.value; } };
const handleSaveParams = () => { try { localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(tuningParams.value)); alert('現在の設定をブラウザに保存しました。'); } catch (e) { alert('設定の保存に失敗しました。'); } };
const handleExportParams = () => { console.clear(); console.log(JSON.stringify(tuningParams.value, null, 2)); alert('現在の設定を開発者コンソールに出力しました。'); };
const handleResetParams = () => { 
  if (confirm('現在の調整を破棄し、全ての設定を初期値に戻します。よろしいですか？')) {
    tuningParams.value = JSON.parse(JSON.stringify(masterTunedParams)); 
    alert('設定を初期化しました。');
  }
};


// --- 音楽生成関数 ---
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
    const { sampler: epiano } = samplers.epiano;
    const { sampler: kick, baseNote: kickNote } = samplers.kick;
    const { sampler: snare, baseNote: snareNote } = samplers.snare;
    Tone.Transport.bpm.value = 80 + rng() * 15;
    const chords = [['C2', 'Eb2', 'G2'], ['A1', 'C2', 'E2']];
    let currentChord = chords[Math.floor(rng() * 2)]!;
    const kickSeq = new Tone.Sequence((time, note) => { if(note) kick.triggerAttackRelease(kickNote, '8n', time, 0.9 + rng() * 0.1); }, [1,0,1,0,1,0,1,0], "8n").start(0);
    const snareSeq = new Tone.Sequence((time, note) => { if(note) snare.triggerAttackRelease(snareNote, '8n', time, 0.8 + rng() * 0.2); }, [0,1,0,1], "4n").start(0);
    const chordLoop = new Tone.Loop((time) => { epiano.triggerAttackRelease(currentChord, '2n', time, 0.7 + rng() * 0.2); }, "1n").start(0);
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
    if (!Tone || !rideFilter || !samplers.eguitar || !samplers.ebass || !samplers.rockKick || !samplers.rockSnare || !samplers.crash || !samplers.tomHigh || !samplers.tomMid || !samplers.tomFloor || !samplers.ride) return false;
    const { eguitar, ebass, rockKick, rockSnare, crash, tomHigh, tomMid, tomFloor, ride } = samplers;
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
        if (!Tone) return;

        const currentSection = songBlueprint[currentSectionIndex]!;
        const role = currentSection.role;

        // --- Section Transition Logic ---
        if (measuresIntoSection === 0) {
            const sectionStartTime = time;
            if (role === ROLES.GUITAR_SOLO) {
                ebass.sampler.volume.rampTo(-6, 0.5, sectionStartTime);
                ride.sampler.volume.rampTo(tuningParams.value.ride.volume - 6, 0.5, sectionStartTime);
            } else {
                ebass.sampler.volume.rampTo(0, 0.5, sectionStartTime);
                ride.sampler.volume.rampTo(tuningParams.value.ride.volume, 0.5, sectionStartTime);
            }
        }
        
        // --- Instrument Pattern Generation for this measure ---
        if (role !== ROLES.DRUM_BREAK) {
            // Guitar
            const guitarPattern = role === ROLES.GUITAR_SOLO ? guitarSoloPatterns[Math.floor(rng() * guitarSoloPatterns.length)]! : guitarBackingPatterns[Math.floor(rng() * guitarBackingPatterns.length)]!;
            guitarPattern.forEach(noteEvent => {
                const noteTime = time + Tone!.Time(noteEvent.time).toSeconds();
                eguitar.sampler.triggerAttackRelease(noteEvent.note, noteEvent.dur, noteTime, 0.9 + rng() * 0.1);
            });

            // Bass
            const bassPattern = bassBackingPatterns[Math.floor(rng() * bassBackingPatterns.length)]!;
            bassPattern.forEach((note, index) => {
                if (note) {
                    const noteTime = time + index * Tone!.Time('8n').toSeconds();
                    ebass.sampler.triggerAttackRelease(note, '8n', noteTime, 0.9);
                }
            });
        }
        
        // --- Drums for this measure ---
        createRockDrums(rng, { rockKick, rockSnare, ride, crash, tomHigh, tomMid, tomFloor, rideFilter }, time, role, measuresIntoSection);

        // --- Advance Counters ---
        measureCounter++;
        measuresIntoSection++;
        if (measuresIntoSection >= currentSection.duration) {
            currentSectionIndex++;
            measuresIntoSection = 0;
            if (currentSectionIndex >= songBlueprint.length) {
                masterLoop.stop(0); // End of song
            }
        }
    }, '1m').start(0);
    
    scheduledEvents.push(masterLoop);
    return true;
};

const createRockDrums = (rng: () => number, instruments: any, time: number, role: Role, measureInSec: number) => {
    const { rockKick: kick, rockSnare: snare, ride, crash, tomHigh, tomMid, tomFloor, rideFilter } = instruments;
    if (!Tone) return;

    if (measureInSec === 0) { // Add crash on the first measure of a section
        crash.sampler.triggerAttack('C4', time);
    }
    
    if (role === ROLES.DRUM_BREAK) {
        if (measureInSec % 2 === 0) tomFloor.sampler.triggerAttack('C4', time);
        else snare.sampler.triggerAttack('C4', time);
        return;
    }
    
    const isFillMeasure = measureInSec % 4 === 3;
    const basePattern = { kick: [1,0,0,0,1,0,0,0], snare: [0,0,1,0,0,0,1,0], ride: [1,1,1,1,1,1,1,1] }; // 8-beat

    for (let i = 0; i < 8; i++) {
        const stepTime = time + i * Tone!.Time('8n').toSeconds();
        if (isFillMeasure && i >= 6) { // Simple fill on 4th measure
            if (i === 6) tomMid.sampler.triggerAttack('C4', stepTime);
            if (i === 7) tomFloor.sampler.triggerAttack('C4', stepTime);
            continue;
        }
        if (basePattern.kick[i]) kick.sampler.triggerAttack('C4', stepTime, 1.0);
        if (basePattern.snare[i]) snare.sampler.triggerAttack('C4', stepTime, 0.9);
        if (basePattern.ride[i]) {
            const vel = 0.5 + rng() * 0.5;
            rideFilter.frequency.value = 2000 + (vel * 8000);
            ride.sampler.triggerAttack('C4', stepTime + (rng() - 0.5) * 0.02, vel);
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
      @close="closeSoundCheckModal"
      @playSound="handlePlaySound"
      @update-param="handleUpdateParam"
      @save-params="handleSaveParams"
      @export-params="handleExportParams"
      @reset-params="handleResetParams"
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