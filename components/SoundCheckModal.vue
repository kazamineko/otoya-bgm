<template>
  <div v-if="isVisible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <button class="close-button" @click="close">×</button>
      <h2>サウンドチューニング</h2>
      <p>各楽器の音響特性を詳細に調整します。</p>
      
      <div class="global-actions">
        <button @click="saveParams" class="action-button">現在の設定を保存</button>
        <button @click="exportParams" class="action-button">コンソールに出力</button>
        <button @click="resetParams" class="action-button action-button-reset">初期設定に戻す</button>
      </div>

      <ul class="sound-list">
        <li v-for="instrument in instruments" :key="instrument" class="sound-item">
          <details class="instrument-details" :open="instrument === 'eguitar' || instrument === 'ebass'">
            <summary class="instrument-summary">
              <span class="instrument-name">{{ instrument }}</span>
              <div class="play-buttons">
                <template v-if="instrument === 'eguitar'">
                  <button @click.prevent="playSound('target_eguitar', 'target_sampler')" title="新しいマルチサンプル音源">新Sampler</button>
                  <button @click.prevent="playSound('eguitar', 'target')" title="最終的に目指すべき理想の音(WAV再生)">目標サウンド</button>
                  <a href="/C5_s6_01.wav" download="target-sound-C5.wav" class="download-button" title="目標サウンドをダウンロード">DL</a>
                </template>
                <template v-else-if="instrument === 'ebass'">
                  <button @click.prevent="playSound('target_ebass', 'target_sampler')" title="マスターが作成した新しいマルチサンプルベース音源">新Rock Bass</button>
                </template>
                <template v-else>
                  <button @click.prevent="playSound(instrument, 'sampler')">Sampler</button>
                  <button @click.prevent="playSound(instrument, 'raw')">原音</button>
                </template>
              </div>
            </summary>
            
            <div class="sliders" v-if="tuningParams[instrument] || tuningParams['target_' + instrument]">
              <!-- eGuitar & eBass Sampler Logic -->
              <template v-if="instrument === 'eguitar' || instrument === 'ebass'">
                <div class="sub-header">{{ instrument === 'eguitar' ? 'マルチサンプル設定' : 'Sampler 設定' }}</div>
                <div class="slider-container">
                  <label>Volume</label>
                  <input type="range" min="-40" max="6" step="0.1" :value="tuningParams['target_' + instrument].volume" @input="updateParam('target_' + instrument, 'volume', $event)">
                  <span>{{ tuningParams['target_' + instrument].volume.toFixed(1) }} dB</span>
                </div>
                <div class="slider-container">
                  <label>Attack</label>
                  <input type="range" min="0" max="2" step="0.001" :value="tuningParams['target_' + instrument].attack" @input="updateParam('target_' + instrument, 'attack', $event)">
                  <span>{{ tuningParams['target_' + instrument].attack.toFixed(3) }} s</span>
                </div>
                <div class="slider-container">
                  <label>Release</label>
                  <input type="range" min="0" max="30" step="0.1" :value="tuningParams['target_' + instrument].release" @input="updateParam('target_' + instrument, 'release', $event)">
                  <span>{{ tuningParams['target_' + instrument].release.toFixed(2) }} s</span>
                </div>
                <template v-if="instrument === 'eguitar'">
                  <div class="slider-container">
                    <label>EQ Low</label>
                    <input type="range" min="-12" max="12" step="0.5" :value="tuningParams['target_eguitar'].eqLow" @input="updateParam('target_eguitar', 'eqLow', $event)">
                    <span>{{ tuningParams['target_eguitar'].eqLow.toFixed(1) }} dB</span>
                  </div>
                  <div class="slider-container">
                    <label>EQ Mid</label>
                    <input type="range" min="-12" max="12" step="0.5" :value="tuningParams['target_eguitar'].eqMid" @input="updateParam('target_eguitar', 'eqMid', $event)">
                    <span>{{ tuningParams['target_eguitar'].eqMid.toFixed(1) }} dB</span>
                  </div>
                  <div class="slider-container">
                    <label>EQ High</label>
                    <input type="range" min="-12" max="12" step="0.5" :value="tuningParams['target_eguitar'].eqHigh" @input="updateParam('target_eguitar', 'eqHigh', $event)">
                    <span>{{ tuningParams['target_eguitar'].eqHigh.toFixed(1) }} dB</span>
                  </div>
                </template>
              </template>
              <!-- Other Instruments -->
              <template v-else>
                <div class="slider-container">
                  <label>Volume</label>
                  <input 
                    type="range" 
                    min="-40" 
                    :max="['rockKick', 'rockSnare', 'crash', 'tomHigh', 'tomMid', 'tomFloor'].includes(instrument) ? 12 : 6" 
                    step="0.1" 
                    :value="tuningParams[instrument].volume" 
                    @input="updateParam(instrument, 'volume', $event)">
                  <span>{{ tuningParams[instrument].volume.toFixed(1) }} dB</span>
                </div>
                <div class="slider-container">
                  <label>Attack</label>
                  <input type="range" min="0" max="2" step="0.001" :value="tuningParams[instrument].attack" @input="updateParam(instrument, 'attack', $event)">
                  <span>{{ tuningParams[instrument].attack.toFixed(3) }} s</span>
                </div>
                <div class="slider-container">
                  <label>Release</label>
                  <input type="range" min="0" max="5" step="0.01" :value="tuningParams[instrument].release" @input="updateParam(instrument, 'release', $event)">
                  <span>{{ tuningParams[instrument].release.toFixed(2) }} s</span>
                </div>
              </template>
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
defineProps<{
  isVisible: boolean;
  instruments: string[];
  tuningParams: Record<string, any>;
}>();

