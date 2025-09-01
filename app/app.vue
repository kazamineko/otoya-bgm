<!-- app.vue -->
<template>
  <div id="app" :class="`bg-${currentGenre || 'default'}`">
    <div class="container">
      <header class="header">
        <p class="loading-message" v-if="isLoading">{{ loadingMessage }}</p>
        <h1>AI-BGM 喫茶「おとや」</h1>
      </header>
      <main class="main-content">
        <p class="instruction">本日のBGMをお選びください</p>
        <div class="menu">
          <button @click="playGenre('focus')" :class="{ active: currentGenre === 'focus' }" :disabled="isLoading">
            <strong>集中ブレンド</strong>
            <small>思考を妨げない、静かな雨音のような音楽。</small>
          </button>
          <button @click="playGenre('relax')" :class="{ active: currentGenre === 'relax' }" :disabled="isLoading">
            <strong>リラックス・デカフェ</strong>
            <small>心のコリをほぐす、優しい陽だまりのような音楽。</small>
          </button>
          <button @click="playGenre('jazz')" :class="{ active: currentGenre === 'jazz' }" :disabled="isLoading">
            <strong>ジャズ・スペシャル</strong>
            <small>夜の静寂に寄り添う、マスターこだわりの一杯。</small>
          </button>
          <button @click="playGenre('lofi')" :class="{ active: currentGenre === 'lofi' }" :disabled="isLoading">
            <strong>Lo-Fi・ビター</strong>
            <small>懐かしいレコードに針を落とす、あの感覚をあなたに。</small>
          </button>
        </div>
        <div class="controls">
          <button class="play-pause-button" @click="togglePlayPause" :disabled="!currentGenre || isLoading">
            {{ isPlaying ? '■' : '▶' }}
          </button>
        </div>
        <div class="seed-display" v-if="currentSeed">
          <p>レコード番号 (シード値):</p>
          <div class="seed-value">
            <span>{{ currentSeed }}</span>
            <button @click="copySeed" class="copy-button" title="コピー">📄</button>
          </div>
          <button class="play-from-seed-button" @click="playFromCurrentSeed">このレコードを聴く</button>
        </div>
      </main>
    </div>
  </div>
</template>

<script>
import seedrandom from 'seedrandom';

