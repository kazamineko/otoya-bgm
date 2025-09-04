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

type TuningParams = Record<string, { volume: number; attack: number; release: number; distortion?: number }>;

const tuningParams = ref<TuningParams>({});
const LOCAL_STORAGE_KEY = 'otoya-tuning-params-v11-virtuoso'; // Virtuoso Build

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
    "eguitar": { "volume": 0, "attack": 0.01, "release": 1.5, "distortion": 0.9 },
    "ebass": { "volume": 6, "attack": 0.01, "release": 1.5 },
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
    
    // --- Virtual Amp Rig & Nuance Engine ---
    loadingMessage.value = '仮想アンプとAI奏者を準備しています...';
    guitarPreComp = new Tone.Compressor({threshold: -20, ratio: 4, attack: 0.01, release: 0.1});
    guitarDistortion = new Tone.Distortion(tuningParams.value.eguitar?.distortion ?? 0.9);
    guitarPostEQ = new Tone.EQ3({ low: 2, mid: -4, high: 4 });
    guitarCab = new Tone.Convolver('/ir-guitar-cab.wav');
    guitarMakeUpGain = new Tone.Volume(12);

    bassEQ = new Tone.EQ3({ low: 4, mid: 0, high: -2 });
    bassDistortion = new Tone.Distortion(0.2); 
    bassCab = new Tone.Convolver('/ir-bass-cab.wav');
    bassMakeUpGain = new Tone.Volume(8);
    bassPostComp = new Tone.Compressor({threshold: -18, ratio: 4, attack: 0.02, release: 0.2});
    bassSubFilter = new Tone.Filter(120, 'lowpass');
    bassSubGain = new Tone.Volume(0);
    
    rideFilter = new Tone.Filter(10000, 'lowpass');

    await Promise.all([guitarCab.load, bassCab.load]);
    loadingMessage.value = '楽器を最終調整しています...';
    
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
          sampler.chain(guitarPreComp, guitarDistortion, guitarPostEQ, guitarCab, guitarMakeUpGain);
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
        case 'kick': case 'rockKick': case 'tomHigh': case 'tomMid': case 'tomFloor':
        case 'snare': case 'rockSnare': case 'crash': case 'brush':
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

onUnmounted(() => { stopMusic(); });

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
  if (samplers) { Object.values(samplers).forEach(s => s.sampler.releaseAll()); }
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
  if (tuningParams.value[payload.instrument]) { (tuningParams.value[payload.instrument] as any)[payload.param] = payload.value; }
};
const handleSaveParams = () => {
  try {
    localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(tuningParams.value));
    alert('現在の設定をブラウザに保存しました。');
  } catch (e) { alert('設定の保存に失敗しました。'); console.error(e); }
};
const handleExportParams = () => {
  console.clear();
  console.log("--- 喫茶「おとや」マスター認定サウンドパラメータ ---");
  console.log(JSON.stringify(tuningParams.value, null, 2));
  alert('現在の設定を開発者コンソールに出力しました。(F12で確認)');
};

// --- 音楽生成関数 ---
const ROLES = { BACKING: 'backing', GUITAR_SOLO: 'guitar_solo', DRUM_BREAK: 'drum_break' } as const;
type Role = typeof ROLES[keyof typeof ROLES];

const createConcentrationSound = (rng: () => number) => { /* ... */ };
const createRelaxSound = (rng: () => number) => { /* ... */ };
const createLoFiSound = (rng: () => number) => { /* ... */ };
const createJazzSound = (rng: () => number) => { /* ... */ };

