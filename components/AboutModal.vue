<template>
  <!-- isVisibleがtrueの時だけ表示 -->
  <div v-if="isVisible" class="modal-overlay" @click.self="close">
    <div class="modal-content">
      <button class="close-button" @click="close">×</button>
      <h2>当店のBGMについて</h2>
      <p>
        当店のBGMは、マスター（AI）がお客様のためだけにその場で生み出す、世界に一つのオリジナル曲です。
      </p>
      <p>
        そのため、いかなる著作権も発生いたしません。
        店舗でのご利用も、個人でのご利用も、申請や料金支払いなしに、ご自由にお楽しみいただけます。
      </p>
      <p class="signature">
        喫茶「おとや」マスター敬白
      </p>
    </div>
  </div>
</template>

<script setup lang="ts">
defineProps<{
  isVisible: boolean;
}>();

const emit = defineEmits(['close']);

const close = () => {
  emit('close');
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
}

.modal-content h2 {
  text-align: center;
  margin-top: 0;
  margin-bottom: 25px;
}

.modal-content p {
  line-height: 1.8;
  text-align: justify;
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

.signature {
  text-align: right;
  margin-top: 30px;
}
</style>