export default {
  data() {
    return {
      // Audio State
      audioContext: null,
      gainNode: null,
      convolverNode: null,
      sampler: {}, // { 'instrument': AudioBuffer, ... }
      
      // ===【重要】エラー解決のための状態管理 ===
      playingSources: [], // ■ 再生中のAudioBufferSourceNodeのみを管理する配列
      schedulerTimerId: null, // ■ スケジューラのタイマーID
      nextNoteTime: 0.0, // ■ 次のノートの再生開始時間

      // UI State
      isLoading: true,
      loadingMessage: 'マスターが楽器の準備をしています...',
      isPlaying: false,
      currentGenre: null,
      currentSeed: '',
      rng: null, // 乱数生成器
    };
  },

  mounted() {
    // ユーザーの初回アクションを待つため、ここでは初期化しない
    // Safari/Chromeの自動再生ポリシーに対応
  },

  methods: {
    /**
     * オーディオコンテキストと音源の初期化
     */
    async initAudio() {
      if (this.audioContext) return; // 初期化済みなら何もしない

      this.loadingMessage = 'マスターが楽器の準備をしています...';
      this.isLoading = true;

      try {
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.gainNode = this.audioContext.createGain();
        this.convolverNode = this.audioContext.createConvolver();
        this.gainNode.connect(this.convolverNode);
        this.convolverNode.connect(this.audioContext.destination);

        await this.createReverb();
        await this.loadSamples();

        this.loadingMessage = '準備ができました。';
      } catch (e) {
        console.error('オーディオの初期化に失敗しました:', e);
        this.loadingMessage = 'エラー: 楽器の準備に失敗しました。';
      } finally {
        this.isLoading = false;
      }
    },

    /**
     * サンプル音源の読み込み
     */
    async loadSamples() {
      const sampleMap = {
        'piano-c4': '/piano-c4.wav',
        'bass-c1': '/bass-c1.wav',
        'drum-ride': '/drum-ride.wav',
        'drum-brush': '/drum-brush.wav',
        'epiano-c4': '/epiano-c4.wav',
        'drum-kick': '/drum-kick.wav',
        'drum-snare': '/drum-snare.wav',
        'pad-cmaj7': '/pad-cmaj7.wav',
      };
      
      const loadPromises = Object.entries(sampleMap).map(async ([name, path]) => {
        const response = await fetch(path);
        const arrayBuffer = await response.arrayBuffer();
        const audioBuffer = await this.audioContext.decodeAudioData(arrayBuffer);
        this.sampler[name] = audioBuffer;
      });
      
      await Promise.all(loadPromises);
    },

    /**
     * アルゴリズミック・リバーブの生成
     */
    async createReverb() {
      const sampleRate = this.audioContext.sampleRate;
      const length = sampleRate * 2; // 2秒のリバーブ
      const impulse = this.audioContext.createBuffer(2, length, sampleRate);
      const impulseL = impulse.getChannelData(0);
      const impulseR = impulse.getChannelData(1);

      for (let i = 0; i < length; i++) {
        impulseL[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / length, 2.5);
        impulseR[i] = (Math.random() * 2 - 1) * Math.pow(1 - i / length, 2.5);
      }
      this.convolverNode.buffer = impulse;
    },

    /**
     * ジャンルを選択して再生を開始
     */
    async playGenre(genre) {
      await this.initAudio(); // 未初期化なら初期化
      if (this.isLoading) return;

      // 違うジャンルが選択されたか、停止中だった場合は新しい曲を開始
      if (this.currentGenre !== genre || !this.isPlaying) {
        this.stopMusic(); // まず現在の曲を完全に停止
        
        this.currentGenre = genre;
        this.currentSeed = this.generateNewSeed();
        this.rng = seedrandom(this.currentSeed);
        
        this.startScheduler();
      }
    },
    
    /**
     * 再生/一時停止ボタンのトグル
     */
    togglePlayPause() {
        if (this.isPlaying) {
            this.stopMusic();
        } else {
            if (this.currentGenre) {
                this.startScheduler();
            }
        }
    },

    /**
     * 現在のシード値で再生
     */
    playFromCurrentSeed() {
      if (this.currentGenre) {
        this.stopMusic();
        this.rng = seedrandom(this.currentSeed);
        this.startScheduler();
      }
    },

    /**
     * 音楽スケジューラを開始
     */
    startScheduler() {
      if (this.schedulerTimerId !== null) return; // 既に動いていれば何もしない

      this.isPlaying = true;
      this.nextNoteTime = this.audioContext.currentTime + 0.1; // 少し未来から開始
      
      // 200msごとに次のノートをスケジューリング
      this.schedulerTimerId = setInterval(this.scheduleNotes, 200);
    },

    /**
     * ノートのスケジューリング処理（スケジューラの本体）
     */
    scheduleNotes() {
        const scheduleAheadTime = 0.3; // 300ms先までスケジューリング
        
        while (this.nextNoteTime < this.audioContext.currentTime + scheduleAheadTime) {
            this.generateAndPlayNote(this.nextNoteTime);
            
            // 次のノートの時間を決定（ジャンルごとに変える）
            const tempo = this.getTempoForGenre(this.currentGenre);
            this.nextNoteTime += 60.0 / tempo / 2; // 8分音符間隔
        }
    },

    /**
     * ノートを生成して再生をスケジュール
     */
    generateAndPlayNote(time) {
        // ここに各ジャンルごとの音楽生成ロジックを実装
        // この例ではシンプルにランダムな音を鳴らす
        let sampleName = null;
        if (this.currentGenre === 'jazz') {
            const instruments = ['piano-c4', 'bass-c1', 'drum-ride', 'drum-brush'];
            sampleName = instruments[Math.floor(this.rng() * instruments.length)];
        } else if (this.currentGenre === 'lofi') {
            const instruments = ['epiano-c4', 'drum-kick', 'drum-snare'];
            sampleName = instruments[Math.floor(this.rng() * instruments.length)];
        } // ... 他のジャンルも同様に ...
        
        if (sampleName && this.sampler[sampleName]) {
            this.playSound(sampleName, time);
        }
    },

    /**
     * 指定したサンプルを指定時間に再生
     */
    playSound(sampleName, time) {
        const source = this.audioContext.createBufferSource();
        source.buffer = this.sampler[sampleName];
        source.connect(this.gainNode);
        
        // ■■■【修正の核心 ①】再生開始をスケジュール ■■■
        source.start(time);
        
        // ■■■【修正の核心 ②】再生がスケジュールされたノードを追跡リストに追加 ■■■
        this.playingSources.push(source);

        // 再生が終了したら、追跡リストから自動的に削除する（メモリリーク対策）
        source.onended = () => {
            this.playingSources = this.playingSources.filter(s => s !== source);
        };
    },

    /**
     * ■■■【最重要】音楽を安全に停止する関数（全面改修）■■■
     */
    stopMusic() {
      // 1. スケジューラを停止し、新たなノートが追加されないようにする
      if (this.schedulerTimerId !== null) {
        clearInterval(this.schedulerTimerId);
        this.schedulerTimerId = null;
      }
      
      // 2. 現在再生中または再生がスケジュールされている全ての音源に対して停止命令を送る
      //    このリストにあるノードは必ず.start()が呼ばれていることが保証されている
      this.playingSources.forEach(source => {
        try {
          source.stop(0);
        } catch (e) {
          // この設計では理論上エラーは発生しないが、念のためハンドリング
          console.error('An unexpected error occurred while stopping a source node:', e);
        }
      });
      
      // 3. 追跡リストをクリアする
      this.playingSources = [];

      // 4. 再生状態フラグを更新
      this.isPlaying = false;
    },
    
    /**
     * ジャンルに応じたテンポを取得
     */
    getTempoForGenre(genre) {
        switch(genre) {
            case 'focus': return 80;
            case 'relax': return 60;
            case 'jazz': return 110;
            case 'lofi': return 85;
            default: return 90;
        }
    },

    /**
     * 新しいシード値を生成
     */
    generateNewSeed() {
      return Math.random().toString(36).substring(2, 10).toUpperCase();
    },
    
    /**
     * シード値をクリップボードにコピー
     */
    copySeed() {
      navigator.clipboard.writeText(this.currentSeed).then(() => {
        alert('レコード番号をコピーしました！');
      }).catch(err => {
        console.error('コピーに失敗しました', err);
      });
    }
  },
  
  beforeDestroy() {
    // コンポーネントが破棄される際に、全てのオーディオリソースを解放
    this.stopMusic();
    if (this.audioContext) {
      this.audioContext.close();
    }
  }
};
</script>

