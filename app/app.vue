<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import seedrandom from 'seedrandom';
import * as Tone from 'tone';
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

// --- Tone.js関連 ---
let players: Tone.Players | null = null;
let distortion: Tone.Distortion | null = null;
let reverb: Tone.Reverb | null = null;
const scheduledEvents: (Tone.Loop | Tone.Part | Tone.Sequence)[] = [];
let noise: Tone.Noise | null = null;

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
    // 【修正】onMountedでのTone.start()呼び出しを削除。
    // 音声コンテキストの開始はユーザーアクション時に行う。

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
            onerror: (error: Error) => reject(error),
        }).toDestination();
    });

    // エフェクトチェインの設定
    if (players && reverb && distortion) {
        // リバーブに接続
        players.connect(reverb);
        // 特定の楽器をエフェクトにチェイン
        const eguitarPlayer = players.player('eguitar');
        if (eguitarPlayer) {
            eguitarPlayer.chain(distortion);
        }
        // ドラム類は直接出力
        const drumInstruments = ['kick', 'snare', 'rockKick', 'rockSnare', 'crash'];
        drumInstruments.forEach(drum => {
            const player = players!.player(drum);
            if (player) {
                player.toDestination();
            }
        });
    }

    loadingMessage.value = '準備ができました';
    isLoading.value = false;

  } catch (error: any) {
    loadingMessage.value = `エラーが発生しました: ${error.message}`;
    console.error("Error setting up Tone.js:", error);
  }
});

onUnmounted(() => {
    stopMusic();
});

const playMusic = async (menuName: string, seed?: string) => {
  if (isLoading.value) return;
  // 【重要】ユーザーアクションの起点であるこの場所でAudioContextを開始する
  if (Tone.context.state === 'suspended') await Tone.start();
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

  Tone.Transport.start();
  isPlaying.value = true;
  selectedMenu.value = menuName;
};

const stopMusic = () => {
  if (!isPlaying.value && scheduledEvents.length === 0) return;
  Tone.Transport.stop();
  Tone.Transport.cancel(0);

  scheduledEvents.forEach(event => {
    event.stop(0);
    event.dispose();
  });
  scheduledEvents.length = 0;

  if (noise) {
      noise.stop(0);
      noise.dispose();
      noise = null;
  }
  if (players) {
      players.stopAll();
  }
  isPlaying.value = false;
  selectedMenu.value = null;
};

// --- UIイベントハンドラ ---
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
  Tone.Destination.volume.value = Tone.gainToDb(newVolume);
};

const openModal = () => { isModalVisible.value = true; };
const closeModal = () => { isModalVisible.value = false; };

const copySeed = () => {
  navigator.clipboard.writeText(currentSeed.value);
};

const playFromSeed = () => {
  if (seedInput.value) {
    const [menuName, seed] = seedInput.value.split(':');
    const validMenus = ['集中ブレンド', 'リラックス・デカフェ', 'ジャズ・スペシャル', 'Lo-Fi・ビター', 'ロック・ビート'];
    if (menuName && seed && validMenus.includes(menuName)) {
      playMusic(menuName, seed);
    } else {
      alert('レコード番号の形式が正しくないか、存在しないジャンルです。');
    }
  }
};

// --- 音楽生成関数 ---

const createConcentrationSound = (rng: () => number) => {
    if (!noise) {
        noise = new Tone.Noise("pink").start();
    }
    const filter = new Tone.Filter(800, "lowpass").toDestination();
    noise.connect(filter);

    const loop = new Tone.Loop((time) => {
        filter.frequency.rampTo(600 + rng() * 400, 4);
    }, "4m").start(0);
    scheduledEvents.push(loop);
};

const createRelaxSound = (rng: () => number) => {
    if (!players) return;
    const padPlayer = players.player('pad');
    if (padPlayer) {
        padPlayer.loop = true;
        padPlayer.start();
    }
};