const emit = defineEmits(['close', 'playSound', 'updateParam', 'saveParams', 'exportParams', 'resetParams']);

const close = () => emit('close');
const playSound = (instrumentName: string, type: 'sampler' | 'raw' | 'target' | 'target_sampler') => emit('playSound', instrumentName, type);
const saveParams = () => emit('saveParams');
const exportParams = () => emit('exportParams');
const resetParams = () => emit('resetParams');

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
  box-shadow: 0 5px 20px rgba(0,0,0,0.3); max-width: 680px;
  width: 90%; position: relative; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif;
  color: #333; max-height: 90vh; overflow-y: auto;
}
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
.action-button-reset {
  background-color: #7a7a7a;
}
.action-button-reset:hover {
  background-color: #6a6a6a;
}
.sound-list { list-style: none; padding: 0; margin: 0; }
.instrument-details { border-bottom: 1px solid #eee; }
.instrument-summary {
  display: flex; justify-content: space-between; align-items: center;
  padding: 15px 5px; cursor: pointer;
}
.instrument-summary::-webkit-details-marker { display: none; }
.instrument-name { font-size: 1.1em; font-weight: bold; }
.play-buttons { display: flex; gap: 8px; flex-wrap: wrap; justify-content: flex-end;}
.play-buttons button {
  background-color: #363636; color: white; border: none; border-radius: 4px;
  padding: 8px 12px; cursor: pointer; transition: background-color 0.2s ease;
  font-size: 12px;
}
.play-buttons button:hover { background-color: #555; }
.download-button {
  background-color: #238636;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 12px;
  cursor: pointer;
  transition: background-color 0.2s ease;
  font-size: 12px;
  text-decoration: none;
  display: inline-block;
  font-family: inherit;
}
.download-button:hover {
  background-color: #1a6328;
}
.sliders { padding: 10px 15px 20px; background-color: #f9f9f9; }
.slider-container {
  display: grid; grid-template-columns: 120px 1fr 100px;
  align-items: center; gap: 10px; margin-bottom: 8px;
}
.slider-container label { text-align: right; font-size: 0.9em; }
.slider-container input[type="range"] { width: 100%; }
.slider-container span { font-family: monospace; font-size: 0.9em; text-align: left; }
.no-sounds-message { text-align: center; color: #7a7a7a; padding: 20px; }
.sub-header {
  font-weight: bold;
  margin-top: 15px;
  margin-bottom: 10px;
  text-align: center;
  background-color: #eee;
  padding: 5px;
  border-radius: 4px;
}
</style>