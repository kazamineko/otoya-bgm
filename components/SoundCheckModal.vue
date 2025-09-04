<template>
  <div v-if="isVisible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <button class="close-button" @click="close">×</button>
      <h2>サウンドチューニング</h2>
      <p>各楽器の音量、アタック(音の鋭さ)、リリース(余韻)を調整します。</p>
      
      <div class="global-actions">
        <button @click="saveParams" class="action-button">現在の設定を保存</button>
        <button @click="exportParams" class="action-button">コンソールに出力</button>
      </div>

      <ul class="sound-list">
        <li v-for="instrument in instruments" :key="instrument" class="sound-item">
          <details class="instrument-details">
            <summary class="instrument-summary">
              <span class="instrument-name">{{ instrument }}</span>
              <div class="play-buttons">
                <template v-if="instrument === 'eguitar'">
                  <button @click.prevent="playSound(instrument, 'sampler')" title="DI音源を仮想アンプで加工した音">加工後DI</button>
                  <button @click.prevent="playSound(instrument, 'target')" title="最終的に目指すべき理想の音">目標サウンド</button>
                  <button @click.prevent="playSound(instrument, 'raw')" title="エフェクトを何も通していない、録音したままの音">原音DI</button>
                </template>
                <template v-else-if="instrument === 'ebass'">
                  <button @click.prevent="playSound(instrument, 'sampler')" title="DI音源を仮想アンプで加工した音">加工後DI</button>
                  <button @click.prevent="playSound(instrument, 'target')" title="最終的に目指すべき理想の音">目標サウンド</button>
                  <button @click.prevent="playSound(instrument, 'raw')" title="エフェクトを何も通していない、録音したままの音">原音DI</button>
                </template>
                <template v-else>
                  <button @click.prevent="playSound(instrument, 'sampler')">Sampler</button>
                  <button @click.prevent="playSound(instrument, 'raw')">原音</button>
                </template>
              </div>
            </summary>
            
            <div class="sliders" v-if="tuningParams[instrument]">
              <div class="slider-container">
                <label>Volume</label>
                <input type="range" min="-40" max="6" step="0.1" 
                  :value="tuningParams[instrument]!.volume" 
                  @input="updateParam(instrument, 'volume', $event)">
                <span>{{ tuningParams[instrument]!.volume.toFixed(1) }} dB</span>
              </div>
              <div class="slider-container">
                <label>Attack</label>
                <input type="range" min="0" max="2" step="0.001" 
                  :value="tuningParams[instrument]!.attack" 
                  @input="updateParam(instrument, 'attack', $event)">
                <span>{{ tuningParams[instrument]!.attack.toFixed(3) }} s</span>
              </div>
              <div class="slider-container">
                <label>Release</label>
                <input type="range" min="0" max="5" step="0.01" 
                  :value="tuningParams[instrument]!.release" 
                  @input="updateParam(instrument, 'release', $event)">
                <span>{{ tuningParams[instrument]!.release.toFixed(2) }} s</span>
              </div>
              <!-- 専用スライダー -->
              <div v-if="instrument === 'eguitar' || instrument === 'ebass'" class="slider-container">
                  <label>Distortion</label>
                  <input type="range" min="0" max="1" step="0.01"
                      :value="tuningParams[instrument]!.distortion"
                      @input="updateParam(instrument, 'distortion', $event)">
                  <span>{{ (tuningParams[instrument]!.distortion ?? 0).toFixed(2) }}</span>
              </div>
            </div>
          </details>
        </li>
      </ul>

      <div v-if="!instruments.length" class="no-sounds-message">
        <p>BGMを一度再生すると、チューニングを開始できます。</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
type TuningParams = Record<string, { volume: number; attack: number; release: number; distortion?: number }>;

defineProps<{
  isVisible: boolean;
  instruments: string[];
  tuningParams: TuningParams;
}>();

const emit = defineEmits(['close', 'playSound', 'updateParam', 'saveParams', 'exportParams']);

const close = () => emit('close');
const playSound = (instrumentName: string, type: 'sampler' | 'raw' | 'target') => emit('playSound', instrumentName, type);
const saveParams = () => emit('saveParams');
const exportParams = () => emit('exportParams');

const updateParam = (instrument: string, param: string, event: Event) => {
  const value = parseFloat((event.target as HTMLInputElement).value);
  emit('updateParam', { instrument, param, value });
};
</script>

<style scoped>
.modal-overlay {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background-color: rgba(0, 0, 0, 0.6); display: flex;
  justify-content: center; align-items: center; z-index: 1000;
}
.modal-content {
  background-color: #fff; padding: 20px 30px; border-radius: 8px;
  box-shadow: 0 5px 20px rgba(0,0,0,0.3); max-width: 600px;
  width: 90%; position: relative; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif;
  color: #333; max-height: 90vh; overflow-y: auto;
}
.modal-content h2 { text-align: center; margin-top: 0; margin-bottom: 10px; }
.modal-content p { text-align: center; margin-top: 0; margin-bottom: 20px; line-height: 1.6; }
.close-button {
  position: absolute; top: 10px; right: 15px; background: none; border: none;
  font-size: 28px; cursor: pointer; color: #888;
}
.global-actions {
  display: flex; gap: 10px; justify-content: center; margin-bottom: 20px;
}
.action-button {
  background-color: #485fc7; color: white; border: none; border-radius: 4px;
  padding: 10px 15px; cursor: pointer; transition: background-color 0.2s ease;
}
.action-button:hover { background-color: #3e51b3; }
.sound-list { list-style: none; padding: 0; margin: 0; }
.instrument-details { border-bottom: 1px solid #eee; }
.instrument-summary {
  display: flex; justify-content: space-between; align-items: center;
  padding: 15px 5px; cursor: pointer;
}
.instrument-summary::-webkit-details-marker { display: none; }
.instrument-name { font-size: 1.1em; font-weight: bold; }
.play-buttons { display: flex; gap: 8px; }
.play-buttons button {
  background-color: #363636; color: white; border: none; border-radius: 4px;
  padding: 8px 12px; cursor: pointer; transition: background-color 0.2s ease;
  font-size: 12px;
}
.play-buttons button:hover { background-color: #555; }
.sliders { padding: 10px 15px 20px; background-color: #f9f9f9; }
.slider-container {
  display: grid; grid-template-columns: 80px 1fr 100px;
  align-items: center; gap: 10px; margin-bottom: 8px;
}
.slider-container label { text-align: right; font-size: 0.9em; }
.slider-container input[type="range"] { width: 100%; }
.slider-container span { font-family: monospace; font-size: 0.9em; text-align: left; }
.no-sounds-message { text-align: center; color: #7a7a7a; padding: 20px; }
</style>