const createLoFiSound = (rng: () => number) => {
    if (!players) return;
    Tone.Transport.bpm.value = 80 + rng() * 15;

    const kickPattern = [[1,0,0,0,1,0,0,1,0,0,0,0,1,0,0,0], [1,0,0,0,1,0,1,0,1,0,0,0,1,0,1,0]][Math.floor(rng()*2)];
    const snarePattern = [[0,0,0,0,1,0,0,0,0,0,0,0,1,0,0,0], [0,0,0,0,1,0,0,0,0,0,1,0,1,0,0,0]][Math.floor(rng()*2)];
    const chords = [[-1200, -700, -400], [-1700, -1200, -900]];
    let currentChord = chords[Math.floor(rng()*2)];

    const kickSeq = new Tone.Sequence((time, note) => {
        if (note && players) {
            const kickPlayer = players.player('kick');
            if (kickPlayer) {
                kickPlayer.start(time);
            }
        }
    }, kickPattern, "16n").start(0);

    const snareSeq = new Tone.Sequence((time, note) => {
        if (note && players) {
            const snarePlayer = players.player('snare');
            if (snarePlayer) {
                snarePlayer.start(time);
            }
        }
    }, snarePattern, "16n").start(0);

    const chordLoop = new Tone.Loop((time) => {
        if (!players || !currentChord) return;
        const epianoPlayer = players.player('epiano');
        if (epianoPlayer) {
            currentChord.forEach((detune, i) => {
                epianoPlayer.playbackRate = Math.pow(2, detune / 1200);
                epianoPlayer.start(time + i * 0.02, 0, "1n");
            });
        }
    }, "1n").start(0);

    scheduledEvents.push(kickSeq, snareSeq, chordLoop);
};

const createRockSound = (rng: () => number) => {
    if (!players) return;
    Tone.Transport.bpm.value = 125 + rng() * 20;

    const guitar = players.player('eguitar');
    const bass = players.player('ebass');
    let isSolo = false;

    const drumPattern = [
        { time: '0:0', kick: true, snare: false, crash: true },
        { time: '0:1', kick: false, snare: false, crash: false },
        { time: '0:2', kick: true, snare: false, crash: false },
        { time: '0:3', kick: false, snare: false, crash: false },
        { time: '1:0', kick: false, snare: true, crash: false },
        { time: '1:1', kick: true, snare: false, crash: false },
        { time: '1:2', kick: false, snare: false, crash: false },
        { time: '1:3', kick: false, snare: true, crash: false },
    ];

    const drumPart = new Tone.Part((time, value) => {
        if (!players) return;
        if (value.kick) {
            const rockKickPlayer = players.player('rockKick');
            if (rockKickPlayer) rockKickPlayer.start(time);
        }
        if (value.snare) {
            const rockSnarePlayer = players.player('rockSnare');
            if (rockSnarePlayer) rockSnarePlayer.start(time);
        }
        if (value.crash) {
            const crashPlayer = players.player('crash');
            if (crashPlayer) crashPlayer.start(time);
        }
    }, drumPattern).start(0);
    drumPart.loop = true;
    drumPart.loopEnd = '2m';

    const bassPart = new Tone.Sequence((time, note) => {
        if (typeof note === 'number' && bass) {
            bass.playbackRate = Math.pow(2, note / 1200);
            bass.start(time, 0, "8n");
        }
    }, [-1200, -1200, -500, null, 0, 0, null, -200], "8n").start(0);
    bassPart.loop = true;

    const guitarPart = new Tone.Part((time, value) => {
        if (!guitar) return;
        if (isSolo) {
            if (rng() < 0.8) {
                const note = [-1200, -900, -700, -500, 0][Math.floor(rng()*5)];
                if (note !== undefined) {
                    const duration = ['8n', '16n'][Math.floor(rng()*2)];
                    guitar.playbackRate = Math.pow(2, note / 1200);
                    guitar.start(time, 0, duration);
                }
            }
        } else {
            guitar.playbackRate = Math.pow(2, value.detune / 1200);
            guitar.start(time, 0, value.dur);
        }
    }, [{time: '0', dur: '2n', detune: -1200}, {time: '0:2', dur: '4n', detune: -500}]).start(0);
    guitarPart.loop = true;
    guitarPart.loopEnd = '1m';

    const soloToggle = new Tone.Loop(() => {
        isSolo = !isSolo;
    }, '8m').start('4m');

    scheduledEvents.push(drumPart, bassPart, guitarPart, soloToggle);
};