<style>
:root {
  --bg-default: #F5F5DC; /* ベージュ */
  --bg-focus: #E6F0F5; /* 薄い青 */
  --bg-relax: #FFFBEA; /* クリーム */
  --bg-jazz: #333;   /* ダークグレー */
  --bg-lofi: #5D4037; /* ブラウン */
  --text-color-light: #333;
  --text-color-dark: #f5f5f5;
  --accent-color: #D2691E; /* チョコレート */
  --active-color: #8B4513; /* サドルブラウン */
}

#app {
  font-family: 'Helvetica Neue', 'Arial', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  min-height: 100vh;
  transition: background-color 0.5s ease;
  padding: 2rem;
  box-sizing: border-box;
}

/* 背景色設定 */
#app.bg-default { background-color: var(--bg-default); color: var(--text-color-light); }
#app.bg-focus { background-color: var(--bg-focus); color: var(--text-color-light); }
#app.bg-relax { background-color: var(--bg-relax); color: var(--text-color-light); }
#app.bg-jazz { background-color: var(--bg-jazz); color: var(--text-color-dark); }
#app.bg-lofi { background-color: var(--bg-lofi); color: var(--text-color-dark); }
#app.bg-jazz button, #app.bg-lofi button { border-color: var(--text-color-dark); color: var(--text-color-dark); }
#app.bg-jazz button:hover, #app.bg-lofi button:hover { background-color: rgba(255,255,255,0.1); }
#app.bg-jazz button.active, #app.bg-lofi button.active { background-color: var(--text-color-dark); color: var(--bg-jazz); }


.container {
  max-width: 600px;
  margin: 0 auto;
}

.header h1 {
  font-size: 2.5rem;
  font-weight: bold;
  color: var(--accent-color);
  margin-bottom: 2rem;
}

.loading-message {
  position: fixed;
  top: 1rem;
  left: 50%;
  transform: translateX(-50%);
  background-color: rgba(0,0,0,0.7);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 5px;
  z-index: 10;
}

.instruction {
  margin-bottom: 1.5rem;
  font-size: 1.1rem;
}

.menu {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.menu button {
  background: none;
  border: 2px solid var(--text-color-light);
  padding: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
  border-radius: 8px;
  text-align: left;
}

.menu button:hover {
  background-color: rgba(0,0,0,0.05);
  transform: translateY(-2px);
}

.menu button.active {
  background-color: var(--accent-color);
  color: white;
  border-color: var(--accent-color);
  font-weight: bold;
}

.menu button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.menu button strong {
  display: block;
  font-size: 1.2rem;
  margin-bottom: 0.5rem;
}

.menu button small {
  font-size: 0.9rem;
}

.controls {
  margin-top: 2rem;
}

.play-pause-button {
  font-size: 3rem;
  background: none;
  border: none;
  cursor: pointer;
  width: 80px;
  height: 80px;
  line-height: 80px;
  border-radius: 50%;
  transition: all 0.3s ease;
}
.play-pause-button:hover:not(:disabled) {
    background-color: rgba(0,0,0,0.1);
}

.play-pause-button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.seed-display {
  margin-top: 2rem;
  background-color: rgba(0,0,0,0.05);
  padding: 1rem;
  border-radius: 8px;
}

.seed-value {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.5rem;
  font-family: 'Courier New', Courier, monospace;
  font-size: 1.5rem;
  font-weight: bold;
  margin: 0.5rem 0;
}

.copy-button {
  background: none;
  border: none;
  font-size: 1rem;
  cursor: pointer;
}

.play-from-seed-button {
  margin-top: 0.5rem;
  padding: 0.5rem 1rem;
  border-radius: 5px;
  cursor: pointer;
}
</style>