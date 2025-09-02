<template>
  <div v-if="isVisible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <button class="close-button" @click="close">×</button>
      <h2>サウンドチェック</h2>
      <p>各楽器の最終的な音色を確認します。</p>
      <ul class="sound-list">
        <li v-for="instrument in instruments" :key="instrument" class="sound-item">
          <span class="instrument-name">{{ instrument }}</span>
          <button class="play-button" @click="playSound(instrument)">▶ 再生</button>
        </li>
      </ul>
      <div v-if="!instruments.length" class="no-sounds-message">
        <p>楽器が読み込まれていません。</p>
        <p>BGMを一度再生すると、ここで音の確認ができます。</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
defineProps<{
  isVisible: boolean;
  instruments: string[];
}>();

const emit = defineEmits(['close', 'playSound']);

const close = () => {
  emit('close');
};

const playSound = (instrumentName: string) => {
  emit('playSound', instrumentName);
};
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: #fff;
  padding: 30px 40px;
  border-radius: 8px;
  box-shadow: 0 5px 20px rgba(0,0,0,0.3);
  max-width: 500px;
  width: 90%;
  position: relative;
  font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif;
  color: #333;
  max-height: 80vh;
  overflow-y: auto;
}

.modal-content h2 {
  text-align: center;
  margin-top: 0;
  margin-bottom: 15px;
}

.modal-content p {
  text-align: center;
  margin-top: 0;
  margin-bottom: 25px;
  line-height: 1.6;
}

.close-button {
  position: absolute;
  top: 10px;
  right: 15px;
  background: none;
  border: none;
  font-size: 28px;
  cursor: pointer;
  color: #888;
}

.sound-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.sound-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid #eee;
}

.sound-item:last-child {
  border-bottom: none;
}

.instrument-name {
  font-size: 1.1em;
}

.play-button {
  background-color: #363636;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 15px;
  cursor: pointer;
  transition: background-color 0.2s ease;
  font-size: 1em;
}

.play-button:hover {
  background-color: #555;
}

.no-sounds-message {
  text-align: center;
  color: #7a7a7a;
  padding: 20px;
}
</style>