const createJazzSound = (rng: () => number) => {
    if (!players) return;
    Tone.Transport.bpm.value = 100 + rng() * 20;
    Tone.Transport.swingSubdivision = "8n";
    Tone.Transport.swing = 0.05;

    const piano = players.player('piano');
    const bass = players.player('bass');
    const ride = players.player('ride');
    const sax = players.player('sax');
    let isSolo = false;

    const chords: { time: string; chord: string[]; }[] = [
        { time: '0:0', chord: ['E4', 'G4', 'B4', 'D5'] },
        { time: '1:0', chord: ['C4', 'E4', 'G4', 'B4'] },
        { time: '2:0', chord: ['F4', 'A4', 'C5', 'E5'] },
        { time: '3:0', chord: ['F4', 'A4', 'B4', 'E5'] }
    ];

    const pianoPart = new Tone.Part((time, value) => {
        if (!piano) return;
        if (isSolo && rng() < 0.7) return;
        const notesToPlay = rng() < 0.6 ? [value.chord[0], value.chord[2]] : value.chord;

        notesToPlay.forEach((note, i) => {
            piano.playbackRate = Tone.Frequency(note).toFrequency() / Tone.Frequency('C4').toFrequency();
            piano.start(time + i * 0.03, 0, "1n");
        });
    }, chords).start(0);
    pianoPart.loop = true;
    pianoPart.loopEnd = '4m';

    const bassPart = new Tone.Sequence((time, note) => {
        if (!bass) return;
        bass.playbackRate = Tone.Frequency(note).toFrequency() / Tone.Frequency('C1').toFrequency();
        bass.start(time, 0, "4n");
    }, ['C2', 'E2', 'G2', 'A2', 'D2', 'F2', 'A2', 'B2', 'G2', 'B2', 'D3', 'F3'], "4n").start(0);
    bassPart.loop = true;

    const ridePart = new Tone.Loop((time) => {
        if (!ride) return;
        ride.start(time);
    }, "4n").start(0);

    const saxPart = new Tone.Sequence((time, note) => {
        if (!isSolo || typeof note !== 'string' || !sax) return;
        sax.playbackRate = Tone.Frequency(note).toFrequency() / Tone.Frequency('C4').toFrequency();
        sax.start(time, 0, "8n");
    }, ['G4', 'A4', null, 'C5', 'A4', 'G4', null, 'E4'], "8n").start(0);
    saxPart.loop = true;

    const soloToggle = new Tone.Loop(() => {
        isSolo = !isSolo;
    }, '8m').start('4m');

    scheduledEvents.push(pianoPart, bassPart, ridePart, saxPart, soloToggle);
};
</script>

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
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <!-- リラックス・デカフェ -->
          <button class="menu-button" @click="playMusic('リラックス・デカフェ')" :class="{ 'is-active': selectedMenu === 'リラックス・デカフェ' }">
            <div class="menu-content">
              <span class="menu-title">リラックス・デカフェ</span>
              <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
            </div>
            <div v-if="selectedMenu === 'リラックス・デカフェ'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <!-- ジャズ・スペシャル -->
          <button class="menu-button" @click="playMusic('ジャズ・スペシャル')" :class="{ 'is-active': selectedMenu === 'ジャズ・スペシャル' }">
            <div class="menu-content">
              <span class="menu-title">ジャズ・スペシャル</span>
              <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
            </div>
            <div v-if="selectedMenu === 'ジャズ・スペシャル'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <!-- Lo-Fi・ビター -->
          <button class="menu-button" @click="playMusic('Lo-Fi・ビター')" :class="{ 'is-active': selectedMenu === 'Lo-Fi・ビター' }">
            <div class="menu-content">
              <span class="menu-title">Lo-Fi・ビター</span>
              <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
            </div>
            <div v-if="selectedMenu === 'Lo-Fi・ビター'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 2v2"/><path d="M14 2v2"/><path d="M16 8a1 1 0 0 1 1 1v8a4 4 0 0 1-4 4H7a4 4 0 0 1-4-4V9a1 1 0 0 1 1-1h14a4 4 0 1 1 0 8h-1"/><path d="M6 2v2"/></svg>
            </div>
          </button>
          <!-- ロック・ビート -->
          <button class="menu-button" @click="playMusic('ロック・ビート')" :class="{ 'is-active': selectedMenu === 'ロック・ビート' }">
            <div class="menu-content">
              <span class="menu-title">ロック・ビート</span>
              <span class="menu-description">魂を揺さぶる、力強いリズムと歪んだギターのブレンド。</span>
            </div>
            <div v-if="selectedMenu === 'ロック・ビート'" class="active-indicator">
              <svg xmlns="http://www.w3.org/2000/svg" :width="24" :height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" :stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"/></svg>
            </div>
          </button>
        </div>
        <div class="controls-container">
          <button @click="togglePlayback" class="control-button" :disabled="!selectedMenu && !isPlaying" :class="{ 'is-disabled': !selectedMenu && !isPlaying }">
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
    </div>
    <AboutModal :isVisible="isModalVisible" @close="closeModal" />
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