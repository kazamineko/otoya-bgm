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
                <template v-if="instrument === 'eguitar' || instrument === 'ebass'">
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
              <!-- eGuitar 専用 -->
              <template v-if="instrument === 'eguitar'">
                <div class="sub-header">Input Gain (入力音量)</div>
                <div class="slider-container">
                  <label>Gain</label>
                  <input type="range" min="0" max="24" step="0.5" :value="tuningParams.eguitar.inputGain" @input="updateParam('eguitar', 'inputGain', $event)">
                  <span>{{ tuningParams.eguitar.inputGain.toFixed(1) }} dB</span>
                </div>
                <div class="sub-header">Pre-Dist Compressor (音の粒立ち・サステイン)</div>
                <div class="slider-container">
                  <label>Threshold</label>
                  <input type="range" min="-48" max="0" step="1" :value="tuningParams.eguitar.preCompThreshold" @input="updateParam('eguitar', 'preCompThreshold', $event)">
                  <span>{{ tuningParams.eguitar.preCompThreshold }} dB</span>
                </div>
                <div class="slider-container">
                  <label>Ratio</label>
                  <input type="range" min="1" max="20" step="1" :value="tuningParams.eguitar.preCompRatio" @input="updateParam('eguitar', 'preCompRatio', $event)">
                  <span>{{ tuningParams.eguitar.preCompRatio }}:1</span>
                </div>
                <div class="slider-container">
                  <label>Attack</label>
                  <input type="range" min="0.001" max="0.2" step="0.001" :value="tuningParams.eguitar.preCompAttack" @input="updateParam('eguitar', 'preCompAttack', $event)">
                  <span>{{ tuningParams.eguitar.preCompAttack.toFixed(3) }} s</span>
                </div>
                <div class="slider-container">
                  <label>Release</label>
                  <input type="range" min="0.01" max="0.5" step="0.01" :value="tuningParams.eguitar.preCompRelease" @input="updateParam('eguitar', 'preCompRelease', $event)">
                  <span>{{ tuningParams.eguitar.preCompRelease.toFixed(2) }} s</span>
                </div>
                <div class="sub-header">Pre EQ (歪みのキャラクター)</div>
                <div class="slider-container">
                  <label>Mid Freq</label>
                  <input type="range" min="300" max="3000" step="10" :value="tuningParams.eguitar.preEqFreq" @input="updateParam('eguitar', 'preEqFreq', $event)">
                  <span>{{ tuningParams.eguitar.preEqFreq }} Hz</span>
                </div>
                <div class="slider-container">
                  <label>Mid Gain</label>
                  <input type="range" min="0" max="24" step="0.5" :value="tuningParams.eguitar.preEqGain" @input="updateParam('eguitar', 'preEqGain', $event)">
                  <span>{{ tuningParams.eguitar.preEqGain.toFixed(1) }} dB</span>
                </div>
                <div class="slider-container">
                  <label>Distortion</label>
                  <input type="range" min="0" max="1" step="0.01" :value="tuningParams.eguitar.distortion" @input="updateParam('eguitar', 'distortion', $event)">
                  <span>{{ tuningParams.eguitar.distortion.toFixed(2) }}</span>
                </div>
                <div class="sub-header">Post EQ (最終的な音質)</div>
                <div class="slider-container">
                  <label>Low</label>
                  <input type="range" min="-24" max="24" step="0.5" :value="tuningParams.eguitar.postEqLow" @input="updateParam('eguitar', 'postEqLow', $event)">
                  <span>{{ tuningParams.eguitar.postEqLow.toFixed(1) }} dB</span>
                </div>
                 <div class="slider-container">
                  <label>Mid</label>
                  <input type="range" min="-24" max="24" step="0.5" :value="tuningParams.eguitar.postEqMid" @input="updateParam('eguitar', 'postEqMid', $event)">
                  <span>{{ tuningParams.eguitar.postEqMid.toFixed(1) }} dB</span>
                </div>
                 <div class="slider-container">
                  <label>High</label>
                  <input type="range" min="-24" max="24" step="0.5" :value="tuningParams.eguitar.postEqHigh" @input="updateParam('eguitar', 'postEqHigh', $event)">
                  <span>{{ tuningParams.eguitar.postEqHigh.toFixed(1) }} dB</span>
                </div>
                <div class="sub-header">Chorus (音の揺れ・広がり)</div>
                <div class="slider-container">
                  <label>Depth</label>
                  <input type="range" min="0" max="1" step="0.01" :value="tuningParams.eguitar.chorusDepth" @input="updateParam('eguitar', 'chorusDepth', $event)">
                  <span>{{ tuningParams.eguitar.chorusDepth.toFixed(2) }}</span>
                </div>
                 <div class="slider-container">
                  <label>Rate</label>
                  <input type="range" min="0.1" max="8" step="0.1" :value="tuningParams.eguitar.chorusRate" @input="updateParam('eguitar', 'chorusRate', $event)">
                  <span>{{ tuningParams.eguitar.chorusRate.toFixed(1) }} Hz</span>
                </div>
              </template>
              <!-- eBass 専用 -->
              <template v-else-if="instrument === 'ebass'">
                <div class="sub-header">Input Gain (入力音量)</div>
                 <div class="slider-container">
                  <label>Gain</label>
                  <input type="range" min="0" max="24" step="0.5" :value="tuningParams.ebass.inputGain" @input="updateParam('ebass', 'inputGain', $event)">
                  <span>{{ tuningParams.ebass.inputGain.toFixed(1) }} dB</span>
                </div>
                <div class="sub-header">Pre-Dist Compressor</div>
                <div class="slider-container">
                  <label>Threshold</label>
                  <input type="range" min="-48" max="0" step="1" :value="tuningParams.ebass.preCompThreshold" @input="updateParam('ebass', 'preCompThreshold', $event)">
                  <span>{{ tuningParams.ebass.preCompThreshold }} dB</span>
                </div>
                <div class="slider-container">
                  <label>Ratio</label>
                  <input type="range" min="1" max="20" step="1" :value="tuningParams.ebass.preCompRatio" @input="updateParam('ebass', 'preCompRatio', $event)">
                  <span>{{ tuningParams.ebass.preCompRatio }}:1</span>
                </div>
                <div class="slider-container">
                  <label>Attack</label>
                  <input type="range" min="0.001" max="0.2" step="0.001" :value="tuningParams.ebass.preCompAttack" @input="updateParam('ebass', 'preCompAttack', $event)">
                  <span>{{ tuningParams.ebass.preCompAttack.toFixed(3) }} s</span>
                </div>
                <div class="slider-container">
                  <label>Release</label>
                  <input type="range" min="0.01" max="0.5" step="0.01" :value="tuningParams.ebass.preCompRelease" @input="updateParam('ebass', 'preCompRelease', $event)">
                  <span>{{ tuningParams.ebass.preCompRelease.toFixed(2) }} s</span>
                </div>
                <div class="sub-header">Parallel Blend (サウンドの核)</div>
                <div class="slider-container">
                  <label>Sub Blend</label>
                  <input type="range" min="0" max="1" step="0.01" :value="tuningParams.ebass.subBlend" @input="updateParam('ebass', 'subBlend', $event)">
                  <span>{{ (tuningParams.ebass.subBlend * 100).toFixed(0) }}%</span>
                </div>
                <div class="sub-header">DI Path (歪みのキャラクター)</div>
                 <div class="slider-container">
                  <label>Drive</label>
                  <input type="range" min="0" max="1" step="0.01" :value="tuningParams.ebass.drive" @input="updateParam('ebass', 'drive', $event)">
                  <span>{{ tuningParams.ebass.drive.toFixed(2) }}</span>
                </div>
                <div class="slider-container">
                  <label>EQ Low</label>
                  <input type="range" min="-12" max="12" step="0.5" :value="tuningParams.ebass.eqLow" @input="updateParam('ebass', 'eqLow', $event)">
                  <span>{{ tuningParams.ebass.eqLow.toFixed(1) }} dB</span>
                </div>
                <div class="slider-container">
                  <label>EQ Mid</label>
                  <input type="range" min="-12" max="12" step="0.5" :value="tuningParams.ebass.eqMid" @input="updateParam('ebass', 'eqMid', $event)">
                  <span>{{ tuningParams.ebass.eqMid.toFixed(1) }} dB</span>
                </div>
                <div class="slider-container">
                  <label>EQ High</label>
                  <input type="range" min="-12" max="12" step="0.5" :value="tuningParams.ebass.eqHigh" @input="updateParam('ebass', 'eqHigh', $event)">
                  <span>{{ tuningParams.ebass.eqHigh.toFixed(1) }} dB</span>
                </div>
              </template>
              <!-- その他楽器 -->
              <template v-else>
                <div class="slider-container">
                  <label>Volume</label>
                  <input type="range" min="-40" max="6" step="0.1" :value="tuningParams[instrument].volume" @input="updateParam(instrument, 'volume', $event)">
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
const playSound = (instrumentName: string, type: 'sampler' | 'raw' | 'target') => emit('playSound', instrumentName, type);
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
.play-buttons { display: flex; gap: 8px; }
.play-buttons button {
  background-color: #363636; color: white; border: none; border-radius: 4px;
  padding: 8px 12px; cursor: pointer; transition: background-color 0.2s ease;
  font-size: 12px;
}
.play-buttons button:hover { background-color: #555; }
.sliders { padding: 10px 15px 20px; background-color: #f9f9f9; }
.slider-container {
  display: grid; grid-template-columns: 120px 1fr 100px; /* Label width increased */
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