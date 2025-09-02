<template>
  <div v-if="isVisible" class="modal-backdrop">
    <div class="modal-content">
      <header class="modal-header">
        <h2>このBGMについて</h2>
        <button class="close-button" @click="$emit('close')" aria-label="閉じる">×</button>
      </header>

      <section class="modal-body">
        <p>「AI-BGM 喫茶 おとや」は、Tone.js を用いてブラウザ内でBGMを自動生成するアプリです。生成された音源は著作権フリーです。</p>
        <ul>
          <li>処理はすべてお使いの端末上で完結します（サーバー送信なし）。</li>
          <li>音量にはご注意ください。初回は中程度に設定しています。</li>
          <li>生成は確率的に行われるため、同じメニューでも表情が変化します。</li>
        </ul>
        <p class="small">Powered by Tone.js</p>
      </section>

      <footer class="modal-footer">
        <button class="primary" @click="$emit('close')">閉じる</button>
      </footer>
    </div>
  </div>
</template>

<script setup lang="ts">
const props = defineProps<{
  isVisible: boolean
}>()

const emit = defineEmits<{
  (e: 'close'): void
}>()
</script>

<style scoped>
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.5);
  display: grid;
  place-items: center;
  z-index: 1000;
}

.modal-content {
  width: min(640px, 92vw);
  background: #3d2e29;
  color: #f3e6d0;
  border: 1px solid rgba(255,255,255,0.08);
  border-radius: 18px;
  box-shadow: 0 6px 28px rgba(0,0,0,0.4);
}

.modal-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 14px 16px; border-bottom: 1px solid rgba(255,255,255,0.08);
}

.modal-body { padding: 16px; }
.modal-body .small { opacity: 0.8; font-size: 0.9rem; }

.modal-footer {
  display: flex; justify-content: flex-end; gap: 10px;
  padding: 12px 16px; border-top: 1px solid rgba(255,255,255,0.08);
}

button.primary {
  background: #5b4236; color: #f3e6d0; border: none; padding: 8px 12px; border-radius: 12px;
}
.close-button {
  background: transparent; border: none; color: #f3e6d0; font-size: 20px; cursor: pointer;
}
</style>
