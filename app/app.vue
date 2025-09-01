<script setup lang="ts">
import { ref, onMounted } from 'vue';

// --- リアクティブな状態変数 ---
const selectedMenu = ref<string | null>(null); // 選択されたメニュー名
const isPlaying = ref(false); // 再生中かどうか
const volume = ref(0.5); // 音量（0.0 ~ 1.0）

// --- Web Audio API関連の変数 ---
let audioContext: AudioContext;
let oscillator: OscillatorNode; // 音を生成するノード
let gainNode: GainNode; // 音量を制御するノード

// --- 関数定義 ---

// 音楽を再生する関数
const playMusic = (menuName: string) => {
  // すでに再生中なら何もしない
  if (isPlaying.value) {
    // もし違うメニューがクリックされたら、一度止めてから再生するなどの処理も可能
    return;
  }
  
  // AudioContextを初期化（ユーザー操作がきっかけでないと動かないため、ここで初期化）
  if (!audioContext) {
    audioContext = new AudioContext();
  }

  // 音を生成するOscillatorNodeを作成
  oscillator = audioContext.createOscillator();
  oscillator.type = 'sine'; // サイン波（ピーという音）
  oscillator.frequency.setValueAtTime(440, audioContext.currentTime); // 440Hz (ラの音)

  // 音量を制御するGainNodeを作成
  gainNode = audioContext.createGain();
  gainNode.gain.setValueAtTime(volume.value, audioContext.currentTime);

  // ノードを接続: Oscillator -> Gain -> 出力
  oscillator.connect(gainNode).connect(audioContext.destination);

  // 再生開始
  oscillator.start();
  
  // 状態を更新
  isPlaying.value = true;
  selectedMenu.value = menuName;
};

// 音楽を停止する関数
const stopMusic = () => {
  if (oscillator) {
    oscillator.stop();
  }
  isPlaying.value = false;
  selectedMenu.value = null;
};

// 再生/一時停止を切り替える関数
const togglePlayback = () => {
  if (isPlaying.value) {
    stopMusic();
  } else if(selectedMenu.value) {
    // 最後に選んだメニューで再生（この例では音が同じなのでmenuNameは使わない）
    playMusic(selectedMenu.value);
  }
};

// 音量を変更する関数
const handleVolumeChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const newVolume = parseFloat(target.value);
  volume.value = newVolume;
  if (gainNode) {
    gainNode.gain.setValueAtTime(newVolume, audioContext.currentTime);
  }
};
</script>

<template>
  <div class="background-container">
    <div class="content-panel">
      <!-- (タイトルやメニューボタンのHTMLは変更なし) -->
      <h1 class="title">AI-BGM 喫茶「おとや」</h1>
      <p class="subtitle">本日のBGMをお選びください</p>
      <div class="menu-container">
        <button class="menu-button" @click="playMusic('集中ブレンド')">
          <span class="menu-title">集中ブレンド</span>
          <span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span>
        </button>
        <button class="menu-button" @click="playMusic('リラックス・デカフェ')">
          <span class="menu-title">リラックス・デカフェ</span>
          <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
        </button>
        <button class="menu-button" @click="playMusic('ジャズ・スペシャル')">
          <span class="menu-title">ジャズ・スペシャル</span>
          <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
        </button>
        <button class="menu-button" @click="playMusic('Lo-Fi・ビター')">
          <span class="menu-title">Lo-Fi・ビター</span>
          <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
        </button>
      </div>

      <!-- ▼▼▼ ここからが追加した再生コントロールバー ▼▼▼ -->
      <div class="controls-container">
        <button @click="togglePlayback" class="control-button">
          {{ isPlaying ? '■' : '▶' }} <!-- 再生中は■、停止中は▶を表示 -->
        </button>
        <input 
          type="range" 
          min="0" 
          max="1" 
          step="0.01" 
          :value="volume" 
          @input="handleVolumeChange" 
          class="volume-slider"
        />
      </div>
      <!-- ▲▲▲ ここまで ▲▲▲ -->

    </div>
  </div>
</template>

<style>
/* ... (既存のスタイルは変更なし) ... */
body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif; }
.background-container { background-image: url('/bg-main.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
.content-panel { background-color: rgba(255, 255, 255, 0.85); padding: 20px 40px 30px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); backdrop-filter: blur(5px); width: 90%; max-width: 600px; text-align: center; }
.title { color: #363636; font-weight: bold; margin-bottom: 8px; }
.subtitle { color: #555; margin-top: 0; margin-bottom: 30px; }
.menu-container { display: flex; flex-direction: column; gap: 15px; }
.menu-button { background-color: #f5f5f5; border: 1px solid #dbdbdb; border-radius: 4px; padding: 15px 20px; cursor: pointer; transition: all 0.2s ease; text-align: left; display: flex; flex-direction: column; font-family: inherit; }
.menu-button:hover { background-color: #e8e8e8; border-color: #b5b5b5; transform: translateY(-2px); box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
.menu-title { font-size: 1.1em; font-weight: bold; color: #363636; }
.menu-description { font-size: 0.9em; color: #7a7a7a; margin-top: 4px; }

/* ▼▼▼ ここからが追加したスタイル ▼▼▼ */
.controls-container {
  margin-top: 30px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
}

.control-button {
  background-color: #363636;
  color: white;
  border: none;
  border-radius: 50%; /* 丸いボタン */
  width: 50px;
  height: 50px;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: background-color 0.2s ease;
}

.control-button:hover {
  background-color: #555;
}

.volume-slider {
  width: 150px;
  cursor: pointer;
}
/* ▲▲▲ ここまで ▲▲▲ */
</style>