const createRockSound = (rng: () => number) => {
    if (!Tone || !rideFilter || !samplers.eguitar || !samplers.ebass || !samplers.rockKick || !samplers.rockSnare || !samplers.crash || !samplers.tomHigh || !samplers.tomMid || !samplers.tomFloor || !samplers.ride || !tuningParams.value.ebass || !tuningParams.value.ride) {
        return;
    }
    const { eguitar, ebass, rockKick, rockSnare, crash, tomHigh, tomMid, tomFloor, ride } = samplers;
    
    Tone.Transport.bpm.value = 110 + rng() * 60;

    // --- TypeScriptエラー修正: 型定義を整理・修正 ---
    // 最終的にTone.Partに渡され、スケジュールされるイベントの型 (時間は絶対秒数)
    type ScheduledGuitarEvent = { time: number; note: string; dur: ToneType.Unit.Time };
    type ScheduledBassEvent = { time: number; note: string | null };

    // 再利用可能なフレーズパターンを定義するための型 (時間は小節内の相対表記)
    type PatternNote = { time: string; note: string; dur: ToneType.Unit.Time };
    type GuitarPattern = PatternNote[];
    type BassPattern = (string | null)[];
    type Section = { role: Role; duration: number; };

    const guitarBackingPatterns: GuitarPattern[] = [
        [{ time: '0:0', note: 'E3', dur: '4n' }, { time: '0:2', note: 'G3', dur: '8n' }, { time: '0:3', note: 'A3', dur: '8n' }],
        [{ time: '0:0', note: 'B3', dur: '2n' }, { time: '0:2', note: 'A3', dur: '4n' }],
        [{ time: '0:0', note: 'E3', dur: '8n' }, { time: '0:1', note: 'E3', dur: '8n' }, { time: '0:2', note: 'G3', dur: '4n' }],
    ];
    const guitarSoloPatterns: GuitarPattern[] = [
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

    // --- AI Virtuoso: Weaving the Entire Performance ---
    const guitarEvents: ScheduledGuitarEvent[] = [];
    const bassEvents: ScheduledBassEvent[] = [];
    let currentTime = 0;

    songBlueprint.forEach(section => {
        const sectionStart = currentTime;
        const measureDuration = Tone!.Time('1m').toSeconds();
        
        // Dynamics Control
        if (section.role === ROLES.GUITAR_SOLO) {
            ebass.sampler.volume.rampTo(-6, 0.5, sectionStart);
            ride.sampler.volume.rampTo(-15, 0.5, sectionStart);
        } else {
            ebass.sampler.volume.rampTo(tuningParams.value.ebass!.volume, 0.5, sectionStart);
            ride.sampler.volume.rampTo(tuningParams.value.ride!.volume, 0.5, sectionStart);
        }

        for (let i = 0; i < section.duration; i++) {
            const measureStartTime = sectionStart + i * measureDuration;
            if (section.role !== ROLES.DRUM_BREAK) {
                // Weave Guitar Part
                const guitarPattern = section.role === ROLES.GUITAR_SOLO 
                    ? guitarSoloPatterns[Math.floor(rng() * guitarSoloPatterns.length)]!
                    : guitarBackingPatterns[Math.floor(rng() * guitarBackingPatterns.length)]!;
                
                guitarPattern.forEach(noteEvent => {
                    const noteTimeInSeconds = Tone!.Time(noteEvent.time).toSeconds();
                    const absoluteTime = measureStartTime + noteTimeInSeconds;
                    guitarEvents.push({ time: absoluteTime, note: noteEvent.note, dur: noteEvent.dur });
                });

                // Weave Bass Part
                const bassPattern = bassBackingPatterns[Math.floor(rng() * bassBackingPatterns.length)]!;
                bassPattern.forEach((note, index) => {
                    const noteTimeInSeconds = index * Tone!.Time('8n').toSeconds();
                    const absoluteTime = measureStartTime + noteTimeInSeconds;
                    bassEvents.push({ time: absoluteTime, note: note });
                });
            }
        }
        
        createRockDrums(rng, { rockKick, rockSnare, ride, crash, tomHigh, tomMid, tomFloor, rideFilter }, section.duration, sectionStart, section.role);
        currentTime += section.duration * measureDuration;
    });
    
    // --- Final Scheduling ---
    const guitarPart = new Tone.Part<ScheduledGuitarEvent>(((time, value) => {
        eguitar.sampler.triggerAttackRelease(value.note, value.dur, time, 0.9 + rng() * 0.1);
    }), guitarEvents).start(0);
    scheduledEvents.push(guitarPart);

    const bassPart = new Tone.Part<ScheduledBassEvent>(((time, value) => {
        if (value.note) ebass.sampler.triggerAttackRelease(value.note, '8n', time, 0.9);
    }), bassEvents).start(0);
    scheduledEvents.push(bassPart);
};

const createRockDrums = (rng: () => number, instruments: any, durationInMeasures: number, startTime: number, role: Role) => {
    const { rockKick: kick, rockSnare: snare, ride, crash, tomHigh, tomMid, tomFloor, rideFilter } = instruments;
    if (!Tone) return;

    type RhythmPatternData = { kick: number[], snare: number[], ride: number[] };
    const rhythmPatternLibrary: Record<string, RhythmPatternData> = {
        '8-Beat': { kick: [1,0,0,0,1,0,0,0], snare: [0,0,1,0,0,0,1,0], ride: [1,1,1,1,1,1,1,1] },
        '16-Beat': { kick: [1,0,1,0,1,0,1,0], snare: [0,0,1,0,0,0,1,0], ride: [1,1,1,1,1,1,1,1] }
    };

    const patternKeys = Object.keys(rhythmPatternLibrary);
    const basePattern = rhythmPatternLibrary[patternKeys[Math.floor(rng() * patternKeys.length)]!]!;

    let measureCount = 0;
    const drumLoop = new Tone.Loop(time => {
        if (role === ROLES.DRUM_BREAK) {
             if (measureCount % 2 === 0) tomFloor.sampler.triggerAttack('C4', time);
             else snare.sampler.triggerAttack('C4', time);
        } else {
            const isFillMeasure = measureCount % 4 === 3;
            for (let i = 0; i < 8; i++) {
                const stepTime = time + i * Tone!.Time('8n').toSeconds();
                if (isFillMeasure && i >= 6) {
                    if (i === 6) tomMid.sampler.triggerAttack('C4', stepTime);
                    if (i === 7) tomFloor.sampler.triggerAttack('C4', stepTime);
                    continue;
                }
                if (basePattern.kick[i]) kick.sampler.triggerAttack('C4', stepTime, 1.0);
                if (basePattern.snare[i]) snare.sampler.triggerAttack('C4', stepTime, 0.9);
                if (basePattern.ride[i]) {
                    const vel = 0.5 + rng() * 0.5;
                    rideFilter!.frequency.value = 2000 + (vel * 8000);
                    ride.sampler.triggerAttack('C4', stepTime + (rng() - 0.5) * 0.02, vel);
                }
            }
        }
        measureCount++;
    }, '1m').start(startTime).stop(startTime + durationInMeasures * Tone!.Time('1m').toSeconds());
    scheduledEvents.push(drumLoop);
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